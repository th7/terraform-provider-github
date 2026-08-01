## This fork, its branches, and how to check work

`th7/terraform-provider-github` is a fork of
`integrations/terraform-provider-github` that carries one patch ahead of
upstream and publishes it as a preview provider. `PREVIEW.md` is the reference;
read it before moving any branch.

### Branches

| Branch | Holds |
| --- | --- |
| `fix/preserve-unreadable-repo-fields` | Exactly one commit on top of upstream `main`: only what is meant for the upstream pull request. Code here follows upstream's `CONTRIBUTING.md`, `ARCHITECTURE.md` and `DECISIONS.md`. |
| `main` | The default branch, and where sessions read `.my-agent/` from. Exactly one commit on top of the fix branch: docs, CI/CD and `.my-agent/` only, never source. |
| `upstream-main` | Upstream `main` at the fix branch's base. |
| `gh-pages` | The published provider mirror, written by the Preview provider workflow. |

Nothing from `main` may reach the fix branch. Never merge a pull request into
`main`: it would break the one-commit shape. Both invariants are checkable:

```sh
git remote add upstream https://github.com/integrations/terraform-provider-github.git  # if missing
git fetch upstream origin
git rev-list --count upstream/main..origin/fix/preserve-unreadable-repo-fields   # 1
git rev-list --count origin/fix/preserve-unreadable-repo-fields..origin/main     # 1
git diff --stat origin/fix/preserve-unreadable-repo-fields origin/main          # no source files
```

Work lands by rewriting those single commits, which a session cannot put in
place: it may not push `main`, and moving another existing branch needs a
human. Push the result to new branches, say which existing branch each one
replaces and at what commit, and let the human move them.

### Syncing with upstream

Rebase the fix branch onto `upstream/main`, then rebuild `main` as the new fix
branch plus `main`'s single docs/CI commit, and point `upstream-main` at
`upstream/main`. Resolve conflicts toward upstream's current code and style.
Run every check below on the rebased fix branch before handing it over.

### Checks

| Command | What | Warm time |
| --- | --- | --- |
| `go build ./...` | build | ~4 s (~17 s cold) |
| `make test` | unit tests; skips `TestAcc*` | ~7-12 s |
| `golangci-lint run ./...` | CI's lint | ~6 s (~20 s cold) |
| `make lintcheck-new` | strict lint on new code, as CI runs on pull requests | ~13 s |
| docs, below | generate and validate `docs/` | ~3 s |
| `rumdl check ./docs` | docs markdown lint | <1 s |

`tfplugindocs` names the provider after the checkout's directory, and
`/workspace` is not `terraform-provider-github`, so `make generatedocs`,
`make checkdocs` and `make validatedocs` render docs for a provider called
`workspace` and rewrite or delete most of `docs/`. Run them with the name
given:

```sh
cd tools
terraform fmt -recursive ../examples/
go run github.com/hashicorp/terraform-plugin-docs/cmd/tfplugindocs generate --provider-dir .. --provider-name github --rendered-provider-name GitHub
go run github.com/hashicorp/terraform-plugin-docs/cmd/tfplugindocs validate --provider-dir .. --provider-name github
```

`docs/` is generated from `templates/`: edit the template and regenerate, and
commit both. `git status` must be clean after generating.

Acceptance tests (`make testacc`, `TF_ACC=1`) create real resources in a GitHub
organization and need credentials for one. Do not run them; say they were not
run.
