# CipherTrust Manager — Installation & Setup Guide

A step-by-step, command-level walkthrough for deploying and performing the
initial configuration of a **Thales CipherTrust Manager** virtual appliance,
published as a GitHub Pages site using the [Just the Docs](https://just-the-docs.com)
theme.

The guide covers:

- What CipherTrust Manager is — private/public cloud use cases, key management,
  root of trust with an HSM, and endpoint compatibility (KMIP, CTE, CTE-U).
- **Private cloud** deployment on VMware vSphere/ESXi (OVA).
- **Public cloud** deployment on AWS (Marketplace/AMI).
- First-boot and initial configuration (console, networking, web console,
  `ksctl` CLI).
- Connecting endpoints via KMIP, CTE, and CTE-U.
- SIEM integration, including **Splunk**.

## Local preview

```bash
bundle install
bundle exec jekyll serve
# open http://127.0.0.1:4000
```

## Publishing

Pushing to the `main` branch triggers the GitHub Actions workflow in
`.github/workflows/pages.yml`, which builds the site and deploys it to GitHub
Pages. Enable Pages under **Settings → Pages → Build and deployment → Source:
GitHub Actions**.

---

> **Disclaimer:** This is an independent technical walkthrough based on publicly
> available Thales documentation. It is not affiliated with or endorsed by
> Thales. Always verify each step against the official product documentation for
> your specific CipherTrust Manager version before using it in production.
