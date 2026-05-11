---
name: pr-feedback
description: "Address code review comments on a GitHub PR — fix code, push changes, and respond inline to every comment with either the fix or specific reasoning for dismissing. Uses `gh` CLI."
---

# PR Feedback

Address code review feedback on a pull request systematically. Every comment gets a response — either a code fix or an explanation of why it's not being changed.

## Arguments

- PR number (required) — e.g., `/pr-feedback 30`
- If no PR number given, detect from the current branch

## Process

### 1. Fetch comments

Use `gh` to get all review comments on the PR:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments --jq '.[] | {id: .id, path: .path, line: .line, body: .body, user: .user.login, in_reply_to_id: .in_reply_to_id}'
```

Also fetch the PR review body comments (top-level review summaries):

```bash
gh api repos/{owner}/{repo}/pulls/{number}/reviews --jq '.[] | select(.body != "") | {id: .id, body: .body, user: .user.login, state: .state}'
```

Filter out:
- Comments from the PR author (self-comments)
- Comments that already have replies addressing them

### 2. Present comments for triage

Group comments by file. For each comment, show:
- The file and line
- The reviewer's comment
- Your assessment: recommend fix or dismiss, with reasoning

Present this as a summary and ask the user: "Here's what I'd do. Want me to proceed, or adjust any of these?"

### 3. Fix code

For comments marked as fix:
- Read the relevant file
- Make the change
- Run typecheck and tests to verify nothing broke

### 4. Respond to comments

For EVERY comment, post an inline reply via `gh api`:

**For fixes:**
```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments -F body="Fixed — [short description of what changed]." -F in_reply_to={comment_id}
```

**For dismissals:**
```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments -F body="[Specific reasoning why this isn't being changed. Not generic — reference the actual code/context.]" -F in_reply_to={comment_id}
```

**IMPORTANT:** Use `-F` (uppercase), not `-f` (lowercase). The `in_reply_to` field must be sent as an integer, not a string. `-F` sends raw values (integers stay integers), while `-f` wraps everything in quotes (making it a string, which the API rejects).

**For comments nested inside a review body (no per-comment ID):**

Some reviewers (CodeRabbit's "minor comments" block is the canonical case) batch multiple findings inside a single review body. Those nested findings have no individual reply ID — `in_reply_to` won't work. Post a fresh inline review comment at the file/line each finding references instead, never summarize them in a top-level PR comment:

```bash
gh api repos/{owner}/{repo}/pulls/{number}/comments \
  -F body="[Fix or dismissal reasoning, same conventions as above]" \
  -F commit_id="<head sha of the PR branch>" \
  -F path="src/foo.ts" \
  -F line=42 \
  -F side=RIGHT
```

This creates a per-file chat thread the reviewer (and CodeRabbit) can resolve individually. The tracking granularity is the file location.

### 5. Commit and push

If any fixes were made:
- Stage changed files
- Commit using the project's commit-message convention. For conventional commits: `fix: address review feedback on PR #{number}`. For other styles, match what the rest of the PR's branch uses.
- Push to the PR branch

### 6. Summary

Report what was done:
- How many comments addressed (fixed vs dismissed)
- Files changed
- Any comments that need the user's input (ambiguous or requiring a judgment call)

## Rules

- NEVER leave a review comment without a response
- ALWAYS reply inline to each individual comment via `in_reply_to` — never batch multiple responses into a single top-level PR comment. CodeRabbit and other review bots track resolution per-comment, so top-level replies don't resolve individual threads.
- For findings nested inside a review body (no per-comment reply ID), post a NEW inline review comment at the affected file/line — never a top-level PR comment. CodeRabbit's own tip phrases this as "initiate chat on the files or code changes" — reviewers track resolution at the file location, so a top-level summary leaves every finding unresolved no matter how thorough the prose.
- Dismissal responses must be specific — explain WHY with reference to the actual code, not generic "this is fine" responses
- If unsure whether to fix or dismiss, ask the user rather than guessing
- Run typecheck and tests after fixes, before committing — if the project has them
- Don't batch unrelated fixes into one commit if they touch different concerns — but for review feedback on one PR, a single commit is fine
- Use `gh api` for comment replies, not the web UI
