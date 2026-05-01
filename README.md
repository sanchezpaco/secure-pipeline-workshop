# Secure Pipeline Workshop

Welcome to the "Secure Pipeline" workshop! This hands-on workshop teaches you how to build a comprehensive security-focused CI/CD pipeline with multiple layers of security scanning and best practices.

## 🏗️ Workshop Structure

The workshop is organized into different modules, each focusing on a specific aspect of pipeline security:

### 📁 Repository Structure

```
├── .github/workflows/               # GitHub Actions workflows
├── code/                            # Sample vulnerable application
├── infra/                           # Terraform infrastructure
└── workshop/                        # Workshop modules and documentation
```

## 📚 Workshop Modules

> 🐦‍🔥 New here? Start with the [workshop introduction](workshop/) for context on shift-left and CI/CD/CS.

### 1. 🛡️ [Pipeline Security Scan](workshop/pipeline_scan/)
Detect insecure GitHub Actions patterns: unpinned actions, excessive permissions, untrusted authors.

### 2. 🔬 [Code Security Analysis](workshop/code_scan/)
Find vulnerabilities in your application code (SAST) and in its dependencies (SCA).

### 3. 🔐 [Secrets Scan](workshop/secrets_scan/)
Detect and prevent exposure of credentials and sensitive information.

### 4. 🐳 [Container Security Scanning](workshop/container_scan/)
Scan Docker images for vulnerabilities and misconfigurations.

### 5. 🏗️ [Infrastructure as Code (IaC) Security Scan](workshop/iac_scan/)
Analyze Terraform and other IaC definitions for security issues before they are deployed.

### 6. ☁️ [Runtime Infrastructure Scan (live cloud)](workshop/runtime_infra_scan/)
Scan a live AWS account for misconfigurations and drift that static IaC scans can't catch.

### 7. 🤖 [AI Security Analysis](workshop/ai_scan/) *(optional)*
Leverage AI to perform comprehensive, context-aware security reviews of pull requests.


## 🚀 Getting Started

### Prerequisites
- GitHub Account to fork the repository
- Basic knowledge of CI/CD concepts
- Familiarity with containers and cloud infrastructure concepts

> [!TIP]
> While this workshop uses GitHub Actions, most of the skills and best practices you learn can be applied to any CI/CD platform.

### Workshop Flow
1. **Fork this repository** to your GitHub account
2. **Follow each module** in the workshop directory
3. **Run the workflows** and observe the security findings
4. **Learn to fix** the identified vulnerabilities
5. **Implement security best practices**

### Workshop Goal
The idea of this workshop is to demonstrate how to build a "perfect" (secure and practical) CI/CD pipeline using open-source tools (OSS).

**The goal is inspirational, not prescriptive.** We do not want you to copy these examples, but to understand the principles and identify the modular components you can adapt to implement in your own environment.

### 🎓 Learning Outcomes

By completing this workshop, you will:
- Understand the importance of shift-left security
- Learn the key stages of a secure pipeline:
  - Pipeline Security
  - Static and Dynamic Code Analysis
  - Secrets Detection
  - Container Security
  - Infrastructure as Code (IaC) Security
  - Runtime Infrastructure Security
- Know relevant OSS tools for each stage
- Grasp the principles needed to start building or improving your own secure CI/CD process

### Out of Scope (What this workshop is NOT)
- Deep dives into specific development workflows (e.g., Gitflow vs. Trunk-based)
- Focus on a specific application technology stack (language/framework agnostic where possible)
- A definitive statement on the "best" tools (alternatives will be mentioned for key steps)


---

## 🤝 Contributing

This workshop is designed to be continuously improved. Feel free to:
- Report issues or suggest improvements
- Add new security scenarios
- Contribute additional tool integrations
- Share your workshop experience

## 📄 License

This workshop is provided under the MIT License for educational purposes.

---

**Ready to build the perfect secure pipeline? Start [here](workshop/)! 🚀**
