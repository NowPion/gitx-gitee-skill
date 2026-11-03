# Pull requests

Use the standalone workflow for `gitx pr [base]`. Ship uses only the shared title, body, and readiness conventions below; its own reference controls publication sequencing and repository selection.

PR operations use the Gitee CLI and follow the official Gitee skill conventions: always pass `--json` and `--no-tui`, never pass `--ai`, generate titles and bodies yourself from the commits and diff, and require explicit user confirmation before merging.

## Standalone workflow

1. Require a remote named `origin`, a named current branch, a Gitee repository, and an authenticated Gitee CLI (verify with `gitee auth status --no-tui`). If any is unavailable, explain what is missing and do not create a PR.
2. Resolve the base in this order: supplied `[base]`, configured `prBase`, then the default branch from `origin/HEAD`. Verify the resolved branch exists on `origin`; do not fall back when a supplied or configured branch is missing. If no base can be determined, ask the user which one to use. Never hardcode `main`.
3. Inspect the working tree, commits, and diff from the resolved base branch to the current branch. Do not include uncommitted changes in the PR. If the current branch is the base branch or has no commits ahead of it, stop and explain why.
4. Check whether a PR already exists for the current branch and resolved base branch with `gitee pr list --state open --head <current-branch> --base <resolved-base> --json --no-tui`, adding `-R <owner>/<repo>` when outside the repository. If it does, return its `html_url` and do not create another one.
5. Push the current branch to `origin` when needed. If it has no upstream, create one with `git push -u origin <branch>` because `gitx pr` explicitly requests publication. Never force-push.
6. Generate the title, body, and readiness using the shared conventions below.
7. Create the PR with `gitee pr create -t <generated-title> -b <generated-body> --base <resolved-base> --head <current-branch> --json --no-tui`, adding `--draft` when required below, and return the `html_url` from the JSON response. Never pass `--ai`; generate the title and body yourself. If creation fails with 422, a PR for this branch probably already exists—re-check with `gitee pr list` before retrying.

## Title, body, and readiness

Generate a concise PR title from the commits and diff. Generate a normal Markdown body using this structure, with only facts supported by the changes:

```md
## Summary

- <actual change>

## Testing

- <checks run during this task, or "Not run (not requested)">
```

Add `--draft` if explicitly requested or if configured `draftPR` is true without an explicit readiness override; otherwise create it ready for review. Do not change an existing PR's readiness merely to match a preference.

## PR management operations

`gitx pr <subcommand>` routes the management verbs `list`, `view`, `edit`, `diff`, `review`, `comment`, `merge`, `close`, `reopen`, `checkout`, and `fetch` here. PR numbers are integers (e.g. `42`), unlike alphanumeric issue identifiers. All commands require an authenticated Gitee CLI, pass `--json` where supported plus `--no-tui`, and never use `--ai`.

```bash
gitee pr list --state open --json --no-tui                          # also --author/--assignee/--tester/--head/--base/--labels/--milestone-number/--since/--sort/--direction/--limit/--page
gitee pr view 42 --json --no-tui                                     # fields: number,title,body,state,draft,mergeable,head,base,user,assignees,testers,html_url,created_at,merged_at
gitee pr edit 42 -t "New title" -b "Updated body" --json --no-tui    # at least one of -t/-b/--draft; --draft=false unsets a draft
gitee pr diff 42 --no-tui                                            # unified diff; --json=filename,status,additions,deletions for a structured summary
gitee pr review 42 --no-tui                                          # approve; -b "LGTM" attaches a comment; --force is the admin override
gitee pr comment 42 -b "..." --json --no-tui                         # -b is required or the command blocks on an editor
gitee pr merge 42 --json --no-tui                                    # after confirmation; --method merge|squash|rebase, --delete-branch
gitee pr close 42 --json --no-tui
gitee pr reopen 42 --json --no-tui
gitee pr checkout 42 --no-tui                                        # fetch and switch; --branch <name> renames pr_<number>; --force re-fetches
gitee pr fetch 42 --no-tui                                           # fetch without switching
```

Merging is visible and hard to undo: show the PR title, author, head → base, merge method, and whether the source branch will be deleted, then wait for explicit confirmation before running `pr merge`. `checkout` and `fetch` are local git operations and do not support `--json`. When an operation has no direct command, fall back to the [Gitee API](gitee-api.md) reference.

Common errors: `--title is required in non-interactive mode` (create without `-t`), `422` (a PR already exists for the branch — check with `pr list`), a hung command (`pr comment` without `-b`), `404` (wrong PR number), `405/409` on merge (conflicts or unmet requirements), `403` on review (insufficient permission), `could not detect current branch` (not in a git repository — add `-R owner/repo`), `authentication required` (run `gitee auth login`).
