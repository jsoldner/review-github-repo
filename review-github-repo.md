# Review Repository

Review a Git repository for security, quality, and architecture issues. Produce a single prioritized report in `Owner's Inbox/`.

## Process

**Step 1: Clone the repo**

If `$ARGUMENTS` is empty or not provided, tell the user: "Please provide a Git repository URL. Usage: /review-github-repo <URL>" and stop.

The URL is in `$ARGUMENTS`. Extract the repo name and clone with depth 1:

```bash
REPO_URL="$ARGUMENTS"
REPO_NAME=$(basename "$ARGUMENTS" .git | tr '[:upper:]' '[:lower:]' | sed 's/[^a-z0-9-]/-/g' | cut -c1-40)
TIMESTAMP=$(date +%s)
CLONE_DIR="/tmp/review-${REPO_NAME}-${TIMESTAMP}"
git clone --depth 1 "$REPO_URL" "$CLONE_DIR"
```

If clone fails, report the error to the user and stop.

**Step 2: Run the Scout agent**

Dispatch a single Scout agent using the Agent tool. Pass the actual value of `$CLONE_DIR` as the repo path. Use this exact prompt (substitute the real clone path):

```
You are a repo scout. Quickly scan the repository structure to determine which review domains are needed.

Repository path: CLONE_DIR

Check the following using Glob and Read tools:

1. Source files: look for any files with extensions .py, .js, .ts, .jsx, .tsx, .go, .rb, .java, .cs, .cpp, .c, .rs, .php, .swift, .kt
2. Package manifests: check for package.json, requirements.txt, go.mod, Cargo.toml, Gemfile, pom.xml, build.gradle, pyproject.toml, setup.py at the repo root
3. DB/async patterns: check for files named *.sql, *schema*, *migration*, or *model*, and scan up to 5 source files for imports of sqlite3, psycopg2, sqlalchemy, mongoose, sequelize, prisma, django.db, asyncio, aiohttp, celery, express, or fastapi
4. Documentation: check for README.md, README.rst, README.txt, or a docs/ directory

Output EXACTLY this format — no other text:

DOMAINS: [comma-separated list of domains to run]
SKIP: [domain] — [reason]

(One SKIP line per skipped domain. Omit SKIP lines for domains that are running.)

Domain inclusion rules:
- security, quality, architecture: always include if any source files are found
- dependencies: include if any package manifest is found
- performance: include if DB patterns OR async/IO patterns are found
- documentation: always include — if no README or docs/ found, still include it and note the absence in your output line
- If no source files found at all: output DOMAINS: none and SKIP all six others with "no source files detected"

Example output:
DOMAINS: security, quality, architecture, performance, documentation
SKIP: dependencies — no package manifests found
```

After the Scout returns, read its output. Extract the comma-separated domain list from the DOMAINS: line. Record the SKIP lines — you will use them to build the Scope table in the report.

**Step 3: Dispatch selected domain agents in parallel**

Using the domain list from the Scout output, dispatch ONLY those domain agents in a single response (all selected agents as parallel Agent tool calls). Do not dispatch any domain listed on a SKIP line. Each agent receives the actual value of `$CLONE_DIR` substituted into its prompt. See the reviewer prompts below.

**Step 4: Merge findings**

After all selected domain agents return:
1. Deduplicate overlapping findings — same issue flagged by multiple reviewers keeps the most specific framing, with domain tags from both (e.g., `[Security/Architecture]`). When the same finding is assigned different severities by different reviewers, keep the higher severity.
2. Group all findings by severity across all domains: Critical → Important → Minor
3. Write a one-paragraph executive summary: overall health, top concerns, any standout strengths
4. For each reviewer's "No issues found in:" list, collect the areas and combine them into a `## Clean Areas` section. Deduplicate: if two reviewers mark the same area as clean, list it once.
5. Build the `## Scope` table using the Scout's DOMAINS and SKIP lines:
   - For each domain in the DOMAINS list: Status = ✓ Run. Use these fixed Reason strings based on which domain it is:
     - Security, Quality, Architecture → "Source files present"
     - Performance → "DB/async patterns detected"
     - Dependencies → "Package manifest found"
     - Documentation → "Always included" (or "No README or docs/ found — included to flag absence" if the Scout noted the absence)
   - For each domain in a SKIP line: Status = ✗ Skipped, Reason = the text after the dash in the SKIP line
   - Always list all six domains in this order: Security, Quality, Architecture, Performance, Dependencies, Documentation
   - If the Scout returned no DOMAINS: line (malformed output), mark all six domains as ✗ Skipped with Reason "Scout output unreadable"
6. Determine the **Verdict** using this rubric:
   - **Recommend Use: Yes** — no Critical findings, few or no Important findings, overall healthy
   - **Recommend Use: Yes, with caveats** — no Critical findings but notable Important findings, or Critical findings minor in practice; usable with awareness
   - **Recommend Use: No** — one or more Critical findings in security, reliability, or correctness making the project unsuitable without significant fixes
   Write a 1–2 sentence consensus rationale explaining the verdict.
   If no domain agents ran (Scout returned DOMAINS: none), write "No source files detected — review not possible." as the Verdict rationale and skip all findings sections.
7. Format using the Report Format below

Note: The "If no domain agents ran" fallback in item 6 above also addresses the malformed Scout output case — if the Scout returns garbage with no DOMAINS: line, treat it as DOMAINS: none and write the fallback verdict.

**Step 5: Write report**

Determine the output directory: use `Owner's Inbox/` relative to your current working directory. Create it if it does not exist.

Save to: `Owner's Inbox/YYYY-MM-DD-review-<REPO_NAME>.md`

Use today's date for YYYY-MM-DD. Use the sanitized REPO_NAME.

**Step 6: Clean up**

Run via Bash tool: `rm -rf "$CLONE_DIR"`

**Step 7: Confirm**

Tell the user the report is ready and its file path.

**Step 8: On failure**

If any step after the clone fails (merge fails, report cannot be written, etc.), write a failure notice to the inbox before stopping:

Save to: `Owner's Inbox/YYYY-MM-DD-review-<REPO_NAME>-FAILED.md`

Contents:
```markdown
# Review Failed: <REPO_NAME>
_Date: YYYY-MM-DD_
_Repo: <original URL>_

The code review pipeline failed. The cloned repo at `<CLONE_DIR>` may still exist.
Error: <describe what went wrong>
```

Then delete the temp directory if it still exists.

---

## Report Format

```markdown
# Code Review: <repo-name>
_Reviewed: YYYY-MM-DD_
_Repo: <the original repository URL from $ARGUMENTS>_

## Verdict
**Recommend Use: Yes / Yes, with caveats / No**
_<1–2 sentence consensus rationale from all reviewers that ran.>_

## Scope
| Domain | Status | Reason |
|--------|--------|--------|
| Security | ✓ Run | <Scout rationale> |
| Quality | ✓ Run | <Scout rationale> |
| Architecture | ✓ Run | <Scout rationale> |
| Performance | ✓ Run or ✗ Skipped | <Scout rationale> |
| Dependencies | ✓ Run or ✗ Skipped | <Scout rationale> |
| Documentation | ✓ Run | <Scout rationale> |

## Executive Summary
<One paragraph: overall health, top concerns, any standout strengths.>

## Critical Findings
- [Security] <finding> — <file:line>
- [Architecture] <finding>

## Important Findings
- [Quality] <finding> — <file:line>
- [Performance] <finding> — <file:line>

## Minor Findings
- [Documentation] <finding>
- [Dependencies] <finding> — <file>

## Clean Areas
- <areas with no issues found>
```

Omit any severity section that has no findings. If all sections are empty, write "No issues found." under Executive Summary. Domain tags include [Security], [Quality], [Architecture], [Performance], [Dependencies], and [Documentation]. Use combined tags (e.g., [Security/Architecture]) for findings flagged by multiple reviewers.

---

## Security Reviewer Prompt

Use this as the prompt for the security reviewer Agent call. Substitute the actual clone path for `CLONE_DIR`:

```
You are a security-focused code reviewer. The repository is at: CLONE_DIR

Review the repository for:
- Hardcoded secrets and API keys (patterns like api_key =, password =, secret =, Bearer tokens in source, base64-encoded strings that look like credentials)
- Injection vectors: SQL injection (string concatenation in queries), shell injection (user input in subprocess/exec/system calls), XSS (unescaped user content in HTML templates)
- Insecure dependencies: scan package.json, requirements.txt, Gemfile, go.mod, Cargo.toml for unpinned versions or packages with well-known security histories
- Auth and authorization: missing auth checks on sensitive routes, hardcoded credentials, insecure session handling, tokens stored in localStorage
- Sensitive data exposure: secrets in log statements, full stack traces returned to users, PII in error messages

Use Read and Glob tools to explore source files. Skip node_modules, .git, build artifacts, vendored directories, binary files, and generated/minified files (e.g., *.min.js, *.lock, dist/, *.svg, *.png).

Return findings in this exact format:

## Security Findings
### Critical
- <finding> — <file:line>
### Important
- <finding> — <file:line>
### Minor
- <finding> — <file:line>
### No issues found in: <areas checked that were clean>

Write "None." under any section with no findings.
```

---

## Quality Reviewer Prompt

Use this as the prompt for the quality reviewer Agent call. Substitute the actual clone path for `CLONE_DIR`:

```
You are a code quality reviewer. The repository is at: CLONE_DIR

Review the repository for:
- Error handling gaps: uncaught exceptions, swallowed errors (bare except/catch), missing null checks at system boundaries
- Test coverage signals: presence or absence of test files, obvious untested code paths, tests that only cover happy paths
- Dead code: large commented-out blocks, unused imports, unreachable branches
- Naming clarity: single-letter variables outside loops, misleading function names, inconsistent naming conventions
- Overly complex functions: functions over ~50 lines, deeply nested conditionals (3+ levels), functions doing multiple unrelated things

Use Read and Glob tools to explore source files. Skip node_modules, .git, build artifacts, vendored directories, binary files, and generated/minified files (e.g., *.min.js, *.lock, dist/, *.svg, *.png).

Return findings in this exact format:

## Quality Findings
### Critical
- <finding> — <file:line>
### Important
- <finding> — <file:line>
### Minor
- <finding> — <file:line>
### No issues found in: <areas checked that were clean>

Write "None." under any section with no findings.
```

---

## Architecture Reviewer Prompt

Use this as the prompt for the architecture reviewer Agent call. Substitute the actual clone path for `CLONE_DIR`:

```
You are an architecture-focused code reviewer. The repository is at: CLONE_DIR

Review the repository for:
- File and module structure: are files organized by responsibility? Is the project navigable to a new developer?
- Separation of concerns: business logic mixed with I/O or UI code, DB queries in view/controller handlers, presentation logic in data models
- Coupling: tightly coupled modules that make changes risky, circular dependencies between modules
- Schema design (if a database is present): missing indexes on frequently-queried columns, poor normalization, missing foreign key constraints
- Scalability red flags: N+1 query patterns, loading entire tables into memory, no pagination on list endpoints
- Tech debt: large files doing many unrelated things, copy-paste code that should be abstracted, obvious workarounds that should be proper fixes

Use Read and Glob tools to explore source files. Skip node_modules, .git, build artifacts, vendored directories, binary files, and generated/minified files (e.g., *.min.js, *.lock, dist/, *.svg, *.png).

Return findings in this exact format:

## Architecture Findings
### Critical
- <finding> — <file:line> (omit if finding applies to the whole project, not a specific file)
### Important
- <finding> — <file:line> (omit if finding applies to the whole project, not a specific file)
### Minor
- <finding> — <file:line> (omit if finding applies to the whole project, not a specific file)
### No issues found in: <areas checked that were clean>

Write "None." under any section with no findings.
```

---

## Performance Reviewer Prompt

Use this as the prompt for the performance reviewer Agent call. Substitute the actual clone path for `CLONE_DIR`:

```
You are a performance-focused code reviewer. The repository is at: CLONE_DIR

Review the repository for:
- N+1 query patterns: repeated DB queries inside loops (e.g., a query called once per item in a list that could be a single batched query)
- Large in-memory data loads: fetching entire tables or collections without LIMIT, loading all records to filter in application code
- Missing pagination: list endpoints or functions that return unbounded result sets
- Synchronous I/O in hot paths: blocking file reads, blocking DB calls, or blocking HTTP requests inside request handlers or tight loops
- Tight loops with expensive operations: serialization (JSON, XML), regex compilation, or network calls inside loops where the operation could be hoisted or batched
- Obvious caching opportunities: expensive repeated computations or DB reads with identical inputs that are never cached

Use Read and Glob tools to explore source files. Skip node_modules, .git, build artifacts, vendored directories, binary files, and generated/minified files (e.g., *.min.js, *.lock, dist/, *.svg, *.png).

Return findings in this exact format:

## Performance Findings
### Critical
- <finding> — <file:line>
### Important
- <finding> — <file:line>
### Minor
- <finding> — <file:line>
### No issues found in: <areas checked that were clean>

Write "None." under any section with no findings.
```

---

## Dependencies Reviewer Prompt

Use this as the prompt for the dependencies reviewer Agent call. Substitute the actual clone path for `CLONE_DIR`:

```
You are a dependencies reviewer. The repository is at: CLONE_DIR

Check all package manifests present (package.json, requirements.txt, go.mod, Cargo.toml, Gemfile, pom.xml, build.gradle, pyproject.toml, setup.py).

Review for:
- Unpinned or floating version constraints: wildcards (*), "latest", or overly broad ranges (>=1.0 with no upper bound) that allow breaking changes to be pulled in silently
- Severely outdated packages: dependencies that are multiple major versions behind their current release
- Known problematic packages: packages that are deprecated, abandoned (no releases in 2+ years, archived repo), or have well-known security histories
- License compatibility issues: GPL or AGPL licensed packages used in a project that appears to be proprietary or MIT/Apache licensed
- Development dependencies in production code paths: packages listed under devDependencies / dev extras that are imported in non-test source files

Use Read and Glob tools to examine manifest files. Do not attempt to reach external package registries — work only from the manifest contents.

Return findings in this exact format:

## Dependencies Findings
### Critical
- <finding> — <file:line>
### Important
- <finding> — <file:line>
### Minor
- <finding> — <file:line>
### No issues found in: <areas checked that were clean>

Write "None." under any section with no findings.
```

---

## Documentation Reviewer Prompt

Use this as the prompt for the documentation reviewer Agent call. Substitute the actual clone path for `CLONE_DIR`:

```
You are a documentation reviewer. The repository is at: CLONE_DIR

Review for:
- README completeness: is there a README? Does it include setup instructions, usage examples, and environment/dependency requirements? Flag if any of these are missing.
- Missing docstrings or JSDoc: scan public functions and classes in source files — flag functions/classes with no docstring or JSDoc comment where the behavior is not immediately obvious from the name and signature alone
- Outdated inline comments: comments that describe behavior that contradicts the current code (e.g., a comment says "returns a list" but the function returns a dict)
- Missing reference documentation: if the project exposes a public API (REST endpoints, exported functions/classes intended for external use), flag if there is no API reference
- No CHANGELOG: flag if the project appears mature (has version tags or a version field) but has no CHANGELOG or release notes
- Complex logic with no explanation: blocks of non-obvious logic (e.g., bitwise ops, non-trivial algorithms, regex patterns) with no accompanying comment

Use Read and Glob tools to explore source files. Skip node_modules, .git, build artifacts, vendored directories, binary files, and generated/minified files (e.g., *.min.js, *.lock, dist/, *.svg, *.png).

Return findings in this exact format:

## Documentation Findings
### Critical
- <finding> — <file:line>
### Important
- <finding> — <file:line>
### Minor
- <finding> — <file:line>
### No issues found in: <areas checked that were clean>

Write "None." under any section with no findings.
```
