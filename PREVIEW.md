# Preview provider

How the patched provider on this branch is built and published for use in
personal projects, ahead of any upstream release.

This file lives on `main`, which exists so that
`fix/preserve-unreadable-repo-fields` stays clean: that branch carries only
what is intended for a PR to `integrations/terraform-provider-github`.
Everything added on `main` — this file, the background notes, the README
notice, and any build/deploy automation — is preview support and must never
be part of that PR.

## Branches

| Branch | Purpose |
| --- | --- |
| `fix/preserve-unreadable-repo-fields` | Exactly one commit on top of upstream `main`. Only changes intended for a PR upstream. |
| `main` | Exactly one commit on top of `fix/...`. Changes no source — documentation and CI/CD only. |
| `gh-pages` | Deploy location only — the published mirror. |

Both invariants are mechanical, so they can be checked:

```sh
git rev-list --count upstream/main..fix/preserve-unreadable-repo-fields   # 1
git rev-list --count fix/preserve-unreadable-repo-fields..main            # 1
git diff --stat fix/preserve-unreadable-repo-fields main                  # no source files
```

When upstream moves, or the fix commit changes, rebase in that order and
squash `main` back down to a single commit:

```sh
git fetch upstream
git rebase upstream/main fix/preserve-unreadable-repo-fields
git rebase fix/preserve-unreadable-repo-fields main
```

## What is published

Served as a [Terraform provider network mirror][mirror] from the `gh-pages`
branch of this fork.

| | |
| --- | --- |
| URL | `https://th7.github.io/terraform-provider-github/mirror/` |
| Version | `6.13.1-th7.2` — the hyphen marks it a prerelease, so it is never selected automatically. `6.13.1-th7.1`, the patch's first revision, stays in the mirror. |
| Platforms | `linux_amd64` (CI runners) and `linux_arm64` (workstation) |
| Signing | None. Network mirrors are not signature-verified, so no GPG key or registry account is involved. |

The version is deliberately not a plain `6.13.1`: it can never collide with an
upstream release, and a constraint has to name it exactly to get it.

## Consuming it

```hcl
# ~/.terraformrc on the consuming machine
provider_installation {
  network_mirror {
    url     = "https://th7.github.io/terraform-provider-github/mirror/"
    include = ["registry.terraform.io/th7/*"]
  }
  direct { exclude = ["registry.terraform.io/th7/*"] }
}
```

```hcl
terraform {
  required_providers {
    github = {
      source  = "th7/github"
      version = "6.13.1-th7.2"
    }
  }
}
```

Then record **every** platform the project runs on in its lock file:

```sh
terraform providers lock \
  -net-mirror=https://th7.github.io/terraform-provider-github/mirror/ \
  -platform=linux_amd64 -platform=linux_arm64
```

This is not optional. A lock file written by an `init` on an arm64 workstation
carries only the arm64 hash, and an `ubuntu-latest` runner then fails the
checksum check even though the amd64 package is present in the mirror. Note
that `terraform providers lock` ignores the mirrors configured in CLI config —
without an explicit `-net-mirror` it queries the origin registry and reports
that `th7/github` does not exist.

## Publishing a new version

Use the **Preview provider** workflow: Actions → Preview provider → Run
workflow, and give it a version such as `6.13.1-th7.3`. It runs the unit
tests, cross-compiles both platforms, computes each archive's hash, updates
the mirror on `gh-pages`, and then proves the result installs by locking
against the live URL.

Publishing is manual only. Pushes to `main` run the test and build jobs, so a
broken cross-compile is caught early, but never touch the mirror.

Two guards worth knowing about, both of which fail the run:

- A version without a prerelease suffix is rejected, since it could collide
  with an upstream release and be picked up by a plain version constraint.
- A version already present in `index.json` is rejected, because Terraform
  caches by version and consumers may never see the replacement. Re-run with
  `allow_overwrite` if that is genuinely what you want.

### Publishing by hand

Should the workflow be unavailable, the same steps are:

Bump the version; Terraform caches by version, so republishing the same
version does not reliably reach consumers.

```sh
V=6.13.1-th7.3
for GOARCH in amd64 arm64; do
  GOOS=linux GOARCH=$GOARCH CGO_ENABLED=0 go build \
    -ldflags "-s -w -X main.version=${V} -X main.commit=$(git rev-parse HEAD)" \
    -o "terraform-provider-github_v${V}"
  zip "terraform-provider-github_${V}_linux_${GOARCH}.zip" \
    "terraform-provider-github_v${V}"
done
```

Get each archive's `h1:` hash without publishing first, running each platform
in its own empty directory so it is unambiguous which hash belongs to which:

```sh
terraform providers lock -fs-mirror=<dir> -platform=linux_amd64
terraform providers lock -fs-mirror=<dir> -platform=linux_arm64
```

On `gh-pages`, under `mirror/registry.terraform.io/th7/github/`:

- add the zips
- add the new version as a key in `index.json`
- add a `${V}.json` with one entry per platform, naming its archive and hash

The `hashes` field may be omitted, in which case Terraform computes it at
install time.

## Notes

- `.nojekyll` at the root of `gh-pages` keeps Pages from processing the tree.
- Pages must serve the index files as `application/json`.
  `raw.githubusercontent.com` returns `text/plain` and Terraform rejects it, so
  Pages is required rather than merely convenient.
- The artifacts are publicly downloadable by anyone with the URL. If that
  stops being acceptable, the same layout works behind an authenticated HTTPS
  host with a `credentials` block in the CLI config.
- The patch that this preview carries is described in
  [`preserve-unreadable-repo-fields.md`](preserve-unreadable-repo-fields.md).

## Retiring the preview

Once the fix lands upstream and is released: point consuming projects back at
`integrations/github`, refresh their `.terraform.lock.hcl`, and delete the
`gh-pages` mirror entry.

[mirror]: https://developer.hashicorp.com/terraform/internals/provider-network-mirror-protocol
