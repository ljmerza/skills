---
name: review-copilot
description: Review Copilot PR comments, triage, implement approved ones, commit, push, and resolve threads
disable-model-invocation: true
user-invocable: true
argument-hint: [pr-number]
---

# Review Copilot PR Comments

You are reviewing GitHub Copilot's PR review comments, deciding which are worth implementing, and handling the full workflow.

## Step 1: Identify the PR

If `$ARGUMENTS` is provided, use that as the PR number. Otherwise, detect the PR from the current branch:

```
gh pr view --json number -q .number
```

Store the PR number and the repo owner/name (get via `gh repo view --json nameWithOwner -q .nameWithOwner`).

## Step 2: Fetch unresolved Copilot review threads

Use the GitHub GraphQL API to fetch all review threads. Paginate if needed (use `hasNextPage` / `endCursor`).

```
gh api graphql -f query='
query($owner: String!, $repo: String!, $pr: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $pr) {
      reviewThreads(first: 100) {
        nodes {
          id
          isResolved
          path
          line
          originalLine
          comments(first: 5) {
            nodes {
              author { login }
              body
            }
          }
        }
      }
    }
  }
}' -f owner=OWNER -f repo=REPO -F pr=NUMBER
```

Filter to only threads where:
- `isResolved` is `false`
- The first comment's author login is `copilot-pull-request-reviewer`

If no unresolved Copilot comments found, tell the user and stop.

## Step 3: Triage each comment

For each unresolved Copilot comment, read the relevant file and code context to understand the suggestion. Then categorize it:

- **Recommend**: The suggestion improves code quality, fixes a real bug, or aligns with project conventions. Briefly explain why.
- **Skip**: The suggestion is pedantic, wrong, already handled elsewhere, or not worth the churn. Briefly explain why.

## Step 4: Present recommendations and ask the user

Format a clear summary table for the user like:

```
## Copilot Review Comments for PR #N

| # | File | Suggestion | Recommendation | Reason |
|---|------|-----------|----------------|--------|
| 1 | path/to/file.py | Add type hints | Recommend | Matches project conventions |
| 2 | path/to/other.js | Use optional chaining | Skip | Browser compat concern |
```

Then use AskUserQuestion to ask:
- "Which comments should I implement? (e.g. 'all', '1,2,4', 'recommended', or 'none')"

If the user says "none", resolve all threads as ignored and stop.

## Step 5: Implement the approved changes

For each approved comment:
1. Read the file at the relevant path
2. Understand the Copilot suggestion in context
3. Implement the change properly (don't blindly copy suggested code — adapt it to fit the codebase)
4. If the comment asks for new tests, write them following existing test patterns

## Step 6: Commit and push

After all changes are implemented:
1. Stage only the changed files (not unrelated files)
2. Commit with message: `Address Copilot review feedback on PR #N`
3. Push to the current branch

## Step 7: Resolve all threads

For each Copilot comment thread (both implemented AND skipped):
- Use the GraphQL mutation to resolve the thread:

```
gh api graphql -f query='
mutation($threadId: ID!) {
  resolveReviewThread(input: { threadId: $threadId }) {
    thread { isResolved }
  }
}' -f threadId=THREAD_ID
```

Report a summary: "Implemented N comments, skipped M comments. All threads resolved."

## Step 8: Re-request Copilot review

Re-request a review from Copilot so it can evaluate the new changes (requires gh CLI v2.88.0+).

First, check if Copilot has already reviewed the PR by looking at the review threads fetched in Step 2. If there were any Copilot review threads (resolved or not), Copilot has already reviewed and needs a re-request:

```
# Copilot already reviewed — remove and re-add to trigger a fresh review
gh pr edit NUMBER --remove-reviewer @copilot
gh pr edit NUMBER --add-reviewer @copilot
```

If Copilot has NOT reviewed the PR yet (no threads from `copilot-pull-request-reviewer`), simply add it:

```
# First-time request
gh pr edit NUMBER --add-reviewer @copilot
```

If the gh CLI version is too old (< 2.88.0), tell the user to request the Copilot review manually from the GitHub PR UI (click the refresh icon next to Copilot in the Reviewers section, or add Copilot as a reviewer).
