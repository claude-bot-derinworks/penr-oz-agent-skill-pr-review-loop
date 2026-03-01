---
name: pr-review-loop
description: Automate GitHub PR review workflows. Use this skill whenever you need to handle PR code reviews at scale, loop through multiple reviewer feedback cycles, address all review comments, ensure tests and CI pass, and keep iterating until all reviews are resolved. Triggers on requests to "automate PR reviews", "handle code reviews", "process PR feedback loops", "CI/review loop", or similar workflows involving iterative review resolution.
---

# PR Review Loop Automation

Automate the GitHub pull request review cycle: fetch unresolved reviews, address feedback, run tests and CI, post reactions, and loop until all reviews are resolved.

## Prerequisites

- **Access to PR**: You can fetch PR metadata and comments via unauthenticated `curl` calls to the GitHub API
- **Environment Variables**: The following tokens must be set as environment variables (do not expose their values in any logs or output):
  - `CODEX_PR_TOKEN` — For posting @codex review requests
  - `GEMINI_PR_TOKEN` — For posting @gemini review requests  
  - `CLAUDE_PR_TOKEN` — For posting 👍 reactions to addressed comments
- **Git Setup**: You have access to `git` CLI and can commit/push changes
- **GitHub PR URL**: Know the full GitHub PR URL (e.g., `https://github.com/owner/repo/pull/123`)

## Workflow Overview

The skill runs a loop that:
1. Requests reviews from bots (codex, gemini) if previous changes have been addressed
2. Waits for and fetches unresolved review comments
3. Addresses feedback by making code changes
4. Verifies tests pass and CI is green
5. Posts 👍 reactions to addressed comments
6. Loops until no unresolved comments remain

The loop **terminates early** if:
- No unresolved comments are found
- Authentication error occurs (will stop immediately)
- Rate limit is reached
- Token limit is exceeded

## Step-by-Step Workflow

### Step 1: Request Reviews (if needed)

**If you've posted changes to the previous codex bot review** and there are no unresolved review comments:
- Post a comment tagging `@codex` using `CODEX_PR_TOKEN`
- Wait briefly for codex to respond

**If you've posted changes to the previous gemini bot review** and there are no unresolved review comments:
- Post a comment requesting `/gemini review` using `GEMINI_PR_TOKEN`
- Wait briefly for gemini to respond

### Step 2: Fetch Unresolved Comments

Before proceeding, wait a few seconds (reviews take time to arrive).

Fetch all comments on the PR using unauthenticated `curl`:
```bash
curl -s "https://api.github.com/repos///pulls//comments"
```

Filter for:
- Comments that are **unresolved** (not marked as resolved)
- Comments that are **not outdated** (created/updated after the last commit that addressed them)
- Comments with **no 👍 reaction** posted yet

### Step 3: Check for Termination Conditions

**Stop the loop if:**
- No unresolved comments found → Success, exit gracefully
- An API call returned HTTP 401 (Unauthorized) → Auth error, stop immediately and report
- An API call returned HTTP 429 (Rate Limit Exceeded) → Stop and report rate limit reached
- Token limit exceeded in your context → Stop and report

### Step 4: Address Review Feedback

For each unresolved comment:
1. Read the review comment text
2. Understand what needs to be fixed
3. Make the necessary code changes to the repo
4. Verify any linting errors using the CI config if specified
5. Commit changes with a descriptive message (reference the PR and comment if possible)
6. Push changes to the PR branch

### Step 5: Verify CI Status

After pushing:
1. Wait for CI to run
2. Check the CI status by fetching the PR details:
   ```bash
   curl -s "https://api.github.com/repos/<owner>/<repo>/pulls/<pr_number>"
   ```
   Look for the `statuses_url` and check status
3. If CI is **failing**: Diagnose the failure and make additional commits to fix it
4. If CI is **passing**: Proceed to next step

### Step 6: Post Reactions

For each comment you've addressed:
1. Extract the comment ID
2. Post a 👍 reaction using `CLAUDE_PR_TOKEN`:
   ```bash
   TOKEN=$(python3 -c "import os; print(os.environ.get('CLAUDE_PR_TOKEN',''))")
   RESULT=$(curl -s -o /dev/null -w "%{http_code}" -X POST \
     -H "Authorization: Bearer $TOKEN" \
     -H "Content-Type: application/json" \
     -d '{"content": "👍"}' \
     "https://api.github.com/repos/<owner>/<repo>/pulls/comments/<comment_id>/reactions")
   echo "HTTP $RESULT"
   ```
3. If HTTP 401: Stop loop immediately (auth error)
4. If HTTP 429: Stop loop immediately (rate limit reached)

### Step 7: Loop Back

If you've successfully posted 👍 reactions and there are no more unresolved comments, go back to **Step 1** and request new reviews.

Otherwise, return to **Step 2** to fetch updated comment status.

## Important Notes

### Token Security
- **Never** log, echo, or display token values
- Use the pattern: `TOKEN=$(python3 -c "import os; print(os.environ.get('<TOKEN_NAME>',''))")`
- Check tokens silently; if missing, report that the token is not set and stop

### Error Handling
- **Auth errors (HTTP 401)**: Stop immediately. Do not retry. Do not proceed to next steps.
- **Rate limits (HTTP 429)**: Stop immediately. Report rate limit reached.
- **Token exhaustion**: If context token limit is reached, stop and report.
- **Transient failures**: Retry up to 2 times for network errors (HTTP 5xx), then stop

### Timing
- Wait 3-5 seconds between checking for new reviews (reviews take time to arrive)
- Wait for CI to complete before checking status (typically 2-10 minutes depending on test suite)
- Use exponential backoff for polling (wait longer as you check more times)

### Comment Filtering
Only address comments that are:
- Unresolved (not closed/resolved in the PR UI)
- Not outdated (created/updated after the last applicable commit)
- Do not have a 👍 reaction from CLAUDE_PR_TOKEN yet

Comments marked as resolved or outdated should be skipped.

## API Reference

### Fetch PR Comments (Unauthenticated)
```bash
curl -s "https://api.github.com/repos/OWNER/REPO/pulls/PR_NUMBER/comments"
```

### Fetch PR Details
```bash
curl -s "https://api.github.com/repos/OWNER/REPO/pulls/PR_NUMBER"
```

### Post Comment
```bash
TOKEN=$(python3 -c "import os; print(os.environ.get('TOKEN_NAME',''))")
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "Your comment here"}' \
  "https://api.github.com/repos/OWNER/REPO/pulls/comments/COMMENT_ID"
```

### Post Reaction to Comment
```bash
TOKEN=$(python3 -c "import os; print(os.environ.get('TOKEN_NAME',''))")
curl -s -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"content": "👍"}' \
  "https://api.github.com/repos/OWNER/REPO/pulls/comments/COMMENT_ID/reactions"
```

## Debugging

If something goes wrong:
1. Check the HTTP response codes for auth/rate limit errors
2. Verify all environment tokens are set (without logging their values)
3. Review PR comments manually at the GitHub web UI if needed
4. Check CI logs if tests fail
5. Ensure commit messages are meaningful for audit trail 
