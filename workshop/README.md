# Perfect Pipeline Introduction

This workshop is designed to help you understand the importance of shift-left security and how to implement comprehensive security scanning in your pipeline.

All of this without forgetting the human factor, as the final objective is not blind security but enabling your users (or yourself) to build secure software effectively.

Let's start with some important concepts:

### What is a Pipeline? CI/CD/CS?

A pipeline is a series of automated steps that build, test, and deploy software. It is a key part of the software development lifecycle and helps ensure that software is delivered with high quality and security.

Related to pipelines, you have probably heard about the concepts of Continuous Integration and Continuous Delivery. But to build a truly modern and resilient pipeline, we need to formally add a third component: Continuous Security.

- **Continuous Integration (CI)**: This is where developers merge code into a shared branch multiple times a day. Every merge kicks off an automated build and test, ensuring the mainline of your code is never broken for long.

- **Continuous Delivery (CD)**: This takes CI a step further. It’s an automated process that ensures any change passing the tests can be safely deployed to production with a single click, making release day a non-event.

- **Continuous Security (CS)**: This is the crucial next step in the evolution, embedding security into the process. It means automating security controls and tests throughout the pipeline, just like we do for quality, to find vulnerabilities when they are cheapest and easiest to fix.

<p align="center">
<img src="./imgs/CI_CD_CS.png" alt="CI/CD/CS" width="500">
</p>

### What is Shift-Left Security?

Shift-left security is a security practice that aims to move security checks and mitigations as early as possible in the software development lifecycle. This approach helps catch security issues early, before they reach production, and reduces the risk of vulnerabilities being exploited.

<p align="center">
<img src="./imgs/shift-left.png" alt="Shift-Left Security" width="800">
<p align="center"><em>Based on the image from https://devopedia.org/shift-left</em></p>
</p>


## Workshop Index
We suggest you follow the workshop in the following order, but feel free to jump around and explore the different modules.

1. [Pipeline Security Scan](pipeline_scan/)
2. [Code Security Analysis](code_scan/)
3. [Secrets Scan](secrets_scan/)
4. [Container Security Scanning](container_scan/)
5. [Infrastructure as Code Security Scan](iac_scan/)
6. [Runtime Infrastructure Scan](runtime_infra_scan/)
7. [AI Security Analysis](ai_scan/)

## Prerequisites

Before you start:

1. **Fork this repository** — you need write access to push, create branches, and configure secrets.

   [![Fork this repo](https://img.shields.io/badge/Fork-this_repo-2ea44f?logo=github&style=for-the-badge)](https://github.com/unicrons/secure-pipeline-workshop/fork)

2. **Clone your fork and create a working branch.** Keep `main` clean so you can compare your changes against the upstream and reset easily if something goes sideways.

   ```bash
   git clone git@github.com:<your-user>/secure-pipeline-workshop.git
   cd secure-pipeline-workshop
   git checkout -b workshop
   ```

3. **(Recommended) Enable GitHub Advanced Security on your fork** — some tools (Semgrep, Grype, Trivy IaC) upload SARIF and only show full findings in the *Code Scanning* tab. Without GHAS, their job logs look thin; each module's `Solutions` section tells you where to look instead.

4. **Per-module secrets and variables** — most modules work out of the box. The ones that don't:

   | Module | What you need | Where to get it |
   |---|---|---|
   | 2. Code Scan | `NVD_API_KEY` *(secret, optional but recommended for OWASP Dependency Check)* | [Request from NVD](https://nvd.nist.gov/developers/request-an-api-key) |
   | 6. Runtime Infra Scan | `AWS_IAM_ROLE_ARN` *(secret)*, `AWS_REGION` + `AWS_IAM_ROLE_SESSION_DURATION` *(vars)* | IAM role with `SecurityAudit` + `ViewOnlyAccess` and a GitHub OIDC trust policy |

   Configure them at `Settings → Secrets and variables → Actions` on your fork.

## Workshop flow

The pipeline runs through `pipeline-orchestrator.yml`, which calls each module's workflow at `.github/workflows/<module>-scan.yml`. Every module ships as a **placeholder job** that you replace with a real tool's job — this is the core mechanic of the workshop.

For each module:

1. **Read** the module's `README.md` for context (why, common issues, tools available).
2. **Pick a tool** and open `workshop/<module>/<tool>/workflow.yml`. The header comments list any extra prerequisites for that tool.
3. **Replace** the entire `jobs:` section of `.github/workflows/<module>-scan.yml` with the `jobs:` section from the tool's `workflow.yml`. Keep the file's `name:`, `on:`, `inputs:`, `secrets:`, and `outputs:` blocks intact.
4. **Commit and push** to your fork. The orchestrator triggers automatically — follow it in the *Actions* tab.
5. **Read the failure**: each module ships with a planted **bait** (a misconfiguration or vulnerable line) the scanner is meant to catch. The job log points at the file and line. If you get stuck, the module's `Solutions` section is your safety net.
6. **Fix the bait** and push again until the job goes green ✅.
7. **(Optional)** Try a second tool in the same module — it usually flags the same bait, so the existing fix clears it too.
8. **Move on** to the next module.

> ℹ️ **The orchestrator runs every module's job on every push.** Modules whose placeholder you haven't replaced stay green by design (the placeholder just echoes a message). Focus on the job for the module you're working on.

> ℹ️ **Module 6 exception**: the runtime infra scan runs against a *live AWS account*, so there's no planted bait to fix. It's expected to pass green even when Prowler reports findings — see its `Solutions` section.
