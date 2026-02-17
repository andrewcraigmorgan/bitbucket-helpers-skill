---
name: bitbucket-helpers
description: Bitbucket PR helpers - ensures default reviewers are added and branches are closed on merge
---

# Bitbucket PR Helpers

This skill provides standard patterns for Bitbucket PR operations.

## CRITICAL: Default Behaviors

When using Bitbucket MCP tools, ALWAYS follow these patterns:

### Creating Pull Requests

**ALWAYS include default reviewers when creating a PR, but filter out the author.**

Bitbucket does not allow the PR author to be added as a reviewer. You must:
1. Fetch default reviewers with their identifying information
2. Identify which one is the current user (author)
3. Filter out the author before adding reviewers to the PR

```
# Step 1: Fetch default reviewers WITH display names to identify them
mcp__bitbucket__bb_get(
    path: "/repositories/{workspace}/{repo-slug}/default-reviewers",
    jq: "values[*].{uuid: uuid, display_name: display_name, nickname: nickname}"
)

# Step 2: Identify the author
# The author is typically the user configured in the MCP (ATLASSIAN_USER_EMAIL)
# or can be determined from the git config user email
# Common author identifiers to filter out:
#   - Andrew Morgan / andrewmorganmtc
#   - (add other team members as needed)

# Step 3: Create PR with ONLY non-author reviewers
# Filter out any reviewer whose display_name or nickname matches the author
mcp__bitbucket__bb_post(
    path: "/repositories/{workspace}/{repo-slug}/pullrequests",
    body: {
        "title": "PR Title",
        "source": {"branch": {"name": "feature/branch-name"}},
        "destination": {"branch": {"name": "staging"}},
        "description": "## Summary\n...",
        "reviewers": [
            {"uuid": "{uuid-of-non-author-reviewer}"}
            // Do NOT include the author's UUID here
        ]
    }
)
```

**Example: If Andrew Morgan is the author:**
```
# Default reviewers returned:
#   - Andrejs Bucnevs: {9584d0d8-29e2-423a-86c7-6288b7b9b443}
#   - Andrew Morgan: {e8731f55-e830-4ff2-891a-ee2b05f3eefa}  <- AUTHOR, exclude this
#
# PR should only include:
"reviewers": [{"uuid": "{9584d0d8-29e2-423a-86c7-6288b7b9b443}"}]
```

**IMPORTANT:** If you try to add the author as a reviewer, Bitbucket will return a 400 error:
> "reviewers: [Author Name] is the author and cannot be included as a reviewer."

### Merging Pull Requests

**ALWAYS set `close_source_branch: true` when merging.**

```
mcp__bitbucket__bb_post(
    path: "/repositories/{workspace}/{repo-slug}/pullrequests/{id}/merge",
    body: {
        "merge_strategy": "merge_commit",
        "close_source_branch": true
    }
)
```

### Common Workspace/Repo Slugs

| Project | Workspace | Repo Slug |
|---------|-----------|-----------|
| Peachy Nursery | mtcmedia | tudu-laravel-api-and-inertia-app |
| CAHER | mtcmedia | caher-staff-dashboard |

### Known Team Members (for author filtering)

| Name | Nickname | UUID |
|------|----------|------|
| Andrew Morgan | andrewmorganmtc | {e8731f55-e830-4ff2-891a-ee2b05f3eefa} |
| Andrejs Bucnevs | Andrejs Bucnevs | {9584d0d8-29e2-423a-86c7-6288b7b9b443} |

When creating a PR, check the git config or assume the MCP user is the author, then exclude their UUID from reviewers.

## Quick Reference

### Create PR with Reviewers (Complete Workflow)

1. Get reviewers with names: `bb_get /repositories/{ws}/{repo}/default-reviewers` with `jq: "values[*].{uuid: uuid, display_name: display_name}"`
2. Identify author (check git config email or known team members table)
3. Filter out author's UUID from the list
4. Create PR with remaining reviewers: `"reviewers": [{"uuid": "..."}, ...]`

### Merge PR with Branch Cleanup

Always include in merge body:
```json
{
    "merge_strategy": "merge_commit",
    "close_source_branch": true
}
```

## Why This Matters

- **Default reviewers**: Ensures proper code review workflow - PRs without reviewers get missed
- **Author filtering**: Bitbucket rejects PRs where the author is listed as a reviewer - failing to filter causes PR creation to fail
- **Close source branch**: Keeps the repository clean - stale branches cause confusion and clutter
