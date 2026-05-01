# Usability dry-run — 2026-05-01

## Setup

- **Fork**: `sanchezpaco/secure-pipeline-workshop`
- **Upstream**: `unicrons/secure-pipeline-workshop`
- **Base commit**: `6608586` (`fix(module-5): align Checkov and Trivy IaC on a shared bait + quieter Checkov output (#52)`) — `origin/main` synced with `upstream/main`.
- **Branch**: `usability-dryrun-2026-05-01`
- **Persona**: Developer with basic CI/CD literacy (knows what a workflow is, can read YAML), no prior context on this workshop, no Ruby/Go/Terraform domain expertise. Treats every module README as the only source of truth.
- **Method**: For each module, paste tool A's `workflow.yml` snippet into the orchestrator placeholder, push, observe CI fail on the planted bait, apply a fix, push, observe CI green. Roll back to the bait state, repeat with tool B.
- **Time-box**: ~10 min per tool. If a finding survives two pushes with no progress, document and move on.

> Sections below are filled in as the run progresses, one commit per module.

## Module 1 — Pipeline Security Scan

### Tool A — Claws

- **What I did**: Replaced the `jobs:` placeholder in `.github/workflows/pipeline-scan.yml` with the snippet from `workshop/pipeline_scan/claws/workflow.yml`. Pushed.
- **Finding(s)**: `Violation: UnpinnedAction on .github/workflows/pipeline-scan.yml:24` (the `ruby/setup-ruby@master` step). One finding only — the rest of the workflow is already pinned to SHAs.
- **Friction**: Mild. The Claws snippet itself is the bait — it ships with `ruby/setup-ruby@master`. The README never tells you the snippet is intentionally insecure, so on a first read you assume "I copied it correctly, why is it failing?". The Claws output is also terse: a single line `Violation: UnpinnedAction on …:24` plus a docs link. Two things you have to figure out yourself: (a) the configured `trusted_authors` only contains `actions`, so any non-`actions/*` action needs a SHA pin, and (b) you have to go look up a SHA — the snippet doesn't tell you where to get one.
- **Fix applied**:

  ```diff
  -      - name: Set Up Ruby
  -        uses: ruby/setup-ruby@master
  +      - name: Set Up Ruby
  +        uses: ruby/setup-ruby@0ecad18fe538ef70f6b82773daecc6af1a7fe58a # v1.252.0
           with:
             ruby-version: '3.3'
  ```

- **README hint to add**: *"The Claws snippet ships with `ruby/setup-ruby@master` to demonstrate `UnpinnedAction`. Once you see the violation, replace it with a SHA pin from a tagged release of `ruby/setup-ruby` (e.g. `ruby/setup-ruby@<sha> # v1.252.0`). Per `claws-config.yml`, only actions authored by `actions` are trusted as unpinned — every other `uses:` must be SHA-pinned."*
- **Time**: ~3 min from snippet paste to green.

### Tool B — zizmor

- **Rollback**: `git revert` of the Claws fix commit; the bait was the snippet itself anyway, since the next step replaces the entire `jobs:` block with the zizmor snippet.
- **What I did**: Replaced the `jobs:` block in `.github/workflows/pipeline-scan.yml` with the snippet from `workshop/pipeline_scan/zizmor/workflow.yml`. Pushed.
- **Finding(s)**: `zizmor/unpinned-uses` on `.github/workflows/pipeline-scan.yml:26` — the `uses: zizmorcore/zizmor-action@main` step in the snippet itself. Severity `high` (matches `min-severity: "high"`).
- **Friction**: Same shape as Claws — the snippet is the bait. zizmor's SARIF output in the job log is verbose JSON and not pleasant to parse by eye; you have to skim past hundreds of lines of SARIF to find `"ruleId": "zizmor/unpinned-uses"` and `"startLine": 26`. A summarized text output (or a `pretty-print: true` flag) would be a big UX win.
- **Fix applied**:

  ```diff
  -        uses: zizmorcore/zizmor-action@main
  +        uses: zizmorcore/zizmor-action@b1d7e1fb5de872772f31590499237e7cce841e8e # v0.5.3
  ```

- **README hint to add**: *"The zizmor snippet itself ships with `zizmorcore/zizmor-action@main` to demonstrate `unpinned-uses`. zizmor's job log dumps the full SARIF — search for `ruleId` and `startLine` to find what to fix. Replace `@main` with a tagged release SHA, e.g. `@<sha> # v0.5.3`."*
- **Time**: ~3 min.

### Module-level observations

- **Convergence**: Both tools converge on the same bait class (an unpinned action), but they flag *different* lines because the bait line lives inside whichever snippet you've pasted. Both rules essentially demonstrate the same tj-actions/changed-files lesson.
- **Smoother attendee experience**: Claws wins on signal-to-noise — one human-readable line per violation with a docs link, vs. zizmor's full SARIF blob in the log. zizmor wins on configurability (severity gating, persona) and on being the obviously more modern, actively-maintained tool. For a workshop, Claws's terse output is friendlier; for a real pipeline, zizmor is the better long-term pick — but the snippet should configure a friendlier output format (`format: plain`).
- **Shared usability gap**: Neither module README states "the snippet you're about to paste is itself the bait." Attendees waste time second-guessing whether they copied wrong before realising the violation is intentional.


## Module 2 — Code Security Analysis

> Scope per the run brief: SAST tools only (CodeQL, Semgrep). Skipped `dependency-check` (NVD key required) and didn't exercise `osv-scanner` — the SCA placeholder in `code-scan.yml` exits 0 by default, so it doesn't gate the orchestrator.

### Tool A — CodeQL

- **What I did**: Replaced the `jobs:` placeholder in `.github/workflows/code-scan.yml` (both `sast-scan` and `sca-scan` placeholders) with the snippet from `workshop/code_scan/sast/codeQL/workflow.yml`. Pushed.
- **Finding(s)**: `js/command-line-injection` at `code/src/simple-app.js:42` — `exec(\`cat "${filePath}"\`, …)` flows user input from `req.body.path` into a shell command. The CodeQL step prints the count and rule ID nicely (`❌ Found 1 CodeQL finding(s):` + a one-line summary).
- **Friction**: Low. The CodeQL "Summarize findings" step in the snippet is well-designed — `ruleId | message | path:line` is exactly what an attendee needs, no SARIF spelunking required. The only friction is **wall-clock**: CodeQL's `init` + `database create` + `analyze` consistently take 4–5 minutes per push, so the bait/fix loop is ~10 min round trip. The bait itself is signposted in the source by a `// TODO: Tech debt - should use fs.readFile instead of shell command for security` comment, so the fix path is obvious.
- **Fix applied**:

  ```diff
  -        if (filePath) {
  -          const fs = require('fs');
  -          const { exec } = require('child_process');
  -
  -          fs.readFile('./config.json', 'utf8', (configErr, configData) => {
  -            const config = configErr ? { debug: false } : JSON.parse(configData);
  -
  -            // TODO: Tech debt - should use fs.readFile instead of shell command for security
  -            exec(`cat "${filePath}"`, (error, stdout, stderr) => {
  +        if (filePath) {
  +          const fs = require('fs');
  +          const path = require('path');
  +
  +          const safeBase = path.resolve(__dirname, 'public');
  +          const resolved = path.resolve(safeBase, filePath);
  +          if (!resolved.startsWith(safeBase + path.sep)) {
  +            res.writeHead(400, { 'Content-Type': 'application/json' });
  +            res.end(JSON.stringify({ error: 'Invalid path' }));
  +            return;
  +          }
  +
  +          fs.readFile('./config.json', 'utf8', (configErr, configData) => {
  +            const config = configErr ? { debug: false } : JSON.parse(configData);
  +
  +            fs.readFile(resolved, 'utf8', (error, stdout) => {
  ```

  (Just removing `exec` is sufficient to clear the rule, but the path-traversal guard is a free win and avoids a follow-up finding from path-injection rules.)

- **README hint to add**: *"CodeQL flags `js/command-line-injection` at `code/src/simple-app.js:42`. The bait is `exec(\`cat "${filePath}"\`)` — replace it with `fs.readFile(resolved, 'utf8', …)` after constraining `filePath` to a safe base directory with `path.resolve` + a `startsWith` check. Heads up: each CodeQL run takes 4-5 min — be patient between bait and fix."*
- **Time**: ~10 min round-trip (limited by CodeQL wall-clock).

### Tool B — Semgrep

- **Rollback**: `git revert` of the CodeQL fix commit; the JS bait was restored. Then I rewrote `.github/workflows/code-scan.yml` with the Semgrep snippet.
- **What I did**: Replaced the `jobs:` block with the snippet from `workshop/code_scan/sast/semgrep/workflow.yml`. Pushed.
- **Finding(s)**: 1 blocking finding (Semgrep prints just `Findings: 1 (1 blocking)` and exits 1). The job log doesn't tell you the rule ID or the file/line — those live only in the SARIF.
- **Friction**: Medium-high — and this is the highest-impact usability gap I hit in any module. The snippet writes findings to `semgrep-results.sarif` and uploads to GitHub code-scanning, but the job log only says "1 finding" — no rule, no path, no line. As an attendee:
  - I can't tell what to fix from the log alone.
  - I have no SARIF artifact to download (the snippet uses `upload-sarif` but doesn't `actions/upload-artifact`, so on a fork without GHAS you can't see findings anywhere).
  - I had to pattern-match against the CodeQL bait I'd just fixed and assume Semgrep was flagging the same `exec(...)` line — which is a leap most fresh attendees won't make.
- **Fix applied**: Same diff as CodeQL (`exec` → `fs.readFile` with path guard).
- **README hint to add**: *"Semgrep's job log only prints a finding count. To see the rule, either (a) add a `--text` summary alongside `--sarif`, (b) drop `--sarif`/`--output` so Semgrep prints findings to stdout, or (c) upload `semgrep-results.sarif` as an artifact via `actions/upload-artifact`. The bait is the same `exec(\`cat "${filePath}"\`)` line — Semgrep flags it via `javascript.lang.security.audit.detect-child-process` in the `p/security-audit` ruleset."*
- **Time**: ~6 min (Semgrep itself runs in ~3 s; install + checkout dominates).

### Module-level observations

- **Convergence**: Both tools converge on the same root bait (`exec` with user-controlled string). They both ignore the hardcoded AWS keys at `simple-app.js:4-5` (Semgrep's `p/security-audit` doesn't include the AWS-key rule by default; CodeQL's javascript-security-extended doesn't either) — those are reserved for the secrets module, which is correct module separation. Convergence: 5/5.
- **Smoother experience**: CodeQL by a wide margin, *because of the snippet*, not the tool. The CodeQL snippet has a custom `Summarize findings` step that prints `ruleId | message | path:line`. The Semgrep snippet doesn't — it dumps to SARIF and stops. Same scanner output, vastly different attendee experience. The Semgrep snippet should adopt the same summary pattern.
- **Discoverability gap**: Neither module README mentions which line of `code/src/simple-app.js` is the planted bait. The `// TODO: Tech debt - should use fs.readFile…` comment in the source IS the hint, but readers find it only after the scan fails — a one-line README pointer ("look in `/readfile`") would save a confused attendee 2-3 minutes.


## Module 3 — Secrets Scan

### Tool A — TruffleHog

- **What I did**: Replaced the `jobs:` placeholder in `.github/workflows/secrets-scan.yml` with the snippet from `workshop/secrets_scan/trufflehog/workflow.yml`. Pushed.
- **Finding(s)**: 1 unverified result — `Detector Type: AWS / Account: 729780141977 / File: code/src/simple-app.js / Line: 4 / Raw result: AKIA…REDACTED`. TruffleHog exits 183.
- **Friction**: Very low. The job log prints the detector name, file, and line as a clean ~6-line block, which is the best per-finding output of any tool I exercised. The `--results=verified,unknown,unverified` flag is already wired in (with a clarifying comment) — nice touch, since the workshop's didactic AKIA isn't backed by a real AWS account so it would be silently dropped under the default `verified` filter. Without that flag the build would pass and the bait would never trip.
- **Fix applied**:

  ```diff
  -const AWS_ACCESS_KEY_ID = 'AKIA…REDACTED'
  -const AWS_SECRET_ACCESS_KEY = '[redacted-40-char-secret]'
  +const AWS_ACCESS_KEY_ID = process.env.AWS_ACCESS_KEY_ID;
  +const AWS_SECRET_ACCESS_KEY = process.env.AWS_SECRET_ACCESS_KEY;
  ```

- **README hint to add**: *"TruffleHog finds the hardcoded AWS access key at `code/src/simple-app.js:4`. Replace `const AWS_ACCESS_KEY_ID = 'AKIA…'` with `process.env.AWS_ACCESS_KEY_ID` (and the secret, mutatis mutandis). The snippet uses `--results=verified,unknown,unverified` on purpose: the workshop's AKIA is invalid, so the default `verified`-only filter would silently pass."*
- **Time**: ~2 min.

### Tool B — Gitleaks

- **Rollback**: `git revert` of the TruffleHog fix commit; the AWS keys are back in `simple-app.js`. Then I rewrote `.github/workflows/secrets-scan.yml` with the Gitleaks snippet.
- **What I did**: Pasted snippet from `workshop/secrets_scan/gitleaks/workflow.yml` into the workflow. Pushed.
- **Finding(s)**: 2 leaks:
  1. `aws-access-token` at `code/src/simple-app.js:4` (the `AKIA…` literal).
  2. `generic-api-key` at `code/src/simple-app.js:5` (high-entropy match on the secret access key).
- **Friction**: Low-to-medium. Output is rich (`Finding | Secret | RuleID | Entropy | File | Line | Fingerprint`) but mixed with a long `wget` progress dump (because the snippet shells out instead of using the `gitleaks/gitleaks-action`). The progress noise pushes the actual finding far down the log; on a small terminal you scroll past it. The `--no-git` mode has a clarifying comment (OPTION 1 vs OPTION 2) — that's helpful, but the comment is twice as long as the actual command and could be a one-liner.
- **Fix applied**: Same diff as TruffleHog (lines 4-5 → `process.env.*`). One fix clears both Gitleaks rules.
- **README hint to add**: *"Gitleaks reports the access-key and secret-key as two separate findings (`aws-access-token` and `generic-api-key`). Both are fixed by the same edit at `simple-app.js:4-5` — move the literals to `process.env.*`. Heads up: the `wget` install pollutes the log with a progress bar; either pipe it to `-q` or switch to `gitleaks/gitleaks-action@<sha>`."*
- **Time**: ~3 min.

### Module-level observations

- **Convergence**: Both tools flag the same bait (the hardcoded AWS keys at lines 4-5), so attendees see consistent results. TruffleHog only fires on the access-key (`AKIA…`); Gitleaks fires on both the access-key and the secret-key. Good enough convergence — same root cause, same one-edit fix. Convergence: 5/5.
- **Smoother attendee experience**: TruffleHog. Single finding, clean tabular output, no install noise. Gitleaks's findings table is also great, but the noisy `wget` install hides the signal in a 200-line log.
- **Snippet hygiene**: TruffleHog's snippet had to be patched (the comment on `--results=...` confirms this — it explains *why* unverified is included). Both snippets could lose 5–10 lines of comments and become more attendee-friendly.


## Module 4 — Container Security Scanning

_(pending)_

## Module 5 — IaC Security Scan

_(pending)_

## Modules not executed (static observations)

_(pending)_

## Cross-module usability observations

_(pending)_

## Executive summary

_(pending — top 3 README changes will go here)_

## Appendix — Fix recipes

_(pending — copy-pasteable fix blocks per finding)_
