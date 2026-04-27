# Dry-Run Report

**Date:** 2026-04-24
**PR:** https://github.com/sanchezpaco/secure-pipeline-workshop/pull/3
**Branch:** `bump-versions` (3 commits on top of upstream/main)

## Workshop-level fixes to send upstream (MUST)

These need a PR to `unicrons/secure-pipeline-workshop`. They are not attendee-level fixes — they block the workshop for everyone.

1. **Add `online-audits: "false"` to the zizmor invocation** in `workshop/pipeline_scan/zizmor/workflow.yml`
   Reason: zizmor v1.24.1 has a **general** design flaw — every online audit (impostor-commit, known-vulnerable-actions, and others) hard-crashes the whole run (`fatal: no audit was performed`) when it encounters a GitHub API HTTP error. We saw at least two distinct failures (`compare` 403 for `github/codeql-action` cross-repo diff, and `advisories` 403 for `actions/checkout@v6.0.2`) — fixing one only surfaces the next. Upstream zizmor issues #1874 and #1252 describe the pattern.
   Workaround adopted in dry-run (commit `b728a50`): `online-audits: "false"` input to the zizmor-action, which disables all online audits in one shot. Trade-off: attendees lose `repojacking`, `known-vulnerable-actions`, `impostor-commit`, `artipacked` online checks — they still get the bulk (`unpinned-uses`, `template-injection`, `excessive-permissions`, `hardcoded-container-credentials`, etc.).
   Suggestion: in parallel, open a bug on `zizmorcore/zizmor` asking online audits to degrade API errors to warnings.

   #### Local reproduction of the zizmor `impostor-commit` bug (for upstream report)

   **Version:** zizmor v1.24.1 (latest at 2026-04-24)
   **Affects:** any workflow pinning `github/codeql-action/*@<sha>` (and likely other huge action repos)

   **Setup** — minimal `.github/workflows/code-scan.yml` that triggers it:
   ```yaml
   name: SAST
   on: workflow_call
   jobs:
     codeql-scan:
       runs-on: ubuntu-latest
       steps:
         - uses: actions/checkout@de0fac2e4500dabe0009e67214ff5f5447ce83dd # v6.0.2
         - uses: github/codeql-action/init@b2f9ef845756500b97acbdaf5c1dd4e9c1d15734 # v3.35.2
         - uses: github/codeql-action/analyze@b2f9ef845756500b97acbdaf5c1dd4e9c1d15734 # v3.35.2
   ```

   **Reproduce:**
   ```bash
   export GH_TOKEN=$(gh auth token)   # authenticated PAT, plenty of quota
   gh api rate_limit --jq '.resources.core | "pre:  \(.remaining)/\(.limit)"'

   time zizmor --min-severity=high .github/workflows/code-scan.yml
   echo "exit=$?"

   gh api rate_limit --jq '.resources.core | "post: \(.remaining)/\(.limit)"'
   ```

   **Observed:**
   ```
   pre:  3642/5000

   INFO zizmor: 🌈 zizmor v1.24.1
   WARN audit: impostor_commit: fast path impostor check failed for
        github/codeql-action/init@b2f9ef84...: request error while accessing GitHub API
   [~17 min of silent retries with no progress output]
   fatal: no audit was performed
   'impostor-commit' audit failed on file://.github/workflows/code-scan.yml
   Caused by:
       2: Cache error: Cache error: error sending request for url
          (https://api.github.com/repos/github/codeql-action/compare/refs/tags/v2.13.4...b2f9ef84...)

   real  17m17s
   exit=1

   post: 1174/5000   # 2468 API calls consumed, all returning 403 Forbidden
   ```

   **Why it fails (refined after replicating the calls directly with curl):**
   - `impostor-commit` proves a pinned SHA is reachable from the upstream repo's refs by calling `GET /repos/github/codeql-action/compare/<ref>...<sha>` for many candidate refs.
   - For `github/codeql-action`, the **direct API call returns `404 Not Found` deterministically** (verified with `curl` both with PAT auth and without — same result), because the older refs zizmor tries (`v2.13.4`, `codeql-bundle-20210622`, random branches) and the modern SHA simply have no ancestral relation in the repo's reorganized history. `404` is non-retryable: the answer won't change on retry.
   - A nearby compare like `compare/v3.30.1...v3.35.2` returns `200 OK` (2.1 MB JSON), confirming the endpoint is healthy — only the cross-era refs fail with 404.
   - Zizmor nonetheless retries aggressively. After ~2500 retries it eventually hits its own rate limit and surfaces a `403 Forbidden` — that's the error message we see in the CI logs, but the actual underlying problem is the per-ref 404 storm, not a hard endpoint size limit.
   - Double "Cache error: Cache error:" in the error chain is a separate formatting bug.

   **Extra notes:**
   - Not a rate-limit issue: reproduced with a PAT sitting at `3642/5000`; the 403 is on the endpoint, not on the quota.
   - The retry loop consumes ~2500 requests in ~17 min — a single invocation can reasonably brick the hourly quota of a modest PAT.
   - Workaround: `online-audits: "false"` (zizmor-action input) or `--no-online-audits` (CLI). Loses repojacking / known-vulnerable-actions / artipacked as collateral.
   - Related upstream issues: `zizmorcore/zizmor#1874` (artipacked fails hard on github.com HTTP errors), `#1252` (rate-limit errors surfaced as misleading validation errors). Neither targets impostor-commit specifically.

   **Suggested zizmor fixes (priority order):**
   1. **Don't retry 404 on the compare endpoint**. 404 from `/compare/<base>...<head>` means the refs are not ancestrally related — non-recoverable. Bail out for that ref immediately and try the next one.
   2. Treat exhausted-refs-without-success as `WARN` for the specific `uses:` entry, not fatal for the whole run.
   3. Cap total retry budget per audit (today ~2500 calls / ~17 min per pin is unsustainable).
   4. Emit a summary line naming the offending `repo@sha` pins that couldn't be verified, so users can decide per-pin what to do.
   5. Fix the duplicated "Cache error: Cache error:" formatting in the error chain.

2. **CodeQL false-pass** — Workshop currently queries `gh api /code-scanning/alerts?pr=<n>&tool_name=CodeQL`. The `?pr=<n>` filter scopes results to **alerts introduced by the PR diff**, not all open alerts on the codebase. Two failure modes:
   - No baseline on `main` → API returns 0 → green falso (estado actual del dry-run).
   - Baseline en main + PR no cambia el archivo vulnerable → API returns 0 even though the vulnerabilities are still there (still false pass — the workshop expects scanners to teach attendees about *existing* vulns, not just regressions introduced by their PR).
   The teaching intent of the workshop is "scan the codebase, detect intentional vulns, learn to fix them" — that requires reporting all open findings, not the PR delta.
   Options: (a) drop the `?pr=<n>` filter so the count reflects all open alerts on the ref, (b) count raw SARIF findings from the analyze step output (mirrors what Semgrep does and works on first run), (c) require attendees to set up CodeQL on `main` AND introduce vulns inside their PR (changes the pedagogy and adds setup overhead).

3. **Dependency-Check needs NVD_API_KEY** — see Tool 2.3 below. Since 2024 the NVD feed rate-limits unauthenticated clients aggressively, so dependency-check effectively returns 0 findings. Workshop should document the need for this secret or swap to a tool that doesn't require NVD (npm audit, osv-scanner, trivy fs).

4. **TruffleHog flags hide the fake-AWS-credential finding** — `workshop/secrets_scan/trufflehog/workflow.yml` invokes trufflehog with `--results=verified,unknown --fail`. Trufflehog's "verified" mode calls AWS with the candidate credential; the workshop's `code/src/simple-app.js:3-4` uses fake AKIA keys, AWS responds "invalid access key", and trufflehog **discards the match as a false positive** → exit 0 (false-pass).
   Reproduction (local): scan a file containing `AWS_ACCESS_KEY_ID = 'AKIA2T2SJH6MS337PDWL'` with `trufflehog filesystem . --results=verified,unknown --fail` → 0 secrets, exit 0. Same scan with `--no-verification` → 1 unverified result for the AKIA, exit 183.
   Pedagogical impact: an attendee who runs only TruffleHog (skipping gitleaks) sees green from the first run and never goes through the fail/fix loop the module is designed to teach.
   Suggested fix: use `--results=verified,unknown,unverified` (or `--no-verification`) so detected-but-unverified secrets still trigger the fail. Alternatively, add a README note explaining the limitation, but that's weaker because the green result is misleading regardless.

5. **GitHub Push Protection blocks the workshop's didactic AKIA commit** — When the attendee tries to commit `code/src/simple-app.js` with the hardcoded AWS credentials (the very setup that secrets_scan is supposed to detect), GitHub's Push Protection rejects the push: `Push cannot contain secrets — Amazon AWS Access Key ID detected`. The workshop becomes un-runnable until the attendee manually unblocks each secret via the per-secret URL or disables Push Protection on their fork.
   Suggested fix: document in the workshop intro README that attendees must either (a) click the unblock URL when push is rejected, or (b) disable Push Protection in `Settings → Code security → Secret scanning → Push protection` before starting.

6. **Misleading "critical" wording in container_scan output** — `workshop/container_scan/{grype,trivy}/workflow.yml` prints `❌ Container scan failed - critical vulnerabilities detected` when the scanner exits non-zero, but the `severity-cutoff` is `high`. Attendees see "critical" and look in vain for critical-severity findings; the real findings are high or medium. Cosmetic but causes confusion.
   Suggested fix: change the message to `❌ Container scan failed - vulnerabilities at or above the configured severity threshold detected`.

7a. **ai_scan module needs a full revamp** — Skipped in this dry-run because the workshop's `workshop/ai_scan/gemini/workflow.yml` requires a `GEMINI_API_KEY` secret that's not provisioned by default. Beyond the secret setup, the module's pedagogy (LLM-driven security review of PR diffs) needs reconsideration in light of how AI tooling has evolved since the workshop was first written: prompts are unaudited, output is non-deterministic, hallucination risk is unaddressed, and the action calls a third-party API that may have changed surface area. Recommended actions for upstream:
   - Decide whether the module stays as a "single-tool demo" (Gemini) or becomes an abstract "AI-assisted security review" lesson with multiple optional providers (Claude, OpenAI, Gemini).
   - Provide a fallback path that doesn't require an external API key (e.g., local LLM via ollama, or skip the action with a meaningful no-op).
   - Add guardrails around prompt injection (the AI is reviewing PR diffs which can contain attacker-controlled text — this is its own attack surface and should be discussed in the module README).
   - Re-evaluate the security claims of "AI Security Analysis" — currently the README sells it as "contextual understanding" and "reduced false positives", which is overstated for current state-of-the-art models against unknown codebases.
   - Until the revamp is done, recommend marking the module as "experimental / out of scope for the v1 workshop deliverable".

7b. **container_scan needs a 2-step fix, not "just bump Node"** — Today the workshop ships `code/Dockerfile` with `FROM node:16.14.0-alpine` and no extra hardening. Bumping the Node base alone does NOT achieve a green scan in 2026:
   - `node:16.14.0-alpine` → 147 findings (alpine system + node binary + bundled npm 8.x)
   - `node:24-alpine` / `node:25-alpine` → still 1 HIGH (`GHSA-c2c7-rcm5-vvqj` in picomatch 4.0.3, bundled inside npm 11.x)
   - `node:20-alpine` (no npm install) → 11 HIGHs in npm's bundled `cross-spawn`/`glob`/`minimatch` (npm 10.x bundle)
   - **Only after BOTH bumping Node AND adding `RUN npm install -g npm@11.13.0`** does grype+trivy go green: that's because npm 11.13.0 bundles `picomatch ≥ 4.0.4`, which is the patched version.
   - Verified working combinations: `node:20-alpine + npm@11.13.0` ✅ green; `node:25-alpine + npm@11.13.0` ✅ green.

   **Pedagogically — 2-step lesson** (recommended for the upstream workshop):
   1. **Step 1**: Attendee bumps the Node base image (e.g., `FROM node:16.14.0-alpine → FROM node:25-alpine`). Old OS / node binary CVEs disappear.
   2. **Step 2**: Attendee discovers the residual transitive HIGH in npm's bundled deps and adds `RUN npm install -g npm@11.13.0 && npm cache clean --force` to the Dockerfile. That replaces the bundled npm tree with a patched one.
   3. ✅ scanners green.

   Why this is more honest than "just bump Node": real-world security has *layers*. The base image is one. The transitive deps the image ships are another. Bumping the base alone is necessary but not sufficient. Teaches the attendee to think about both layers and verifies the fix with two scanner re-runs.

   **Suggested upstream change**: rewrite the relevant section of `workshop/container_scan/README.md` (and possibly the module's README) to describe this 2-step model. Keep `code/Dockerfile` as `FROM node:16.14.0-alpine` (no npm install) so step 1 has visible payoff and step 2 has a concrete remaining finding to motivate.

8. **DAST module is referenced but not implemented** — `.github/workflows/dast.yml` exists as a placeholder in the orchestrator pipeline and instructs the attendee to "Copy from: workshop/dast/{tool}/workflow.yml", but that path does not exist. There is no `workshop/dast/` directory, no DAST README, and no DAST mention in the root or workshop READMEs. An attendee who follows the orchestrator flow hits a dead-end at this step.
   Workaround applied in dry-run (commit on `bump-versions`): replace the misleading placeholder text with an explicit "🚧 DAST — Coming soon" no-op message. The job still passes (no-op) so the orchestrator stays green.
   Suggested upstream change: either (a) ship the DAST module (e.g., OWASP ZAP / nuclei against the deployed app — would also need the runtime infrastructure, currently AWS-stubbed in deploy-application), or (b) leave the explicit "coming soon" placeholder so attendees aren't confused, and remove or document the orphan dast.yml's instructions.

## Baseline (all placeholders intact, only the 3 bump commits)

🟢 **PASS** — Every orchestrator child workflow runs green with the placeholders. The major bumps do not break action resolution or workflow YAML parsing.

## Module 1 — pipeline_scan

### Tool 1.1 — claws

- **Status:** 🟢 Works as intended (fails with the expected didactic finding)
- **Commit:** `fb0455a` — substituted placeholder with `workshop/pipeline_scan/claws/workflow.yml`
- **Finding produced:** `Violation: UnpinnedAction on .github/workflows/pipeline-scan.yml:23`
  - Line 23 is `uses: ruby/setup-ruby@master` — unpinned, author `ruby` not in `trusted_authors: ["actions"]`
  - Exactly the finding the module intends to teach
- **Friction:** none. Instructions worked verbatim; config was already in place.

### Tool 1.2 — zizmor

- **Status:** 🟢 Works as intended (passes because findings are below `min-severity: "high"`, but findings are detected and uploaded as SARIF to Security tab)
- **Commit:** `eca12d7` — substituted placeholder with `workshop/pipeline_scan/zizmor/workflow.yml`
- **Findings produced:** zizmor 1.24.1 emitted SARIF entries including rules `unpinned-uses` and `template-injection`. The `unpinned-uses` rule flagged `zizmorcore/zizmor-action@main` — the expected didactic finding.
- **Friction:** minor — the zizmor job is named `zizmor`, but the top-level `pipeline-scan.yml` placeholder declares `workflow_call` output referencing `jobs.pipeline-scan.outputs.result`. That output resolves to empty string. No runtime impact, but the workflow_call contract is technically broken. Not worth a fix for the workshop.

## Module 2 — code_scan

### Tool 2.1 — CodeQL (SAST)

- **Status:** 🟡 Runs but passes falsely. Not a bump regression — same behavior on upstream.
- **Commit:** `3df9db3`
- **Key logs:**
  - `##[warning]1 issue was detected with this workflow: Please specify an on.push hook to analyze and see code scanning alerts from the default branch on the Security tab.`
  - `Found 2 raw diagnostic messages.` (CodeQL analysis did detect issues)
  - `Found 0 CodeQL security alerts` (the `gh api /code-scanning/alerts?pr=3&tool_name=CodeQL` call returned zero)
  - Job prints `✅ CodeQL scan completed - no security alerts` and exits 0
- **Root cause:** CodeQL PR scans only surface alerts that are *new relative to a baseline* on the default branch. Since the default branch was never scanned, the API returns 0 even when the PR has findings. The workshop's pass/fail logic depends on this API count, so the step incorrectly reports success.
- **Suggested workshop fix (out of scope for this dry-run):** either (a) count raw SARIF findings instead of API alerts, or (b) instruct attendees to first merge the scan workflow to `main` (so CodeQL has a baseline) before opening the PR. At minimum, document the gotcha in the module README.
- **Friction:** this is the loudest bug in the workshop's current form. First-time attendees will see a green ✅ and assume CodeQL works when in fact it silently misses findings.

### Tool 2.2 — Semgrep (SAST)

- **Status:** 🟢 Works as intended (fails with the expected findings)
- **Commit:** `0a17aa0` (with later `d9c9289` to pin `upload-sarif` to v3.35.2)
- **Findings produced:** `Ran 129 rules on 7 files: 3 findings. ✅ Scan completed successfully. Findings: 3 (3 blocking)` → job exits 1 with `❌ Semgrep scan failed - vulnerabilities detected`. The 3 findings correspond to the hardcoded `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` at `code/src/simple-app.js:3-4` and the `exec("cat "+filePath)` command injection at line 41.
- **Friction:** SARIF upload step hit `API rate limit exceeded for installation` — this is a side effect of our intense API usage during this dry-run session, not a workshop bug. Semgrep's scan itself produced correct results.

### Combined pipeline_scan + code_scan issue — zizmor vs codeql-action

- **Status:** 🔴 Real bug in the workshop when both `pipeline_scan/zizmor` and `code_scan` are active (Semgrep or any other tool that references `github/codeql-action`).
- **Symptom:** `'impostor-commit' audit failed on file://./.github/workflows/code-scan.yml` → `HTTP status client error (403 Forbidden) for url https://api.github.com/repos/github/codeql-action/compare/refs/tags/codeql-bundle-20210621...<sha>` (or a random-branch variant).
- **Root cause:** zizmor's `impostor-commit` audit queries GitHub's `compare` API between a random ref in the target repo and the pinned SHA. For `github/codeql-action` the repo history is too large and the API returns 403. Reverting to older SHAs of codeql-action does not fix it — the issue surfaces for any SHA pin of that specific repo.
- **Workaround applied in this dry-run:** pipeline-scan temporarily reverted to the upstream placeholder (commit `974b618`) while testing code_scan tools in isolation.
- **Suggested workshop fix:** add a zizmor config (`zizmor.yml` in repo root) that skips `impostor-commit` for `github/codeql-action`, or upstream should teach users how to configure zizmor to tolerate this GitHub API quirk.
