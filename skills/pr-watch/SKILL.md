---
name: pr-watch
description: One pass over a GitHub PR's review comments - decide which need a fix, fix and push them, reply to the rest, and re-request Greptile review when it was Greptile's feedback. Run as `/loop /pr-watch <pr-url>` to keep watching.
argument-hint: <pr-url>
---

# Watch a PR for review comments

PR: $ARGUMENTS

This is a single pass. Under `/loop` it repeats; keep each pass idempotent so a re-run never handles the same comment twice.

## 1. Check the PR is still open

```sh
gh pr view <url> --json state,headRefName,baseRefName,author,url
gh api user -q .login
```

If the PR is merged or closed, say so and stop the loop (ScheduleWakeup with `stop: true`, or CronDelete for a fixed-interval loop).

## 2. Find unhandled comments

Pull unresolved review threads with GraphQL so resolved ones drop out:

```sh
gh api graphql -f query='
query($owner:String!,$repo:String!,$n:Int!){repository(owner:$owner,name:$repo){pullRequest(number:$n){
  reviewThreads(first:100){nodes{id isResolved isOutdated path line
    comments(first:50){nodes{databaseId author{login} body createdAt}}}}
  reviews(last:20){nodes{author{login} state body submittedAt}}
}}}' -F owner=<owner> -F repo=<repo> -F n=<number>
```

Also check top-level PR comments (`gh api repos/<owner>/<repo>/issues/<n>/comments`) and review bodies with `CHANGES_REQUESTED` or actionable text.

A thread or comment is unhandled when it's unresolved and its last comment is not from the gh user (me). Skip everything else. Skip Greptile's summary/overview comments; only its inline findings and explicit requests count. If nothing is unhandled, report "no new comments" in one line and end the pass.

## 3. Decide each one

Read the code the comment points at, and enough around it to judge. Question the reviewer the way you'd want to be questioned: bots and humans are both sometimes wrong. A fix is warranted when the comment identifies a real bug, a real risk, or a reasonable change the reviewer clearly wants. It is not warranted when it's wrong about the code, speculative, a style nit that contradicts the repo's conventions (CLAUDE.md / AGENTS.md), or out of scope for this PR.

When a human reviewer asks for something you disagree with, don't silently decline. Reply with your reasoning and leave the thread open for them.

## 4. Fix

```sh
git status --short
gh pr checkout <url>
```

If the working tree has uncommitted changes, stop and tell the user rather than stashing or overwriting them.

Make the minimal fix, matching the surrounding code. Run the repo's tests/typecheck for what you touched. Commit using the repo's and user's commit conventions (one commit per logical fix is fine; group trivial ones), then `git push`. Never force-push.

## 5. Reply

For each handled thread, reply on it (`gh api repos/<owner>/<repo>/pulls/<n>/comments/<comment-id>/replies -f body=...`):

- fixed: one line saying what changed and the commit sha.
- not fixed: one or two sentences on why.

Resolve threads you fixed (GraphQL `resolveReviewThread(input:{threadId:...})`). Leave declined threads from humans unresolved; resolve declined bot threads.

## 6. Re-request Greptile

If any fix this pass came from a Greptile comment (author login contains `greptile`), after pushing post exactly once:

```sh
gh pr comment <url> --body "@greptile review again"
```

Don't post it when nothing was pushed, and don't post it twice in one pass.

## 7. Report

A few lines: what was fixed (with shas), what was declined and why, whether Greptile was re-pinged.
