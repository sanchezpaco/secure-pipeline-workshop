# Code Security Analysis (SAST/SCA)

This workshop module covers Static Application Security Testing (SAST) and Software Composition Analysis (SCA) to identify security vulnerabilities in application code and dependencies.

## What is SAST (Static Application Security Testing)?

SAST analyzes source code, bytecode, or binary code to identify security vulnerabilities without executing the program. It examines code patterns, data flow, and control flow to detect potential security issues.

## What is SCA (Software Composition Analysis)?

SCA identifies and analyzes open source components and third-party dependencies in applications to detect known vulnerabilities, license compliance issues, and outdated packages.

## Why is Code Security Important?

Malicious actors can exploit vulnerabilities in your code to gain unauthorized access, steal sensitive data, or disrupt your systems.

## Common Code Security Issues

### SAST Findings:
- **SQL Injection** - Unsafe database queries
- **Cross-Site Scripting (XSS)** - Unvalidated user input
- **Command Injection** - Direct execution of system commands with user data
- **Path Traversal** - Unsafe file access with user-controlled paths
- **Authentication Flaws** - Weak authentication mechanisms
- **Authorization Issues** - Missing access controls
- **Hardcoded Secrets** - Credentials in source code
- **Input Validation** - Missing or improper validation

### SCA Findings:
- **Known Vulnerabilities** - CVEs in dependencies
- **Outdated Packages** - Dependencies with security updates
- **License Issues** - Non-compliant licenses
- **Supply Chain Risks** - Malicious packages

## Tools Used in This Module

### SAST tools

- [**CodeQL**](https://github.com/github/codeql) - GitHub's semantic code analysis engine for deep security analysis
  - [GitHub Action](https://github.com/github/codeql-action) | [Documentation](https://codeql.github.com/docs/)
- [**Semgrep**](https://github.com/semgrep/semgrep) - Fast pattern-based security scanner with extensive rule sets
  - [Documentation](https://semgrep.dev/docs/) | [Community Rules](https://semgrep.dev/explore)

> **Note**: Different ecosystems have specialized SAST tools (e.g., ESLint with security plugins for JavaScript, Bandit for Python, Brakeman for Ruby on Rails) that can provide more targeted analysis alongside general-purpose scanners.

### SCA tools

These two SCA tools pull vulnerability data from different sources, so the same dependency may surface (or not) depending on the choice. Worth knowing both:

- [**osv-scanner**](https://github.com/google/osv-scanner) - Modern, lockfile-based scanner from Google. Queries [OSV.dev](https://osv.dev/), an open-standard database that aggregates GitHub Advisories, RustSec, PyPA, GoVulnDB and others. No external setup, no API keys, no DB downloads.
  - [GitHub Action](https://github.com/google/osv-scanner-action) | [Documentation](https://google.github.io/osv-scanner/)
- [**OWASP Dependency Check**](https://github.com/dependency-check/DependencyCheck) - Long-running OWASP project, focused on identifying known vulnerabilities in declared dependencies. Sources its data from NVD directly.
  - [GitHub Action](https://github.com/dependency-check/Dependency-Check_Action) | [Documentation](https://jeremylong.github.io/DependencyCheck/)
  - **Heads up**: since 2024 NVD aggressively rate-limits unauthenticated clients, so without an `NVD_API_KEY` ([request one here](https://nvd.nist.gov/developers/request-an-api-key)) Dependency Check effectively returns 0 findings — a silent false-pass that defeats the point of running it. If you don't want to manage an external key, prefer osv-scanner above.

## Learning Objectives

By the end of this module, you will:
- Understand the difference between SAST and SCA
- Learn to configure and run security scanners
- Understand common vulnerability patterns
- Learn to integrate security scanning into CI/CD

## Security Checklist

- [ ] SAST integrated into CI/CD pipeline
- [ ] SCA scanning for all dependencies
- [ ] Regular dependency updates
- [ ] Security hotspots addressed
- [ ] False positive management
- [ ] Security metrics tracking
