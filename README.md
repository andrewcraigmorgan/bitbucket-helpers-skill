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

### 2. Dynamic Author Detection

Identifies the PR author by checking `git config user.email` and matching against the default reviewers response. No hardcoded team member lists required.

**Optional**: For performance, you can cache team member UUIDs in your project's CLAUDE.md under a "Bitbucket Team Members" section.

### 3. Branch Cleanup on Merge

Ensures `close_source_branch: true` is always set when merging PRs to keep the repository clean.

### 4. Workspace/Repo Detection

Extracts workspace and repo slug from git remote URL or project's CLAUDE.md.

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
