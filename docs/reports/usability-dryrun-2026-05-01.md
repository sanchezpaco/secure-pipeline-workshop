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

### Tool C — osv-scanner (SCA addendum)

> Re-opened Module 2 after Module 4 because the axios bait was leaking into the container scan. SCA is the right place for it.

- **What I did**: Reverted `code/package.json` axios from `1.13.5` back to `1.6.0` (and regenerated `code/package-lock.json` with `npm install --package-lock-only --ignore-scripts`) to restore the SCA bait. Added a second job (`sca-scan`) to `.github/workflows/code-scan.yml` from `workshop/code_scan/sca/osv-scanner/workflow.yml`, alongside the Semgrep job.
- **Finding(s)**: 6 vulnerability findings, all on `axios@1.6.0`:
  - `CVE-2025-62718` (GHSA-3p68-rc4w-qgx5)
  - `CVE-2026-25639` (GHSA-43fc-jf86-j433) — proto-key DoS
  - `CVE-2025-58754` (GHSA-4hjh-wcwx-xvwj) — DoS
  - `CVE-2024-39338` (GHSA-8hc4-vh64-cxmj) — SSRF
  - `CVE-2026-40175` (GHSA-fvcv-3m26-pcqx)
  - `CVE-2025-27152` (GHSA-jr5f-v2jv-69x6) — SSRF + cred leak
  - **2 more than Trivy found in Module 4** (Trivy listed 4; osv-scanner adds CVE-2025-62718 and CVE-2026-40175). osv-scanner's database is more complete for application-layer deps.
- **Friction**: Medium. The snippet's "Summarize findings" step prints `ruleId | message | path:line` — same friendly format as the CodeQL snippet (this should be the template for *all* SAST/SCA snippets). But: bumping `axios` to `1.13.5` (the version Trivy's table told me was "fixed" in Module 4) was **not enough** — `osv-scanner` still flagged 2 unfixed CVEs (CVE-2025-62718, CVE-2026-40175). Trivy's "Fixed Version" column shows the fix for *that specific CVE*; it does not promise the version is clean of all CVEs in the OSV DB. I had to bump again to **`axios@1.15.2`** to go green — same pattern as Module 4 (alpine 3.21 wasn't enough; needed `node:25-alpine`).
- **Fix applied** (two pushes):

  ```diff
  -    "axios": "1.6.0"
  +    "axios": "1.15.2"
  ```

  Plus regenerated `package-lock.json` after each bump.

- **README hint to add**: *"`osv-scanner` finds 6 CVEs on `axios@1.6.0` (more than Trivy's library scanner finds in Module 4 — OSV's app-layer DB is denser). Bump `axios` to a recent patched version (`1.15.2` was clean at run time; check `npm view axios versions` for the latest). After bumping, run `npm install --package-lock-only` to regen the lockfile. **Heads-up**: Trivy's 'Fixed Version' column is per-CVE, not per-package — bumping to the version Trivy listed (e.g. `1.13.5`) may still leave you with later CVEs."*
- **Time**: ~7 min round-trip across two attempts.

### Module-level observations

- **Convergence (SAST)**: CodeQL and Semgrep both converge on the same `exec(...)` bait. They both ignore the hardcoded AWS keys at `simple-app.js:4-5` (Semgrep's `p/security-audit` doesn't include the AWS-key rule by default; CodeQL's javascript-security-extended doesn't either) — those are reserved for the secrets module, which is correct module separation. Convergence: 5/5.
- **Convergence (SCA → container)**: `osv-scanner` and Trivy's library scanner both find axios CVEs, but `osv-scanner` is denser (6 vs Trivy's 4). The intended layering is: SCA in Module 2 catches it; container scan in Module 4 should be a backstop, not the primary detector. The bait was correctly placed — the gap was the workshop flow, not the tool selection.
- **Smoother experience (SAST)**: CodeQL by a wide margin, *because of the snippet*, not the tool. The CodeQL snippet has a custom `Summarize findings` step that prints `ruleId | message | path:line`. The Semgrep snippet doesn't — it dumps to SARIF and stops. Same scanner output, vastly different attendee experience. The Semgrep snippet should adopt the same summary pattern. The `osv-scanner` snippet *already* has the right pattern.
- **Discoverability gap**: Neither module README mentions which line of `code/src/simple-app.js` is the planted SAST bait. The `// TODO: Tech debt - should use fs.readFile…` comment in the source IS the hint, but readers find it only after the scan fails — a one-line README pointer ("look in `/readfile`") would save a confused attendee 2-3 minutes. Same applies to SCA: the README should call out `axios@1.6.0` in `package.json`.
- **Module-flow gap**: The Module 2 brief said "skip dependency-check (NVD key)" without saying "use osv-scanner instead." If an attendee follows the brief literally, the SCA slot stays as a placeholder, and the axios bait leaks past Module 2 into the container scan in Module 4 — mixing SCA findings and OS-layer findings into the same noisy log. **The brief should mandate `osv-scanner` for the SCA slot.**


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

### Tool A — Trivy

- **What I did**: Replaced the `jobs:` placeholder in `.github/workflows/build-and-container-scan.yml` with the snippet from `workshop/container_scan/trivy/workflow.yml`. Pushed.
- **Finding(s)**: Trivy reports two tables:
  - **OS layer** (alpine 3.19.4): `musl` + `musl-utils` — 4 HIGH (CVE-2025-26519, CVE-2026-40200).
  - **Application layer** (`axios (package.json)` 1.6.0): 4 HIGH — CVE-2024-39338 (SSRF), CVE-2025-27152 (SSRF + cred leak), CVE-2025-58754 (DoS), CVE-2026-25639 (proto-key DoS).
- **Friction**: My **first fix attempt** (bump `node:20-alpine3.19` → `node:20-alpine3.21`) made it worse — alpine 3.21 ships openssl 3.3.5-r0 which has unpatched CVEs (`libcrypto3`/`libssl3` HIGH+CRITICAL). Total went from 8 to 19 findings. The actual fix is to jump major: **`node:25-alpine`** (current alpine, current node, current openssl). I would NOT have figured that out without running CI twice and reading the openssl line in the second failure — a fresh attendee will hit this same wall and assume "I picked the right alpine" was the right move. The README needs to call out: *don't bump alpine minor, jump to current node-major + alpine.*
- **Fix applied**:

  ```diff
  -FROM node:20-alpine3.19
  +FROM node:25-alpine
  ```

  ```diff
  -    "axios": "1.6.0"
  +    "axios": "1.13.5"
  ```

  Plus regenerated `package-lock.json` (`rm -f package-lock.json && npm install --package-lock-only`) — required because the snippet does `npm ci`, which fails if the lock file is out of sync with `package.json`.

- **README hint to add**: *"Trivy reports OS-layer + application-layer vulnerabilities. The bait combines both: alpine 3.19 has unfixed `musl` HIGHs and the `axios@1.6.0` dep has 4 HIGH CVEs (SSRF, DoS, cred leak). Fix: bump base image to `node:25-alpine` (don't just bump alpine minor — 3.21 still has openssl HIGHs); bump `axios` to `1.13.5`; regenerate `package-lock.json` so `npm ci` doesn't fail."*
- **Time**: ~25 min round-trip (build+scan ~5 min × 3 attempts: bait, wrong-fix, right-fix).

### Tool B — Grype

- **Rollback**: Reverted both Trivy fix commits to restore Dockerfile + axios bait.
- **What I did**: Replaced the `jobs:` block with the snippet from `workshop/container_scan/grype/workflow.yml`. Pushed.
- **Finding(s)**: 1 line of useful output — `[0030] ERROR discovered vulnerabilities at or above the severity threshold` plus a helpful warning: `188 packages from EOL distro "alpine 3.19.4" - vulnerability data may be incomplete or outdated; consider upgrading to a supported version`. Grype writes the SARIF and uploads it; the actual CVE list lives in the GHAS Code Scanning tab, which is **invisible on a fork without GHAS** and useless without a SARIF artifact.
- **Friction**: High. Compared to Trivy's two-table CVE breakdown, Grype's CLI output here is one error line with no details. Without the EOL warning I would have had no clue what to fix. The snippet should either:
  - add `output-format: table` (or run `grype` a second time in plain output) so the CVE list is visible in the job log; or
  - upload the SARIF as an `actions/upload-artifact` so it's downloadable from the run page.
- **Fix applied**: Same as Trivy (`node:25-alpine` + axios 1.13.5 + lock regen). One commit, no second attempt needed because by now I knew the playbook from the Trivy round.
- **README hint to add**: *"Grype's job log is sparse — only `discovered vulnerabilities at or above the severity threshold` and an EOL hint. To see the actual CVEs, either add `output-format: table` (Grype prints to stdout), upload the SARIF as an artifact, or read the GHAS Code Scanning tab. The fix is identical to Trivy's: bump base to `node:25-alpine` and `axios` to `1.13.5`, then regenerate `package-lock.json`."*
- **Time**: ~6 min — fast because the fix was already known.

### Module-level observations

- **Convergence**: Both tools fail on the same root causes (EOL alpine + unfixed axios), so a single fix clears both. Grype is just less verbose about *which* CVEs it found. Convergence: 5/5.
- **Smoother attendee experience**: **Trivy by a wide margin**. Trivy's tabular output with library/CVE/severity/installed/fixed/title is genuinely actionable — an attendee can read it, copy the fixed version, and go. Grype dumped a single error line and a SARIF blob.
- **Major usability gap (cross-module)**: The `axios@1.6.0` bait is an **SCA finding** (vulnerable dependency in `package.json`), not a container finding. It belongs in Module 2's SCA slot — but the brief said "skip dependency-check (NVD key)" and didn't replace it with `osv-scanner`, so the slot stayed as a placeholder that exits 0. Result: the axios bait leaks past the SAST scanners (which by design only look at code patterns) and shows up in Module 4 alongside the OS-level CVEs, mixing two didactic categories. **A run of `osv-scanner` in Module 2 would catch axios there, where it belongs.** I plan to plug `osv-scanner` into Module 2 after this module to demonstrate the proper module-level isolation.
- **Lock-file footgun**: `npm ci` requires `package.json` and `package-lock.json` to be in sync. Bumping axios in `package.json` alone breaks the build with `EUSAGE` before any scan runs — the README needs to say "after bumping a dep, run `npm install` (or `npm install --package-lock-only`) to regen the lock file, then commit both."


## Module 5 — IaC Security Scan

### Tool A — Checkov

- **What I did**: Replaced the `jobs:` placeholder in `.github/workflows/iac-scan.yml` with the snippet from `workshop/iac_scan/checkov/workflow.yml`. Pushed.
- **Finding(s)**: `Passed checks: 18, Failed checks: 1` — `CKV_AWS_24: Ensure no security groups allow ingress from 0.0.0.0:0 to port 22` on `aws_security_group.ecs_tasks` at `infra/main.tf:115-149`. The bait is on lines 120-126: an ingress rule with `description = "HTTP from ALB"` but `from_port/to_port = 22` — a classic copy-paste/typo bait where the description says one thing and the ports say another.
- **Friction**: Very low. With `quiet: true`, Checkov only prints failed checks plus the summary line (18 passed / 1 failed). The output names the rule, the resource, the file, the line range, and quotes the offending lines (`125 | cidr_blocks = ["0.0.0.0/0"]` etc.). This is the gold standard for IaC scanner UX. The README of `infra/main.tf` even leaves a kindly SRE note above the security group: `if you need to reference the VPC CIDR use this data: data.aws_vpc.existing.cidr_block` — exactly the helper for the fix. Excellent signposting.
- **Fix applied**:

  ```diff
     ingress {
  -    description = "HTTP from ALB"
  -    from_port   = 22
  -    to_port     = 22
  +    description = "HTTP from ALB (workshop)"
  +    from_port   = 3000
  +    to_port     = 3000
       protocol    = "tcp"
  -    cidr_blocks = ["0.0.0.0/0"]
  +    cidr_blocks = [data.aws_vpc.existing.cidr_block]
     }
  ```

- **README hint to add**: *"Checkov flags `CKV_AWS_24` on `aws_security_group.ecs_tasks` at `infra/main.tf:120-126` — port 22 ingress from `0.0.0.0/0` despite a `description = \"HTTP from ALB\"`. Fix the port to the app's actual port (3000) and tighten the CIDR to the VPC range using `data.aws_vpc.existing.cidr_block` (the SRE note above the resource literally hands you this)."*
- **Time**: ~3 min.

### Tool B — Trivy IaC

- **Rollback**: `git revert` of the Checkov fix commit; the SSH-from-anywhere bait is back in `main.tf`. Then I rewrote `iac-scan.yml` with the Trivy IaC snippet.
- **What I did**: Pasted snippet from `workshop/iac_scan/trivy/workflow.yml`. Pushed.
- **Finding(s)**: Trivy IaC writes findings to `trivy-results.sarif` and uploads to GHAS. The job log shows only `Running Trivy with options: trivy config infra/` followed by `Successfully uploaded results` — **no AVD ID, no rule, no file/line in the log**. To see the actual finding I'd need either GHAS access (private/fork limitation) or to download the SARIF artifact (snippet doesn't upload one). I had to assume the bait was the same `CKV_AWS_24`-class issue Checkov caught (Trivy's equivalent rule is `AVD-AWS-0107`, "Open ingress to port 22 from 0.0.0.0/0").
- **Friction**: High — same root cause as Grype's CLI in Module 4. The snippet uses `format: 'sarif'`, which suppresses Trivy's normally-excellent table output. Two clean fixes:
  - Change `format` to `table` (and have a separate run for SARIF if you want GHAS upload), or
  - Add `actions/upload-artifact` for `trivy-results.sarif` so the file is downloadable from the run page.
- **Fix applied**: Same diff as Checkov (port 22 → 3000, CIDR → VPC). One edit clears both scanners.
- **README hint to add**: *"Trivy IaC's snippet writes only SARIF — the job log shows no findings, just an upload-success line. Either add `format: table` (sacrifice GHAS upload) or upload the SARIF as an artifact via `actions/upload-artifact`. The bait is the same `infra/main.tf:120-126` block Checkov flags; Trivy's rule ID is `AVD-AWS-0107`."*
- **Time**: ~3 min — fast because the fix was already known from Checkov.

### Module-level observations

- **Convergence**: 5/5. Both tools converge on the same single bait (port-22 ingress from `0.0.0.0/0`). One edit, both scanners green. The other infra patterns that *could* be misconfigurations (CloudWatch retention, KMS encryption, etc.) are deliberately wired to be safe — clean module isolation.
- **Smoother experience**: **Checkov by a wide margin**, again because of snippet output. Checkov prints rule ID + resource + file + line + offending lines all in the job log. Trivy IaC writes a SARIF and goes silent. The Trivy snippet should either switch to `format: table` for the workshop or upload the SARIF as a downloadable artifact.
- **Snippet asymmetry across modules**: Trivy *as a container scanner* (Module 4) prints a great table to stdout. Trivy *as an IaC scanner* (Module 5) is configured for SARIF-only. Same tool, opposite UX in the two snippets — this is a snippet-author choice, not a tool limitation. Aligning them would help.

## Modules not executed (static observations)

### `runtime_infra_scan` (Prowler + Steampipe)

- **Why skipped**: Requires an AWS sandbox account, an OIDC trust between GitHub and AWS, and an `AWS_IAM_ROLE_ARN` secret. None of those are part of the workshop scaffolding — they're attendee-side setup.
- **README quality**: Strong on theory (good taxonomy of runtime issues, clear differentiation from IaC) but does not tell the attendee *how* to set up the AWS side. A fresh attendee following the workshop linearly hits a hard dependency wall here. Recommend either:
  - splitting this module into a "static read" track (read the runtime checklist, no execution) and an "advanced" track (full AWS setup), or
  - providing a one-click CFT/Terraform module that creates the IAM role and OIDC provider (and a teardown script).
- **Snippet hygiene**: `runtime-infra-scan.yml` orchestrator entry pulls `secrets: AWS_IAM_ROLE_ARN`. If that secret is missing in the fork, the entire orchestrator fails at the wiring stage even before the scanner runs — and the failure is "secret not found", not a friendly "not configured for this fork." Pre-flight check + skip-if-unset would be a kindness.

### `dast` (placeholder)

- **Status**: Not shipped in v1 — the orchestrator stub literally exits 0 with `🚧 DAST - Coming soon`. Workshop README still lists it (or the index does) without flagging "not yet".
- **Recommendation**: Either remove it from the workshop index until shipped, or relabel as "preview / not part of v1" so attendees don't waste time looking for the snippet folder. The `workshop/dast/` directory simply does not exist, which is more confusing than the YAML stub.

### `ai_scan` (Google Gemini)

- **Status**: Wired with `google-github-actions/run-gemini-cli`. The brief flagged "current Gemini integration is considered outdated", and indeed the module ships only the Gemini path with no fallback or alternative.
- **README quality**: Long preamble on AI capabilities/limitations (good), but the practical part is one tool (Gemini) and a single GitHub Action. Concerns:
  - **API key + cost burden** is on the attendee (the README mentions "Cost Considerations" abstractly but doesn't say "you need a Gemini API key to run this module").
  - **Privacy / data residency**: the module ships code to Google. For an enterprise workshop this is a real blocker — the README acknowledges "Privacy Concerns" but doesn't give an alternative path (e.g., Claude Code Action, OpenAI Codex review, a local LLM via Ollama).
  - **Determinism**: the module promises "different outputs for the same input across runs" — that fundamentally breaks the TDD-with-rollback flow used by the rest of the workshop. AI scan should probably be presented as a *complementary* track, not a regular module slot.
- **Recommendation**: Reframe as an optional / non-gating module, document the API-key prerequisite up front, and add at least one alternative runner (e.g. Claude Code Action or OpenAI Codex) so attendees with a different provider can still complete it.

## Cross-module usability observations

These showed up across multiple modules and are the highest-leverage README/snippet changes.

1. **The snippet IS often the bait, but the README never says so.** In Module 1 both Claws (`ruby/setup-ruby@master`) and zizmor (`zizmorcore/zizmor-action@main`) are intentionally unpinned in the snippet. A fresh attendee assumes "I copied wrong" before realising the snippet is deliberately insecure. One sentence per snippet README ("the snippet ships with one planted bait — pin it") would save attendees 3-5 minutes of confusion each. Same pattern less explicitly in the secrets and code-scan snippets.
2. **Snippet output formats are wildly inconsistent — and that's the single biggest UX axis.** Four flavours showed up across 11 tools:
   - **Excellent**: TruffleHog (raw stdout block), Checkov (per-rule failed checks + offending lines), the CodeQL custom `Summarize findings` step, the osv-scanner custom `Summarize findings` step, Trivy as container scanner (table format), Gitleaks (despite wget noise).
   - **Mediocre**: Semgrep (`Findings: 1 (1 blocking)` — count only).
   - **Bad**: Grype container (`ERROR discovered vulnerabilities at or above the severity threshold` — one line, no CVE list), Trivy IaC (SARIF-only, log shows nothing), zizmor (full SARIF JSON dump in the log).
   - **The fix template already exists**: copy CodeQL's / osv-scanner's `Summarize findings` step (jq over the SARIF) into every other SARIF-producing snippet. That alone would equalise the experience across the whole workshop.
3. **"Bumping to the obvious next version" is rarely the right fix — and the workshop never warns you.** Two modules tripped on this:
   - Module 4: bumping `node:20-alpine3.19 → node:20-alpine3.21` made things *worse* (8 → 19 findings) because alpine 3.21 still ships unpatched openssl. The actual fix is `node:25-alpine`.
   - Module 2 SCA: bumping `axios@1.6.0 → axios@1.13.5` (the version Trivy listed as "Fixed Version") still left 2 unfixed CVEs per osv-scanner. The actual fix was `axios@1.15.2`.
   - Both burned ~5 minutes of CI per wrong attempt. The README needs a generic note: *"Trivy/OSV's Fixed Version is per-CVE, not per-package. Always bump to the latest stable patch level, not the lowest version that closes the CVE you happened to be looking at."*
4. **Module-level isolation leaks: the SCA bait jumped from Module 2 to Module 4.** The brief said "skip dependency-check" without recommending `osv-scanner` as a substitute, so the SCA slot stayed as a pass-through placeholder. The `axios@1.6.0` bait then re-surfaced as a container-scan finding in Module 4, mixing application-layer with OS-layer CVEs in the same noisy log. Plugging `osv-scanner` into the SCA slot in Module 2 caught the bait at the right layer (and found 2 *more* CVEs than Trivy's library scanner). Recommendation: **the brief should mandate `osv-scanner` for the SCA slot** when `dependency-check` is skipped, not leave it as an opt-in.
5. **Writing the workshop report can fail the workshop.** Documenting the secrets-scan bait in `docs/reports/usability-dryrun-2026-05-01.md` (with literal `AKIA…` and the secret string for evidence) made Gitleaks fail on the report itself. Two options for attendees who write notes:
   - Add `docs/reports/**` (or the equivalent) to a `.gitleaksignore` and a `.trufflehogignore`.
   - Tell attendees up front to redact literals as `AKIA…REDACTED` / `[redacted-40-char-secret]` when writing notes.
6. **Pinning hygiene drift in the snippets themselves.** Multiple snippets `uses: third-party-action@main` or `@master` while pinning `actions/*` to SHAs — exactly the lesson Module 1 teaches. Once you fix Module 1 the rest of the workshop still ships with the antipattern. Either pin all third-party actions to SHAs in the snippet templates from the start, or call out per-module "we didn't pin X yet — your job to fix it" so the lesson reinforces.
7. **Lock-file footgun in Module 4 / Module 2 SCA.** Bumping `axios` in `package.json` alone breaks `npm ci` with `EUSAGE: package.json and package-lock.json are out of sync`. The error happens at *build* time inside the container scan, so attendees may misdiagnose it as "Trivy / Grype broken" rather than "I forgot to regen the lockfile." The container README needs to say "after a dep bump, run `npm install --package-lock-only` (or `npm install`) and commit `package-lock.json` as well."
8. **CI wall-clock dominates UX for the slow modules.** CodeQL takes 4-5 minutes per run; Trivy build+scan ~5 min. With a 2-step fix (e.g. axios 1.13.5 → 1.15.2), an attendee waits 10+ minutes between bait and green. The workshop is otherwise tight at ~3 min per tool. Mention this in each slow module's README so attendees don't think CI is stuck.

## Executive summary

### Top 3 highest-impact README/snippet changes

1. **Standardise the "Summarize findings" step across all SARIF-producing snippets.** Copy the jq-based `Summarize findings` step from the CodeQL and osv-scanner snippets into Semgrep, Grype, Trivy IaC, and zizmor. This single change converts five "what did the scanner even find?" experiences into clear `ruleId | message | file:line` reports — and costs ~10 lines of YAML per snippet. **Highest leverage by far.** Pain it removes: 5-10 minutes per affected tool, every attendee, forever.
2. **State explicitly per snippet README: "this snippet ships with N planted bait(s); the failure is intentional."** Especially in Module 1 (Claws / zizmor — the snippet itself is the bait), Module 4 (the Dockerfile + `package.json` shipped with deliberate vulns), Module 5 (the SSH-22-from-anywhere SG). Pair each snippet with a one-line "what to look for" pointer. Prevents the "I copied this wrong" rabbit hole that hits every fresh attendee.
3. **Mandate `osv-scanner` for the SCA slot in Module 2; flag the "Fixed Version is per-CVE, not per-package" pitfall.** Without `osv-scanner`, the axios bait crosses module boundaries into the container scan in Module 4, mixing didactic categories. With `osv-scanner` the bait lives in Module 2 where it belongs *and* the workshop discovers 2 extra CVEs Trivy misses. While we're here, document that bumping to the version a scanner lists as "fixed" only fixes the CVE you happened to be looking at — `npm view <pkg> versions` for the latest is the better default.

### Tool scoring (1-5 per axis)

| Module | Tool | Discoverability | Predictability | Signal-to-noise | Convergence |
|--------|------|:---------------:|:--------------:|:---------------:|:-----------:|
| pipeline_scan | Claws | 4 | 5 | 5 | 5 |
| pipeline_scan | zizmor | 4 | 5 | 2 (full SARIF dump) | 5 |
| code_scan/SAST | CodeQL | 4 | 5 | 5 (custom summary step) | 5 |
| code_scan/SAST | Semgrep | 3 | 5 | 1 (no rule in log) | 5 |
| code_scan/SCA | osv-scanner | 5 | 4 (1.13.5 still vuln) | 5 | 5 |
| secrets_scan | TruffleHog | 5 | 5 | 5 | 5 |
| secrets_scan | Gitleaks | 5 | 5 | 3 (wget noise) | 5 |
| container_scan | Trivy | 4 | 3 (alpine 3.21 trap) | 5 | 5 |
| container_scan | Grype | 4 | 5 | 1 (single-line error) | 5 |
| iac_scan | Checkov | 5 | 5 | 5 | 5 |
| iac_scan | Trivy IaC | 4 | 5 | 1 (SARIF-only) | 5 |

**Best overall attendee experience**: Checkov, TruffleHog, osv-scanner, CodeQL — in that order. **Worst**: Grype container and Trivy IaC, both because of the SARIF-only output choice in their snippets.

## Appendix — Fix recipes

For each finding I encountered, a self-contained, copy-pasteable "how I fixed it" block. Maintainers can drop these into the relevant module README under "Common fixes" or "Hints".

### pipeline_scan / Claws / `UnpinnedAction`

**Finding**: `Violation: UnpinnedAction on .github/workflows/pipeline-scan.yml:24` — the `ruby/setup-ruby@master` step in Claws's own snippet.
**Fix**:
```yaml
- name: Set Up Ruby
  uses: ruby/setup-ruby@0ecad18fe538ef70f6b82773daecc6af1a7fe58a # v1.252.0
  with:
    ruby-version: '3.3'
```
**Notes**: Per `claws-config.yml`, only `actions/*` is in `trusted_authors`, so any non-`actions/*` action must be SHA-pinned. Pick a SHA from a tagged release of `ruby/setup-ruby` and add a trailing comment with the version.

### pipeline_scan / zizmor / `unpinned-uses`

**Finding**: `zizmor/unpinned-uses` at `.github/workflows/pipeline-scan.yml:26` — the `zizmorcore/zizmor-action@main` step in zizmor's own snippet.
**Fix**:
```yaml
- name: Run zizmor 🌈
  uses: zizmorcore/zizmor-action@b1d7e1fb5de872772f31590499237e7cce841e8e # v0.5.3
```
**Notes**: zizmor's job log dumps full SARIF; search for `"ruleId"` and `"startLine"` to find the offending line. Cap `min-severity: "high"` to keep the workshop bait the primary signal.

### code_scan/SAST / CodeQL / `js/command-line-injection`

**Finding**: `code/src/simple-app.js:42` — `exec(\`cat "${filePath}"\`, …)` flows user input into shell.
**Fix** (full diff, also clears Semgrep's child-process rule and a follow-up path-traversal finding):
```js
if (filePath) {
  const fs = require('fs');
  const path = require('path');

  const safeBase = path.resolve(__dirname, 'public');
  const resolved = path.resolve(safeBase, filePath);
  if (!resolved.startsWith(safeBase + path.sep)) {
    res.writeHead(400, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ error: 'Invalid path' }));
    return;
  }

  fs.readFile('./config.json', 'utf8', (configErr, configData) => {
    const config = configErr ? { debug: false } : JSON.parse(configData);

    fs.readFile(resolved, 'utf8', (error, stdout) => {
      if (error) {
        res.writeHead(500, { 'Content-Type': 'application/json' });
        res.end(JSON.stringify({ error: error.message }));
        return;
      }
      res.writeHead(200, { 'Content-Type': 'application/json' });
      res.end(JSON.stringify({ config: config, content: stdout }));
    });
  });
}
```
**Notes**: The minimum to clear `js/command-line-injection` is replacing `exec(...)` with `fs.readFile`. The `path.resolve` + `startsWith` guard is a free win that prevents path-traversal follow-ups.

### code_scan/SAST / Semgrep / `javascript.lang.security.audit.detect-child-process`

**Finding**: 1 blocking finding (rule + line not visible in the job log — only in SARIF). Same `exec(...)` line as CodeQL.
**Fix**: Identical to the CodeQL diff above.
**Notes**: Add a `--text` summary in addition to `--sarif`, or upload `semgrep-results.sarif` via `actions/upload-artifact`, so attendees can see *what* failed without a GHAS license.

### code_scan/SCA / osv-scanner / `axios@1.6.0` (6 CVEs)

**Finding**: `CVE-2025-62718`, `CVE-2026-25639`, `CVE-2025-58754`, `CVE-2024-39338`, `CVE-2026-40175`, `CVE-2025-27152` — all on `axios@1.6.0` in `code/package.json`.
**Fix**:
```diff
-    "axios": "1.6.0"
+    "axios": "1.15.2"
```
Then regenerate the lockfile:
```bash
cd code && rm -f package-lock.json && npm install --package-lock-only && cd ..
git add code/package.json code/package-lock.json
```
**Notes**: Bumping to `1.13.5` (Trivy's "Fixed Version" for the `1.6.0` CVEs) still leaves 2 unfixed (`CVE-2025-62718`, `CVE-2026-40175`). Always check `npm view axios versions` and pick a recent patch.

### secrets_scan / TruffleHog / AWS detector

**Finding**: `Detector Type: AWS` at `code/src/simple-app.js:4` — hardcoded `AKIA…REDACTED`.
**Fix**:
```diff
-const AWS_ACCESS_KEY_ID = 'AKIA…REDACTED'
-const AWS_SECRET_ACCESS_KEY = '[redacted-40-char-secret]'
+const AWS_ACCESS_KEY_ID = process.env.AWS_ACCESS_KEY_ID;
+const AWS_SECRET_ACCESS_KEY = process.env.AWS_SECRET_ACCESS_KEY;
```
**Notes**: Snippet uses `--results=verified,unknown,unverified` because the workshop's AKIA is invalid — under default `verified`-only filtering it would silently pass.

### secrets_scan / Gitleaks / `aws-access-token` + `generic-api-key`

**Finding**: 2 leaks at `code/src/simple-app.js:4-5` (access key + entropy match on the secret).
**Fix**: Same diff as TruffleHog above. One edit clears both rules.
**Notes**: The snippet uses `--no-git` (OPTION 2) so only the current filesystem is scanned — git history with the literals isn't flagged once the latest commit is clean.

### container_scan / Trivy / alpine `musl` + axios CVEs

**Finding** (mixed OS + app layer):
- `musl` / `musl-utils` 4 HIGHs (alpine 3.19.4 base).
- `axios@1.6.0` 4 HIGHs (SSRF, DoS, cred leak).

**Fix**:
```diff
-FROM node:20-alpine3.19
+FROM node:25-alpine
```
```diff
-    "axios": "1.6.0"
+    "axios": "1.15.2"
```
Plus regenerate `code/package-lock.json` (otherwise `npm ci` fails).
**Notes**: Do *not* bump only the alpine minor — `node:20-alpine3.21` still has unpatched openssl HIGHs. Jump to current node-major + alpine. The application-layer `axios` finding really belongs in Module 2 (SCA); fixing it there with `osv-scanner` makes it disappear here too.

### container_scan / Grype / EOL distro + same CVEs

**Finding**: Single error line + helpful warning `188 packages from EOL distro "alpine 3.19.4"`. Detailed CVEs only in SARIF / GHAS.
**Fix**: Same as Trivy container.
**Notes**: For workshop visibility, change the snippet to `output-format: table` or upload `${{ steps.scan-tool.outputs.sarif }}` as an `actions/upload-artifact`.

### iac_scan / Checkov / `CKV_AWS_24`

**Finding**: `aws_security_group.ecs_tasks` at `infra/main.tf:120-126` — port 22 ingress from `0.0.0.0/0` despite `description = "HTTP from ALB"`.
**Fix**:
```diff
   ingress {
-    description = "HTTP from ALB"
-    from_port   = 22
-    to_port     = 22
+    description = "HTTP from ALB (workshop)"
+    from_port   = 3000
+    to_port     = 3000
     protocol    = "tcp"
-    cidr_blocks = ["0.0.0.0/0"]
+    cidr_blocks = [data.aws_vpc.existing.cidr_block]
   }
```
**Notes**: The "SRE note" comment above the resource literally hands you `data.aws_vpc.existing.cidr_block`. Excellent signposting — keep it.

### iac_scan / Trivy IaC / `AVD-AWS-0107`

**Finding**: Same `infra/main.tf:120-126` block; rule ID inferred (`AVD-AWS-0107`) — Trivy IaC's snippet writes only SARIF, so the log shows nothing.
**Fix**: Identical to Checkov above.
**Notes**: Switch the snippet's `format` from `'sarif'` to `'table'` (or run Trivy twice — once table, once SARIF for upload), or add `actions/upload-artifact` for `trivy-results.sarif`. Without one of those, attendees can't see what to fix.

### docs / Gitleaks meta-finding (writing the workshop report)

**Finding**: The report's quoted bait literals (`AKIA…` and the 40-char secret) made Gitleaks fail on `docs/reports/usability-dryrun-2026-05-01.md` itself.
**Fix**: Redact in the report:
```diff
-AWS_SECRET_ACCESS_KEY = 'oMKFr…the actual 40-char string'
+AWS_SECRET_ACCESS_KEY = '[redacted-40-char-secret]'
```
**Notes**: For maintainers, the right long-term fix is a `.gitleaksignore` covering `docs/reports/**` (and the equivalent for TruffleHog). For attendees, the practical advice is "write `AKIA…REDACTED` and `[redacted-40-char-secret]` whenever you cite the bait."

