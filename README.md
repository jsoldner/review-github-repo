# /review-github-repo

A Claude Code slash command that performs a multi-agent security, quality, and architecture review of any public Git repository.

## What It Does

Point it at any repo URL and it:

1. **Scouts** the repo to determine which review domains apply
2. **Dispatches up to 6 parallel review agents** — Security, Quality, Architecture, Performance, Dependencies, Documentation — based on what's present
3. **Merges and deduplicates** findings across all agents
4. **Produces a prioritized report** with a Verdict, Executive Summary, and findings grouped by severity (Critical → Important → Minor)

Reports are saved to `Owner's Inbox/` in your working directory.

## Installation

Copy `review-github-repo.md` into your Claude Code commands directory:

```bash
cp review-github-repo.md ~/.claude/commands/review-github-repo.md
```

## Usage

```
/review-github-repo <repo-url>
```

**Examples:**

```
/review-github-repo https://github.com/expressjs/express
/review-github-repo https://github.com/pallets/flask
/review-github-repo git@github.com:your-org/your-repo.git
```

## Output

The report is written to:

```
Owner's Inbox/YYYY-MM-DD-review-<repo-name>.md
```

The directory is created automatically if it doesn't exist.

## Report Structure

```
# Code Review: <repo-name>

## Verdict
Recommend Use: Yes / Yes, with caveats / No

## Scope
Table showing which domains ran and why.

## Executive Summary
One-paragraph overall assessment.

## Critical Findings
## Important Findings
## Minor Findings
## Clean Areas
```

## Requirements

- Claude Code (claude.ai/code or the CLI)
- Git installed and available in your PATH
- A Claude model with multi-agent tool use support (Sonnet or Opus recommended)

## Customizing the Output Path

By default, reports go to `Owner's Inbox/` relative to your current working directory. To change this, edit the output path in Step 5 of the skill file.

## License

MIT
