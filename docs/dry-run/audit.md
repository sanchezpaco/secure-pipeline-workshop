# Dependency Audit — Phase 1 Output (revised)

**Date:** 2026-04-24
**Repo state:** `main` at `2faa532` (1:1 with `upstream/main` after Phase 0 reset)

## Correction to first draft

My first draft said "the workshop is already current" — that was wrong. I classified actions by their pinned version comments without checking against upstream. Today the pins are ~6 months behind: e.g. `actions/checkout` is on v4.2.2 but v6.0.2 exists; `github/codeql-action` is on v3.30.1 but v4.35.2 exists.

Also wrong: I classified `ruby/setup-ruby@master` and `zizmorcore/zizmor-action@main` as BUMP for supply-chain hardening. They're actually **KEEP** — the `claws-config.yml` has `UnpinnedAction: trusted_authors: ["actions"]`, so these unpinned third-party actions are **intentional findings** the module teaches attendees to detect.

## Final classification

### 🟢 KEEP (intentional teaching material)

| Item | File | Reason |
|---|---|---|
| `lodash@4.17.15` | `code/package.json` | SCA / code-scan CVE material |
| `node:16.14.0-alpine` | `code/Dockerfile` | Container-scan CVE material |
| `ruby/setup-ruby@master` | `workshop/pipeline_scan/claws/workflow.yml` | Intentional claws finding (UnpinnedAction rule, author not in trusted list) |
| `zizmorcore/zizmor-action@main` | `workshop/pipeline_scan/zizmor/workflow.yml` | Intentional finding for both zizmor and claws UnpinnedAction rule |

### 🟠 BUMP — user chose "major bumps también"

All bumps re-pin to SHA + tag comment.

**Group A — Core CI actions**

| Action | From | To |
|---|---|---|
| `actions/checkout` | v4.2.2 | v6.0.2 |
| `actions/upload-artifact` | v4.6.2 | v7.0.1 |
| `actions/github-script` | v7.0.1 | v9.0.0 |
| `aws-actions/configure-aws-credentials` | v4.1.0 | v6.1.0 |

**Group B — Docker build actions**

| Action | From | To |
|---|---|---|
| `docker/setup-buildx-action` | v3.11.1 | v4.0.0 |
| `docker/metadata-action` | v5.7.0 | v6.0.0 |
| `docker/build-push-action` | v6.18.0 | v7.1.0 |

**Group C — Security scanners**

| Action | From | To |
|---|---|---|
| `aquasecurity/trivy-action` | 0.28.0 | v0.36.0 |
| `anchore/scan-action` | v6.5.0 | v7.4.0 |
| `github/codeql-action/*` | v3.30.1 / v3.29.5 | v3.35.2 (v4 reverted — see note below) |
| `turbot/steampipe-action-setup` | v1 | v1.6.0 |
| `dependency-check/Dependency-Check_Action` | @main SHA | v1.1.0 |

### 🔵 Already current (no action)

- `bridgecrewio/checkov-action@v12.1347.0` — still latest
- `turbot/powerpipe-action-setup@v1` — latest is v1.0.0 = current
- `turbot/powerpipe-action-check@v1.0.1` — current
- `google-gemini/gemini-cli-action` — no release tags published (404 on `/releases/latest`), keep SHA-pinned as-is

### Terraform — leave as-is for now

- `required_version = ">= 1.0"` and `aws ~> 5.0` both stay. Bumping Terraform or the AWS provider majors mid-audit risks changing the set of IaC findings Checkov/Trivy report for `infra/main.tf`, which is outside the tooling-only scope.

## Commit plan

1. **Commit A** — Group A (core CI actions)
2. **Commit B** — Group B (docker actions)
3. **Commit C** — Group C (security scanners)

Each commit reviewable with `git diff HEAD~1 HEAD`. If Phase 3 exposes a breakage, `git revert <commit>` rolls back that group cleanly.

## Risks acknowledged

- Major bumps can change input/output schemas. Examples to watch:
  - `actions/checkout` v4→v6: default permissions, `persist-credentials` behaviour
  - `docker/build-push-action` v6→v7: attestation and provenance defaults
  - `github/codeql-action` v3→v4: CodeQL bundle version floor, config schema
  - `anchore/scan-action` v6→v7: output format changes
  - `aws-actions/configure-aws-credentials` v4→v6: OIDC/role-chaining behavior
- Mitigation: if any module fails in Phase 3 walkthrough due to a bump, revert only that commit.

## Phase 3 revert — codeql-action v4 → v3.35.2

**Trigger:** zizmor in `pipeline_scan/zizmor/workflow.yml` failed with:
```
'impostor-commit' audit failed on file://./.github/workflows/code-scan.yml
  HTTP status client error (403 Forbidden) for url
  https://api.github.com/repos/github/codeql-action/compare/refs/tags/codeql-bundle-20210621...7fc6561ed893d15cec696e062df840b21db27eb0
```

**Root cause:** zizmor's `impostor-commit` audit calls GitHub's compare API between the oldest tag in the repo (`codeql-bundle-20210621`, from 2021) and the pinned SHA. The compare payload for 4+ years of `github/codeql-action` history is too large and GitHub returns 403. This didn't happen with v3 pins because the compare window was shorter.

**Resolution:** partial revert in commit `fcc8852` (plus `d9c9289` for the already-substituted `.github/workflows/code-scan.yml`). All `github/codeql-action/{init,analyze,upload-sarif}` pins moved from v4.35.2 → v3.35.2 (same-major latest). This is still a bump over the original v3.30.1 / v3.29.5 pins and unifies the inconsistency.

**Update (2026-04-25):** subsequent investigation showed the zizmor failure is reproducible on **both** v3 and v4 of codeql-action — the trigger isn't the major version, it's that any SHA pin to `github/codeql-action` causes zizmor's `impostor-commit` audit to deterministically fail on the GitHub compare API (404 / 403). With the workshop-level fix `online-audits: "false"` for zizmor (commit `b728a50`), the v3↔v4 distinction is moot for the zizmor issue.

**TODO — bump codeql-action back to v4 before December 2026:** GitHub deprecates the v3 codeql-action bundle in Dec 2026; pinning to v3.35.2 indefinitely is a known dead-end. Once `online-audits: "false"` is in place upstream and verified, re-bump `github/codeql-action/{init,analyze,upload-sarif}` to a v4.x SHA. Track upstream advisories for the deprecation date.

**Reinforcing signal observed in CI on 2026-04-25** — every workflow that uses `github/codeql-action/upload-sarif@<v3.x SHA>` now emits this annotation:

> Node.js 20 actions are deprecated. The following actions are running on Node.js 20 and may not work as expected: github/codeql-action/upload-sarif@b2f9ef845756500b97acbdaf5c1dd4e9c1d15734. Actions will be forced to run with Node.js 24 by default starting June 2nd, 2026. Node.js 20 will be removed from the runner on September 16th, 2026. Please check if updated versions of these actions are available that support Node.js 24. To opt into Node.js 24 now, set the FORCE_JAVASCRIPT_ACTIONS_TO_NODE24=true environment variable on the runner or in your workflow file. Once Node.js 24 becomes the default, you can temporarily opt out by setting ACTIONS_ALLOW_USE_UNSECURE_NODE_VERSION=true.
> https://github.blog/changelog/2025-09-19-deprecation-of-node-20-on-github-actions-runners/

This is the same v3 / Node.js 20 issue from another angle: codeql-action v3 ships its action.yml with `using: node20`. v4 ships with `using: node24`. After Sept 16, 2026 v3 simply won't run. Bumping to v4 is no longer optional, and the only blocker (zizmor's impostor-commit fail) is now mitigated by `online-audits: "false"`.

**Learning:** when bumping a major action, a downstream security scanner (zizmor here) may fail for reasons unrelated to the action's own API — in this case because GitHub's compare API chokes on large histories. Worth a note in future bump reviews.
