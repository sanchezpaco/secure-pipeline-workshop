# Container Security Scanning

This workshop module covers container security scanning to identify vulnerabilities in container images and configurations.

## Why is Container Security Important?
Containers are now one of the most common ways to deploy applications. They are also attractive targets for attackers who can use them to access underlying infrastructure, steal secrets (including cloud credentials), or execute arbitrary code.

Container security scanning analyzes images and container configurations to identify security vulnerabilities, misconfigurations, and compliance issues before releasing them.

## Common Container Security Issues

### Image Vulnerabilities:
- **Image Vulnerabilities** - Outdated or vulnerable dependencies, both in the base image and in the application dependencies.
- **Supply Chain** - Untrusted base images. Trojanized or typo-squatted images pulled from public registries.
- **Secrets in Images** - Hardcoded credentials or keys.

### Configuration Issues:
- **Root User** - Running containers as root
- **Excessive Privileges** - Unnecessary capabilities can allow container escape.
- **Exposed Ports** - Unintended network exposure
- **Missing Security Controls** - No health checks or resource limits
- **Weak Network Segmentation** - Overly permissive ingress rules allow lateral movement
- **Dangerous Volumes** - Mounting host volumes can expose sensitive data or allow privilege escalation.

## Tools Used in This Module

- [**Trivy**](https://github.com/aquasecurity/trivy) - Vulnerability, misconfiguration and secret scanner for containers
  - Also supports scanning filesystems, IaC or repos, but we focus on containers for this module
- [**Grype**](https://github.com/anchore/grype) - Fast vulnerability scanner for container images and filesystems
  - Developed by Anchore, uses multiple vulnerability databases and generates SBOM automatically

## Learning Objectives

By the end of this module, you will:
- Understand container security risks
- Learn to scan images for vulnerabilities
- Understand supply chain security for containers
- [🔜] Identify Dockerfile security best practices

> [!TIP]
> **A clean scan can take more than one fix.** Bumping the base image
> clears OS-package and language-runtime CVEs, but transitive dependencies
> bundled inside the new image (e.g., libraries shipped with the global
> package manager) may still surface as HIGH findings. Expect to fix in
> layers — re-run the scanner between each change to see what's left.

## Security Checklist

- [ ] **Use minimal, trusted base images** and rebuild them regularly.
- [ ] **Patch all High/Critical CVEs** before shipping; fail the build if any remain.
  - Note: You can also ignore them if you verified the vulnerability is not exploitable.
- [ ] **Run containers as a non-root user** with only the capabilities they truly need.
- [ ] **Remove build tools & secrets** via multi-stage builds to shrink the attack surface.
- [ ] **Sign images and verify signatures** at pull/admission time to ensure provenance.
- [ ] **Enable runtime protection** ([seccomp](https://docs.docker.com/engine/security/seccomp/)/[AppArmor](https://docs.docker.com/engine/security/apparmor/) profiles, resource limits, live detection).
- [ ] **Implement health and readiness checks**



## References
- [NetRise Releases Supply Chain Visibility & Risk Study, Edition 2: Containers, Revealing Signicant Visibility and Risk Challenges within Common Containers](https://www.netrise.io/en/company/announcements/netrise-releases-supply-chain-visibility-risk-study-revealing-signicant-visibility-and-risk-challenges-within-common-containers): Research finds containers have over 600 vulnerabilities a piece on average.
- [Top 10 Container Security Issues](https://www.sentinelone.com/cybersecurity-101/cloud-security/container-security-issues/)
- [Docker Engine security](https://docs.docker.com/engine/security/)

### Other Tools
- [slimtoolkit/slim](https://github.com/slimtoolkit/slim): Tool to reduce the size of container images (making them more secure).

## Solutions (spoilers — open only when stuck)

> The container is built from `code/Dockerfile`, which ships with an EOL base image, and uses `code/package.json`, which pins a vulnerable version of `axios`. Both layers are intentional bait. Both scanners (Trivy and Grype) flag the same root causes.

> ⚠️ **Lock-file footgun**: when you bump `axios` in `package.json`, you must also regenerate `code/package-lock.json` — otherwise `npm ci` fails inside the Docker build with `EUSAGE: package.json and package-lock.json are out of sync`, before the scanner even runs. The error looks like a Trivy/Grype issue but it isn't.

<details>
<summary><b>Trivy</b> — alpine OS-layer HIGHs + axios HIGHs</summary>

**What Trivy flagged** (two tables):
- **OS layer** (`alpine 3.19.4`): `musl` and `musl-utils` HIGHs (e.g. `CVE-2025-26519`, `CVE-2026-40200`).
- **Application layer** (`axios (package.json)` 1.6.0): HIGHs for SSRF (`CVE-2024-39338`, `CVE-2025-27152`), DoS (`CVE-2025-58754`), proto-key DoS (`CVE-2026-25639`).

**Reading the output**: Trivy prints a clean table per layer (Library / Vulnerability / Severity / Status / Installed Version / Fixed Version / Title) — this is the gold-standard output across the whole workshop.

> ⚠️ **Don't just bump alpine minor**: bumping `node:20-alpine3.19` → `node:20-alpine3.21` will *increase* the finding count (alpine 3.21 still ships an unpatched openssl). Jump to a current node-major instead. Also: Trivy's "Fixed Version" column is per-CVE; bumping to that version only closes the CVE you happened to look at.

**Fix** — bump base image and dependency, then regenerate the lockfile:

```diff
-FROM node:20-alpine3.19
+FROM node:25-alpine
```

```diff
   "dependencies": {
-    "axios": "1.6.0"
+    "axios": "1.15.2"
   }
```

```bash
cd code
rm -f package-lock.json
npm install --package-lock-only
cd ..
git add code/Dockerfile code/package.json code/package-lock.json
```

> The application-layer `axios` finding really belongs in Module 2 (SCA). If you've already fixed it there with `osv-scanner`, this part is a no-op here.

</details>

<details>
<summary><b>Grype</b> — same root cause, terser output</summary>

**What Grype reports**: a single line — `[…] ERROR discovered vulnerabilities at or above the severity threshold` — plus a helpful warning: `188 packages from EOL distro "alpine 3.19.4"`. The actual CVE list is uploaded as SARIF to the GitHub Code Scanning tab; if your fork doesn't have GHAS, the run page shows nothing actionable. Two snippet improvements help: switch `output-format: sarif` to `table` for the workshop, or upload the SARIF as an `actions/upload-artifact`.

**Same root cause as Trivy**: EOL alpine base + vulnerable `axios`.

**Fix** — identical to the Trivy fix above.

</details>