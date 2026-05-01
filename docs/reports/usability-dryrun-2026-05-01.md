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

_(pending)_

## Module 3 — Secrets Scan

_(pending)_

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
