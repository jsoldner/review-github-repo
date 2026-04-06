# Review Repository

Review a Git repository for security, quality, and architecture issues. Produce a single prioritized report in `Owner's Inbox/`.

## Process

**Step 1: Clone the repo**

If `$ARGUMENTS` is empty or not provided, tell the user: "Please provide a Git repository URL. Usage: /review-repo <URL>" and stop.

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

Save to: `Owner's Inbox/YYYY-MM-DD-review-<REPO_NAME>.html`

Use today's date for YYYY-MM-DD. Use the sanitized REPO_NAME.

**Step 6: Clean up**

Run via Bash tool: `rm -rf "$CLONE_DIR"`

**Step 7: Confirm**

Tell the user the report is ready and its full file path.

**Step 8: On failure**

If any step after the clone fails (merge fails, report cannot be written, etc.), write a failure notice to the inbox before stopping:

Save to: `Owner's Inbox/YYYY-MM-DD-review-<REPO_NAME>-FAILED.html`

Contents:
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Review Failed: <REPO_NAME></title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; font-size: 14px; line-height: 1.6; color: #1a1a2e; background: #f4f5f7; padding: 32px 16px; }
    .container { max-width: 960px; margin: 0 auto; }
    .header { background: #1a1a2e; color: #fff; border-radius: 10px 10px 0 0; padding: 28px 32px 24px; }
    .header h1 { font-size: 22px; font-weight: 700; margin-bottom: 6px; }
    .header-meta { font-size: 12px; color: #9ca3af; }
    .banner { background: #7f1d1d; border-top: 1px solid #991b1b; padding: 18px 32px; display: flex; align-items: flex-start; gap: 14px; color: #fecaca; }
    .banner-icon { font-size: 22px; flex-shrink: 0; margin-top: 1px; }
    .banner-label { font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 1px; opacity: 0.75; margin-bottom: 4px; }
    .banner-title { font-size: 17px; font-weight: 700; color: #fff; margin-bottom: 6px; }
    .banner p { font-size: 13px; line-height: 1.55; opacity: 0.9; margin-bottom: 6px; }
    .card { background: #fff; padding: 28px 32px; border-left: 1px solid #e5e7eb; border-right: 1px solid #e5e7eb; border-bottom: 1px solid #e5e7eb; border-radius: 0 0 10px 10px; }
    code { font-family: "SF Mono", "Fira Code", Consolas, monospace; background: #f3f4f6; padding: 1px 5px; border-radius: 3px; font-size: 12px; color: #374151; }
  </style>
</head>
<body>
<div class="container">
  <div class="header">
    <div class="header-meta" style="margin-bottom:8px;">Code Review Report &nbsp;·&nbsp; YYYY-MM-DD</div>
    <h1><REPO_NAME></h1>
    <div class="header-meta"><original URL></div>
  </div>
  <div class="banner">
    <div class="banner-icon">&#9888;</div>
    <div>
      <div class="banner-label">Review Failed</div>
      <div class="banner-title">Pipeline Error</div>
      <p>The code review pipeline failed. The cloned repo at <code><CLONE_DIR></code> may still exist.</p>
      <p><strong>Error:</strong> <describe what went wrong></p>
    </div>
  </div>
  <div class="card">
    <p style="color:#6b7280;font-size:13px;">Review could not be completed. Please check the error above and retry.</p>
  </div>
</div>
</body>
</html>
```

Then delete the temp directory if it still exists.

---

## Report Format

Produce a self-contained HTML file. Before writing, count your findings: tally Critical, Important, and Minor totals — you will need these numbers in three places (stats pills, section heading badges, and the verdict banner area). Use this template:

**Tag rules:**
- Single-domain finding: `<span class="tag tag-security">Security</span>` (use the matching class below)
- Multi-domain finding: `<span class="tag tag-multi">Security / Architecture</span>` (one combined tag, slash-separated)
- Tag classes: `tag-security`, `tag-quality`, `tag-architecture`, `tag-performance`, `tag-dependencies`, `tag-documentation`, `tag-multi`

**Verdict banner:**
- "Recommend Use: No" → `class="verdict verdict-no"`, icon `&#9888;`
- "Recommend Use: Yes, with caveats" → `class="verdict verdict-caveats"`, icon `&#9888;`
- "Recommend Use: Yes" → `class="verdict verdict-yes"`, icon `&#10003;`

**Finding structure** — every finding uses three sub-elements:
1. `.finding-header`: tag(s) + `.finding-title` (bold, short label)
2. `.finding-body`: full explanation paragraph
3. `.finding-location`: grayed file/line references wrapped in `<code>`

**Omit** any severity section (heading + findings list) that has no findings. If all three sections are empty, add `<p style="color:#6b7280;font-style:italic;">No issues found.</p>` after the Executive Summary.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Code Review: REPO_NAME</title>
  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; font-size: 14px; line-height: 1.6; color: #1a1a2e; background: #f4f5f7; padding: 32px 16px; }
    .container { max-width: 960px; margin: 0 auto; }
    .header { background: #1a1a2e; color: #fff; border-radius: 10px 10px 0 0; padding: 28px 32px 24px; }
    .header h1 { font-size: 22px; font-weight: 700; letter-spacing: -0.3px; margin-bottom: 6px; }
    .header-meta { font-size: 12px; color: #9ca3af; }
    .header-meta a { color: #60a5fa; text-decoration: none; }
    .header-meta a:hover { text-decoration: underline; }
    .verdict { padding: 18px 32px; display: flex; align-items: flex-start; gap: 14px; }
    .verdict-no      { background: #7f1d1d; border-top: 1px solid #991b1b; color: #fecaca; }
    .verdict-caveats { background: #78350f; border-top: 1px solid #92400e; color: #fde68a; }
    .verdict-yes     { background: #14532d; border-top: 1px solid #166534; color: #bbf7d0; }
    .verdict-icon { font-size: 22px; flex-shrink: 0; margin-top: 1px; }
    .verdict-label { font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 1px; opacity: 0.75; margin-bottom: 4px; }
    .verdict-title { font-size: 17px; font-weight: 700; color: #fff; margin-bottom: 6px; }
    .verdict p { font-size: 13px; line-height: 1.55; opacity: 0.9; }
    .card { background: #fff; padding: 28px 32px; border-left: 1px solid #e5e7eb; border-right: 1px solid #e5e7eb; }
    .card:last-child { border-radius: 0 0 10px 10px; border-bottom: 1px solid #e5e7eb; }
    .stats { display: flex; gap: 12px; flex-wrap: wrap; margin-bottom: 24px; }
    .stat-pill { display: flex; align-items: center; gap: 7px; padding: 6px 13px; border-radius: 20px; font-size: 12px; font-weight: 600; }
    .stat-critical  { background: #fef2f2; color: #991b1b; border: 1px solid #fecaca; }
    .stat-important { background: #fffbeb; color: #92400e; border: 1px solid #fde68a; }
    .stat-minor     { background: #f9fafb; color: #4b5563; border: 1px solid #e5e7eb; }
    .stat-pill .dot { width: 8px; height: 8px; border-radius: 50%; flex-shrink: 0; }
    .dot-critical  { background: #ef4444; }
    .dot-important { background: #f59e0b; }
    .dot-minor     { background: #9ca3af; }
    h2 { font-size: 15px; font-weight: 700; color: #111827; margin: 28px 0 12px; padding-bottom: 6px; border-bottom: 2px solid #e5e7eb; }
    h2:first-child { margin-top: 0; }
    .section-title-row { display: flex; align-items: center; gap: 10px; }
    .count-badge { font-size: 11px; font-weight: 700; padding: 1px 8px; border-radius: 10px; }
    .badge-critical  { background: #fee2e2; color: #b91c1c; }
    .badge-important { background: #fef3c7; color: #b45309; }
    .badge-minor     { background: #f3f4f6; color: #6b7280; }
    p { margin-bottom: 10px; }
    p:last-child { margin-bottom: 0; }
    table { width: 100%; border-collapse: collapse; font-size: 13px; margin-bottom: 4px; }
    thead tr { background: #f9fafb; }
    th { text-align: left; padding: 8px 12px; font-size: 11px; font-weight: 700; text-transform: uppercase; letter-spacing: 0.5px; color: #6b7280; border-bottom: 1px solid #e5e7eb; }
    td { padding: 8px 12px; border-bottom: 1px solid #f3f4f6; vertical-align: top; }
    tr:last-child td { border-bottom: none; }
    .status-run  { color: #16a34a; font-weight: 600; }
    .status-skip { color: #9ca3af; }
    .findings-list { display: flex; flex-direction: column; gap: 10px; }
    .finding { border-radius: 6px; padding: 12px 14px; border-left: 4px solid transparent; font-size: 13px; line-height: 1.55; }
    .finding-critical  { background: #fef2f2; border-color: #ef4444; }
    .finding-important { background: #fffbeb; border-color: #f59e0b; }
    .finding-minor     { background: #f9fafb; border-color: #d1d5db; }
    .finding-header { display: flex; align-items: flex-start; gap: 8px; margin-bottom: 5px; }
    .tag { display: inline-block; font-size: 10px; font-weight: 700; letter-spacing: 0.5px; text-transform: uppercase; padding: 2px 7px; border-radius: 3px; flex-shrink: 0; margin-top: 1px; }
    .tag-security      { background: #fee2e2; color: #991b1b; }
    .tag-quality       { background: #ede9fe; color: #5b21b6; }
    .tag-architecture  { background: #e0f2fe; color: #0369a1; }
    .tag-performance   { background: #fef3c7; color: #92400e; }
    .tag-dependencies  { background: #dcfce7; color: #166534; }
    .tag-documentation { background: #f3e8ff; color: #6b21a8; }
    .tag-multi         { background: #f1f5f9; color: #475569; }
    .finding-title { font-weight: 700; color: #111827; }
    .finding-body { color: #374151; margin-top: 2px; }
    .finding-location { margin-top: 6px; font-size: 11.5px; color: #6b7280; }
    code { font-family: "SF Mono", "Fira Code", Consolas, monospace; background: #f3f4f6; padding: 1px 5px; border-radius: 3px; font-size: 12px; color: #374151; }
    .clean-list { list-style: none; display: flex; flex-direction: column; gap: 5px; }
    .clean-list li { font-size: 13px; color: #374151; padding-left: 20px; position: relative; }
    .clean-list li::before { content: "✓"; color: #16a34a; font-weight: 700; position: absolute; left: 0; }
    hr { border: none; border-top: 1px solid #e5e7eb; margin: 24px 0; }
  </style>
</head>
<body>
<div class="container">

  <div class="header">
    <div class="header-meta" style="margin-bottom:8px;">Code Review Report &nbsp;·&nbsp; YYYY-MM-DD</div>
    <h1>REPO_NAME</h1>
    <div class="header-meta"><a href="REPO_URL" target="_blank">REPO_URL</a></div>
  </div>

  <!-- Verdict: set class to verdict-no / verdict-caveats / verdict-yes; icon &#9888; for No/Caveats, &#10003; for Yes -->
  <div class="verdict verdict-no">
    <div class="verdict-icon">&#9888;</div>
    <div>
      <div class="verdict-label">Verdict</div>
      <div class="verdict-title">Recommend Use: No</div>
      <p>1–2 sentence consensus rationale.</p>
    </div>
  </div>

  <div class="card">

    <!-- Stats pills: fill in actual counts -->
    <div class="stats">
      <div class="stat-pill stat-critical"><span class="dot dot-critical"></span>N Critical</div>
      <div class="stat-pill stat-important"><span class="dot dot-important"></span>N Important</div>
      <div class="stat-pill stat-minor"><span class="dot dot-minor"></span>N Minor</div>
    </div>

    <h2>Scope</h2>
    <table>
      <thead><tr><th>Domain</th><th>Status</th><th>Reason</th></tr></thead>
      <tbody>
        <tr><td>Security</td>      <td class="status-run">&#10003; Run</td><td>Source files present</td></tr>
        <tr><td>Quality</td>       <td class="status-run">&#10003; Run</td><td>Source files present</td></tr>
        <tr><td>Architecture</td>  <td class="status-run">&#10003; Run</td><td>Source files present</td></tr>
        <tr><td>Performance</td>   <td class="status-run">&#10003; Run</td><td>DB/async patterns detected</td></tr>
        <tr><td>Dependencies</td>  <td class="status-run">&#10003; Run</td><td>Package manifest found</td></tr>
        <tr><td>Documentation</td> <td class="status-run">&#10003; Run</td><td>Always included</td></tr>
        <!-- For skipped domains: <tr><td>Domain</td><td class="status-skip">&#10007; Skipped</td><td>reason</td></tr> -->
      </tbody>
    </table>

    <hr>

    <h2>Executive Summary</h2>
    <p>One paragraph: overall health, top concerns, any standout strengths.</p>

    <hr>

    <!-- Omit this section entirely if no Critical findings -->
    <h2><span class="section-title-row">Critical Findings <span class="count-badge badge-critical">N</span></span></h2>
    <div class="findings-list">

      <div class="finding finding-critical">
        <div class="finding-header">
          <!-- Single domain: -->
          <span class="tag tag-security">Security</span>
          <!-- Multi-domain (use ONE combined tag): -->
          <!-- <span class="tag tag-multi">Security / Architecture</span> -->
          <span class="finding-title">Short descriptive title of the finding</span>
        </div>
        <div class="finding-body">Full explanation of the issue, its impact, and how to fix it.</div>
        <div class="finding-location"><code>FileName.java:line</code> &nbsp;·&nbsp; <code>OtherFile.java:line</code></div>
      </div>

    </div>

    <hr>

    <!-- Omit this section entirely if no Important findings -->
    <h2><span class="section-title-row">Important Findings <span class="count-badge badge-important">N</span></span></h2>
    <div class="findings-list">

      <div class="finding finding-important">
        <div class="finding-header">
          <span class="tag tag-quality">Quality</span>
          <span class="finding-title">Short descriptive title</span>
        </div>
        <div class="finding-body">Full explanation.</div>
        <div class="finding-location"><code>FileName.java:line</code></div>
      </div>

    </div>

    <hr>

    <!-- Omit this section entirely if no Minor findings -->
    <h2><span class="section-title-row">Minor Findings <span class="count-badge badge-minor">N</span></span></h2>
    <div class="findings-list">

      <div class="finding finding-minor">
        <div class="finding-header">
          <span class="tag tag-documentation">Documentation</span>
          <span class="finding-title">Short descriptive title</span>
        </div>
        <div class="finding-body">Full explanation.</div>
        <div class="finding-location"><code>FileName.java:line</code></div>
      </div>

    </div>

    <hr>

    <h2>Clean Areas</h2>
    <ul class="clean-list">
      <li>Area with no issues found</li>
    </ul>

  </div>
</div>
</body>
</html>
```

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
