# Bitbucket Helpers Skill

A Claude Code skill that provides standard patterns for Bitbucket PR operations, ensuring consistent workflows and avoiding common pitfalls.

## Installation

Copy the `SKILL.md` file to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/bitbucket-helpers
cp SKILL.md ~/.claude/skills/bitbucket-helpers/
```

## Features

### 1. Default Reviewers with Author Filtering

When creating PRs, Bitbucket doesn't allow the PR author to be listed as a reviewer. This skill documents the correct workflow:

1. Fetch default reviewers **with display names** (not just UUIDs)
2. Identify which reviewer is the current user (author)
3. Filter out the author before adding reviewers to the PR

This prevents the common error:
> "reviewers: [Author Name] is the author and cannot be included as a reviewer."

### 2. Known Team Members Reference

Includes a table of team members with their UUIDs for quick reference when filtering:

| Name | Nickname | UUID |
|------|----------|------|
| Andrew Morgan | andrewmorganmtc | {e8731f55-e830-4ff2-891a-ee2b05f3eefa} |
| Andrejs Bucnevs | Andrejs Bucnevs | {9584d0d8-29e2-423a-86c7-6288b7b9b443} |

### 3. Branch Cleanup on Merge

Ensures `close_source_branch: true` is always set when merging PRs to keep the repository clean.

### 4. Common Workspace/Repo Slugs

Quick reference for frequently used repositories.

## Usage

The skill is automatically loaded when working with Bitbucket MCP tools. It provides patterns for:

### Creating a PR

```
# Step 1: Fetch reviewers with identifying info
mcp__bitbucket__bb_get(
    path: "/repositories/{workspace}/{repo}/default-reviewers",
    jq: "values[*].{uuid: uuid, display_name: display_name, nickname: nickname}"
)

# Step 2: Create PR excluding author from reviewers
mcp__bitbucket__bb_post(
    path: "/repositories/{workspace}/{repo}/pullrequests",
    body: {
        "title": "PR Title",
        "source": {"branch": {"name": "feature/branch"}},
        "destination": {"branch": {"name": "staging"}},
        "reviewers": [{"uuid": "{non-author-uuid}"}]
    }
)
```

### Merging a PR

```
mcp__bitbucket__bb_post(
    path: "/repositories/{workspace}/{repo}/pullrequests/{id}/merge",
    body: {
        "merge_strategy": "merge_commit",
        "close_source_branch": true
    }
)
```

## Requirements

- Claude Code with MCP support
- Bitbucket MCP server configured (`@aashari/mcp-server-atlassian-bitbucket` or similar)

## License

MIT
