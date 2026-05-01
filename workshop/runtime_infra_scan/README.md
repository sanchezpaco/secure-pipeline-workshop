# Runtime Infrastructure Scan

This workshop module focuses on scanning the deployed (runtime) infrastructure for vulnerabilities and misconfigurations that may not be detectable through static analysis of code or configuration files.

## Why is Runtime Infrastructure Scanning Important?

While IaC scans catch many issues pre-deployment, runtime infrastructure scanning is crucial because some vulnerabilities or misconfigurations only manifest once resources are live. This could be due to manual changes, dynamic variables, complex interactions, or even a bug in your IaC provider that end up deploying resources in a way that is not intended.

Continuous runtime scanning helps ensure your infrastructure remains secure after deployment and as it evolves.


## Common Runtime Infrastructure Issues

Many of these issues are shared with the [IaC Security](../iac_scan/) module.

### Access Control
- **Overly Permissive IAM Policies** – wildcard privileges and never-used rights that violate least-privilege.
- **Publicly Accessible Resources** – buckets, APIs, or DBs left open to the internet.
- **Missing Authentication Controls** – services deployed with no auth/MFA, allowing unauthenticated calls.
- **Default or Weak Credentials** – reused passwords and default logins vulnerable to brute-force.

### Encryption
- **Unencrypted Storage at Rest** – plaintext data in S3, EBS, RDS, state files.
- **Unencrypted Backups & Snapshots** – archives stored without server- or client-side encryption.
- **Missing TLS/In-Transit Encryption** – APIs or internal links still on plain HTTP or legacy protocols.
- **Weak or Outdated Cipher Suites** – obsolete algorithms or short keys still in use.

### Network Security
- **Open Security Groups (0.0.0.0/0)** – internet-wide access to SSH, RDP, or high-risk custom ports.
- **Public-Subnet Exposure** – instances or containers with public IPs sitting in public subnets.
- **Mis-scoped Load Balancers/Endpoints** – “internal” services accidentally reachable from the internet.

### Compliance & Governance
- **Insufficient Logging & Audit Trails** – generating blind spots for forensics and incident response.
- **Lack of Continuous Monitoring/Alerts** – misconfigurations persist until breach or audit.
- **Missing Resource Tagging** – untagged assets break cost, ownership, and policy enforcement.
- **Improper Backup Retention/Encryption** – backups stored unencrypted or outside mandated retention windows.

### Differences from IaC Scan
- **Drift** – resources that are not managed by IaC, such as manual changes to resources or manual deployments.

## Tools Used in This Module

- [**Prowler**](https://github.com/prowler-cloud/prowler) - Security best practices assessments for multiple cloud providers (AWS, Azure, GCP, M365, GitHub and more)
  - Can also be used for continuous monitoring and remediation, but we focus on the assessment part
- [**Steampipe**](https://github.com/turbot/steampipe) - SQL-based infrastructure analysis and compliance checking for cloud platforms
  - Uses SQL queries to analyze cloud resources and run security compliance checks across multiple providers

## Learning Objectives

By the end of this module, you will:
- Understand the importance of runtime infrastructure scanning
- Learn to use automated tools to assess live cloud environments


## Security Checklist

- [ ] All systems and services are up to date with security patches
- [ ] No unnecessary open ports or services
- [ ] Strong authentication and access controls in place
- [ ] Security groups/firewalls follow least privilege
- [ ] Encryption enabled for data at rest and in transit
- [ ] No sensitive resources publicly exposed
- [ ] Monitoring and logging enabled for all critical resources
- [ ] Regular runtime scans scheduled and reviewed

## Solutions (spoilers — open only when stuck)

> Unlike the other modules, this one has **no planted bait** to fix. Runtime scans target a *live* AWS account, so the findings depend entirely on whose infrastructure the job assumes into. During the workshop the role points at the maintainers' sandbox account, where you have no permissions to remediate anything — the job is expected to pass green even if Prowler reports findings (note the `-z` flag in the snippet, which suppresses the failing exit code on findings so the pipeline doesn't block on issues you can't fix).

<details>
<summary><b>Prowler</b> — what to expect during the workshop</summary>

**What you'll see**: the `Run Prowler` step assumes the workshop AWS role and scans the `iam` and `s3` services. The HTML report is uploaded as the `prowler-html-report` artifact — download it from the run summary to browse findings grouped by service, severity, and check ID.

**Why nothing is failing**: `prowler aws ... -z` (the "zero exit code" flag) is intentional here. Without it, *any* finding would fail the job, and you'd be staring at issues in someone else's account with no way to fix them. The point of this step in the workshop is to **wire the integration** (OIDC auth, role assumption, artifact upload), not to remediate.

**Reading the report**: open `prowler-html-report/*.html` and look for high-severity items in `iam` (overly permissive policies, unused credentials, root usage) and `s3` (public buckets, unencrypted storage, missing logging). These are the categories you'd triage first on a real account.

</details>

<details>
<summary><b>Running this on your own AWS account</b> — what changes</summary>

When you adopt this job in a real repo, the workflow itself doesn't need to change — only the AWS side and how you treat findings:

1. **Provision an IAM role** in your account with `SecurityAudit` + `ViewOnlyAccess`, plus a trust policy that lets GitHub Actions OIDC (`token.actions.githubusercontent.com`) assume it from your repo.
2. **Set the secrets/vars** in GitHub: `AWS_IAM_ROLE_ARN` (secret), `AWS_REGION` and `AWS_IAM_ROLE_SESSION_DURATION` (variables).
3. **Decide your failure policy**: drop `-z` from the `prowler` command if you want the pipeline to fail on findings, or keep it and gate on severity (e.g. `--severity critical high`) so only the worst issues block merges.
4. **Scope the scan**: `--service iam s3` is fine for a demo, but for real use widen it (`ec2`, `rds`, `kms`, `cloudtrail`, …) or run a full `prowler aws` on a schedule and a narrower scan per PR.
5. **Triage the findings**: most checks map to a specific resource and a remediation hint. Fix the misconfiguration in your IaC (back to module 5) so the next run comes back clean — runtime scans are most useful when paired with IaC scans to catch drift.

</details>

## References
- [Prowler's State of Cloud Security Report 2025](https://prowler.com/blog/cloud-security-report-2025/)
- [Top cloud misconfigurations: A CSPM perspective](https://sysdig.com/blog/top-cloud-misconfigurations/)