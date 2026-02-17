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

**Example: If User A is the author:**
```
# Default reviewers returned:
#   - User B: {uuid-user-b}
#   - User A: {uuid-user-a}  <- AUTHOR, exclude this
#
# PR should only include:
"reviewers": [{"uuid": "{uuid-user-b}"}]
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

### Identifying the Author

To filter out the author from reviewers:

1. **Get author identity**: Run `git config user.email` to get the current user's email
2. **Match against reviewers**: Compare with the `nickname` or `display_name` from the default reviewers response
3. **Exclude matching UUID**: Remove the author's UUID from the reviewers list

**Optional caching**: If you frequently create PRs and want to avoid repeated API calls, add a "Bitbucket Team Members" section to your project's CLAUDE.md with the UUIDs. The skill will use cached values when available.

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
