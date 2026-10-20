---
name: pr
description: Push the current branch and open a GitHub PR with a conventional-commit title and a why-focused description, plus a self-review on complex changes. Use when the user runs /pr.
disable-model-invocation: true
---

# Open a PR from the current branch

## 1. Gather context

```sh
gh repo view --json defaultBranchRef,visibility,nameWithOwner
git status --short
git log --oneline <base>..HEAD
git diff <base>...HEAD --stat
git diff <base>...HEAD
```

`<base>` is the default branch unless the user named another. Stop and say so if you're on the base branch.

If there are uncommitted changes (staged, unstaged, or untracked), commit them before going further. Read the diff and split it into multiple commits when it holds separate logical changes (a refactor plus the feature built on it, an unrelated fix that came along); otherwise use one commit. Stage each group by path; when one file holds changes for more than one commit, write a patch with just that commit's hunks and stage it with `git apply --cached` (`git add -p` is interactive and won't work here). Each commit message is a Conventional Commits headline with no body. Skip anything that looks like it shouldn't be committed (secrets, local config, build output) and mention it to the user. Then re-run the log and diff above so the rest of the steps see the new commits.

Stop and say so if there are still no commits ahead of the base.

Read the full diff, and the surrounding code wherever the diff alone doesn't explain why something changed. The description is only as good as your understanding of the change.

## 2. Title

`type(scope): description`, per Conventional Commits. Keep it as short as it can be while still saying what the PR does, lowercase after the colon, no trailing period. The scope is the package or area the PR touches; if it spans several, use the broader area.

Add `!` after the type/scope only when the repo is public (`visibility: PUBLIC`) and the change breaks an API that external users consume. Private repos and internal-only breakage don't get `!`.

## 3. Description

Two things: what changed, and why. Weight it toward the why, especially for anything a reviewer wouldn't guess from the diff. If implementing the feature meant reaching into unrelated packages, say what was changed there and why it was needed.

- Be direct. Say what the PR does and why in the first sentence or two, then give one sentence each to the non-obvious decisions a reviewer needs to know about. Aim for two or three short paragraphs at most, even on a large PR; a small PR gets a couple of sentences.
- Don't narrate the flow step by step or explain how things work internally. Implementation details (validators, retries, byte comparisons, why a helper moved) belong in self-review comments on those lines, not the description.
- Don't add deploy prerequisites, setup steps, or caveats like "these values are starting points to tune later". Stick to what changed and why.
- Don't use bullets. The rare exception is a very large PR where prose would get hard to follow.
- Write like an engineer talking to a teammate. No hype or filler ("robust", "seamless", "comprehensive", "significantly improves"), no summary flourish at the end.
- Don't mention tests, linters, or tools run to verify the change, and don't list the files changed.
- Only point at specific files when the PR touches many files and just one or two of them need real review attention.
- No "Generated with Claude Code" line or other attribution.

## 4. Push and open

```sh
git push -u origin HEAD
gh pr create --base <base> --title "<title>" --body-file <tmpfile>
```

If a PR already exists for the branch, push and update it with `gh pr edit` instead of creating a new one.

## 5. Self-review (complex changes only)

Skip this for small or mechanical PRs. Do it when the change has logic that isn't obvious from reading it: edge cases being handled, ordering that matters, a workaround, a change in an unrelated package, a deliberate choice between alternatives.

Post one COMMENT review with inline comments on the specific lines:

```sh
gh api repos/<owner>/<repo>/pulls/<number>/reviews --input <tmpfile.json>
```

```json
{
  "commit_id": "<HEAD sha>",
  "event": "COMMENT",
  "comments": [
    {"path": "src/foo.py", "line": 42, "side": "RIGHT", "body": "..."}
  ]
}
```

Each comment is one or two casual sentences calling out what the change is, why it's there, or what edge case it handles. Only comment where a reviewer would actually benefit; a handful of good callouts beats one per hunk. `line` must be a line in the diff on the new side.

## 6. Report

Give the user the PR URL, and the title and description you used.
