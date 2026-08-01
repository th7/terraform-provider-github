# preserve-unreadable-repo-fields.patch

A patch for `integrations/terraform-provider-github`. It stops the provider
reporting a permanent, unresolvable diff on every `github_repository` when the
GitHub token cannot read the merge-related settings — which is the case for any
genuinely read-only token, and so for this repo's plan path.

It is the single commit on `fix/preserve-unreadable-repo-fields`, kept on top of
upstream `main` as [`PREVIEW.md`](PREVIEW.md) describes, and is published as the
preview provider `6.13.1-th7.2`. It was first written on 2026-07-26 against
`b0b30a1`, in a container whose GitHub credentials expired before it could be
pushed; this file began as the record that let the work survive that.

## What it changes

`resourceGithubRepositoryRead` sets thirteen fields from the API response using
go-github's `Get*` accessors. Those accessors dereference a pointer and return
the zero value when it is `nil`, so a field the API **omitted** becomes
indistinguishable from one genuinely reported as `false` or `""`.

GitHub omits those fields in two situations: the repository is archived, or the
caller lacks push/admin access. The API documents the second:

> To view merge-related settings, you must have the `contents:read` and
> `contents:write` permissions.

So a read-only token makes the provider write `false` into state for ten
attributes, which then diff against the configuration on every plan. An apply
cannot settle it either, because the value it would write back is not readable.

The patch adds a generic helper to `github/util.go`, since the bug class is not
repository-specific, and uses the pointers directly:

```go
func setIfPresent[T any](d *schema.ResourceData, key string, value *T) bool {
	if value == nil {
		return false
	}
	_ = d.Set(key, *value)
	return true
}
```

An omitted field now keeps whatever state already holds. The read collects the
fields it could not set and logs one `tflog.Warn` per repository naming them, so
the trade-off shows in the provider's logs (`TF_LOG_PROVIDER=WARN`) rather than
passing silently. It is deliberately not a warning diagnostic, which would fire
on every plan for a token that is permanently read-only.

The `archived` special-case is **left in place**, along with its `TODO:
Validate this behavior as I can see these fields being returned even when
archived`. It may be redundant, but that is unverified, and removing it would be
a second, untested behaviour change on top of this one. The patch's first
revision (2026-08-01) did remove it; see below.

Plus a unit test, `TestResourceGithubRepositoryRead_omittedFields`, which runs
the read against a stub API with and without the merge settings, and a rewrite
of the docs note, which describes the buggy behaviour and lists seven of the ten
affected attributes.

This is the same approach PR #3144 took for `vulnerability_alerts` and
`security_and_analysis`.

Affected: `allow_auto_merge`, `allow_merge_commit`, `allow_rebase_merge`,
`allow_squash_merge`, `allow_update_branch`, `delete_branch_on_merge`,
`merge_commit_message`, `merge_commit_title`, `squash_merge_commit_message`,
`squash_merge_commit_title`. Unaffected, because GitHub does return them to a
read-only caller: `allow_forking`, `has_downloads`,
`web_commit_signoff_required`.

## What it was verified to do

This revision's commit records its check: a scoped installation token holding
`administration:read` and `contents:read`, against a configuration managing nine
repositories. Every one reported the diff on every plan before, none do after,
and reverting to the released provider reproduces it.

The first revision was measured with the configuration and the read-only broker
token held fixed, the provider binary the only variable:

| | Released 6.13.0 | Patched |
| --- | --- | --- |
| `org-github` | 8-attribute phantom diff | No changes |
| Whole `live/` tree | 9 of 17 units permanently dirty | 17/17 clean |

Removing the override reproduced the diff, confirming the override had been in
effect and that nothing else accounted for the difference.

That revision differs from this one only for archived repositories, whose
fields it set from whatever the response held where this one leaves them to the
special-case, and in not logging the warning. For repositories that are not
archived, both write the same state.

## If this lands upstream

Upstream tracking is integrations/terraform-provider-github#3382. Worth knowing
before opening a PR: **that issue asks for the provider to raise an error**
explaining which permissions are missing. This patch does something different
and better — it preserves prior state, so read-only plans work rather than
failing more clearly. A PR needs to make that argument rather than claim to
close the issue as filed, which is why the commit says `Refs #3382` and not
`Fixes`.

Config-driven changes still show normally after this patch, because state holds
the last value a privileged apply observed. What is given up is drift detection
on those ten attributes while the token remains read-only: a change made in the
GitHub UI will not appear in a plan, and the next apply reverts it silently.
The warning names the affected fields for each repository, but appears only in
the logs.
