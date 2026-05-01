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

_(in progress)_

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
