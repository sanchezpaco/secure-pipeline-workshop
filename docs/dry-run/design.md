# Workshop Reset, Dependency Refresh and Dry-Run — Design

**Date:** 2026-04-24
**Repo:** `sanchezpaco/secure-pipeline-workshop` (fork of `unicrons/secure-pipeline-workshop`)
**Owner:** paco.sanchez

## Goal

Leave the fork in a clean, current-tooling state that faithfully reproduces the upstream workshop, then walk through every module end-to-end — as a first-time attendee would — and record findings and friction.

## Non-goals

- Changing workshop pedagogical content (module structure, exercises, expected findings).
- Updating dependencies that are **intentionally vulnerable** (teaching material for the scanners). These stay pinned.
- Deploying real AWS infrastructure. Modules that require a live AWS account are skipped and documented.

## Success criteria

1. `main` of the fork matches `upstream/main` byte-for-byte (or a documented controlled delta).
2. Every GitHub Action in `.github/workflows/` resolves to a non-deprecated version; every security-scanner tool version is current-stable.
3. A dry-run walkthrough of all seven modules is documented in `workshop/DRY_RUN_REPORT.md` with a green/yellow/red status per module and concrete fix notes where friction was found.

## Phases

Work proceeds in strict order. Each phase has a gate — no advancing until the prior phase is signed off.

### Phase 0 — Reset main to upstream

1. Add `upstream` remote: `git@github.com:unicrons/secure-pipeline-workshop.git`
2. `git fetch upstream`
3. Show the user the diff between `origin/main` and `upstream/main` (`git log --oneline origin/main..upstream/main` and the reverse).
4. Ask user to confirm before destructive action.
5. `git checkout main && git reset --hard upstream/main`
6. `git push origin main --force-with-lease` (user already authorized force-push to origin main for this fork in brainstorming).
7. Delete `dry-run` branch locally (`git branch -D dry-run`) and remotely (`git push origin --delete dry-run`).

**Gate:** `git log origin/main -5` matches upstream; `dry-run` branch is gone.

### Phase 1 — Dependency audit (read-only)

Produce `workshop/DRY_RUN_AUDIT.md` with a table per category. Every entry is classified as one of:

- **KEEP** — intentionally vulnerable / required by a workshop exercise
- **BUMP** — tooling that should be updated to latest stable
- **REVIEW** — unclear; needs per-module README cross-reference before deciding

Categories to audit:

| Scope | Files | Expected classification |
|---|---|---|
| Node app dependencies | `code/package.json`, `code/package-lock.json` | KEEP `lodash@4.17.15` (SCA / code-scan material) |
| Base image | `code/Dockerfile` | KEEP `node:16.14.0-alpine` (container-scan material) |
| GitHub Actions `uses:` | `.github/workflows/*.yml` | BUMP all to latest stable major (e.g. `actions/checkout@v4`, `setup-node@v4`) |
| Scanner actions | workflows referencing trivy, checkov, gitleaks, semgrep, zaproxy, prowler, steampipe | BUMP to latest stable release tags, pinned by SHA where reasonable |
| Terraform providers | `infra/providers.tf` | REVIEW — must still produce the IaC findings the `iac_scan` module expects |
| Terraform `required_version` | `infra/providers.tf` | BUMP to a current supported minor |

**Gate:** audit doc committed and reviewed by the user. No code changes yet.

### Phase 2 — Apply updates

One commit per category, in this order, to keep the diff reviewable:

1. **Commit A — GitHub Actions versions**: bump `actions/*` in all workflow files.
2. **Commit B — Scanner tool versions**: bump the third-party scanner actions (trivy-action, checkov-action, gitleaks-action, etc.) and, where the scanner is invoked as a CLI inside a step, bump the pinned version.
3. **Commit C — Terraform** (only if Phase 1 flagged BUMP items that don't break expected findings): bump provider and `required_version`.

After each commit: run `git diff HEAD~1 HEAD` and show the user before proceeding. No force-push during this phase — these are regular commits on `main`, push with normal `git push`.

**Gate:** `gh workflow list` still resolves; yaml lint clean; no workflow references a deprecated action.

### Phase 3 — Dry-run walkthrough

1. Recreate a clean `dry-run` branch from the updated `main`: `git checkout -b dry-run && git push -u origin dry-run`.
2. Follow each module in the documented order, acting as a first-time attendee:
   1. `pipeline_scan` (zizmor, claws, prowler, extra)
   2. `code_scan`
   3. `secrets_scan`
   4. `container_scan`
   5. `iac_scan`
   6. `runtime_infra_scan` — **SKIPPED for live AWS** (Prowler/Steampipe need real credentials). Document as "requires AWS, skipped — verified workflow YAML is valid and triggers correctly via `workflow_dispatch` dry-test only."
   7. `ai_scan` (Gemini — requires `GEMINI_API_KEY` secret; user provides via GitHub repo secrets; we never handle the value).

   For each module:
   - Read its `README.md` top-to-bottom as if new.
   - If the module contains multiple tool subdirectories (e.g. `pipeline_scan/{zizmor,claws,prowler,extra}`), exercise each tool's `workflow.yml` as a separate sub-run and record its status independently.
   - Execute the instructions verbatim (commit, push, trigger workflow).
   - Record in `DRY_RUN_REPORT.md`:
     - Expected findings (per README) vs. actual findings produced.
     - Friction: broken links, ambiguous steps, outdated screenshots, unclear prerequisites.
     - Status: 🟢 clean / 🟡 minor friction / 🔴 blocker.
3. Modules requiring AWS infra (`deploy-infrastructure`, `deploy-application`, `runtime-infra-scan`) are skipped. We verify only: the YAML parses, required secrets are documented, workflow appears in the Actions UI.

**Gate:** `DRY_RUN_REPORT.md` committed with per-module status. Friction items triaged into: "fix now" vs. "file upstream issue".

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| A scanner bump changes the set of findings, invalidating a module's expected-output screenshots or instructions | Phase 1 classifies BUMP items as REVIEW when a module references specific finding counts; during Phase 2, we re-run the scanner locally (where possible) and compare findings before committing |
| `upstream/main` has diverged in a way that loses useful edits in `origin/main` | Phase 0 step 3 shows the diff explicitly and we confirm before reset. Current inspection: `origin/main` has no commits beyond what came from upstream (commits are all `Merge PR from unicrons/initial-release`), so loss risk is ~zero, but we verify |
| A GitHub Actions major-version bump changes input/output schemas | Test each workflow after Commit A by running it via `workflow_dispatch` on the `dry-run` branch; rollback if broken |
| Leaking a secret (GEMINI_API_KEY, any AWS key) into a commit or the report | Use GitHub Secrets only; never paste values; reference via `${{ secrets.NAME }}` placeholders; audit diff before every push |
| Force-push on `main` of the personal fork | Explicit user confirmation in Phase 0, step 4. `--force-with-lease` used to prevent overwriting unseen commits |

## Artifacts produced

- `workshop/DRY_RUN_AUDIT.md` — Phase 1 output, classification tables
- Commits A/B/C on main — Phase 2 output
- Clean `dry-run` branch pushed to origin — Phase 3 setup
- `workshop/DRY_RUN_REPORT.md` — Phase 3 output, per-module status and friction log

## Out of scope / explicitly deferred

- Upstream contributions: if we find bugs in the workshop that are upstream's problem, we file them as issues/PRs on `unicrons/secure-pipeline-workshop` in a **follow-up** session, not during this dry-run.
- Deploying the Terraform infra against a real AWS account.
- Porting the workshop to non-GitHub CI systems.
- Any change to license, security policy, or CODEOWNERS.
