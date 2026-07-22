---
title: Automation with KSCTL
nav_order: 10
has_children: true
---

# Automation with KSCTL
{: .no_toc }

CipherTrust Data Security Platform is fully configurable through built-in REST
APIs. Thales provides **`ksctl`**, a command-line tool for interacting with
those APIs without writing custom scripts or code. This section walks through
configuring CipherTrust Manager — from initial boot to common use cases —
entirely from the command line.
{: .fs-6 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What this section covers

This is a command-level reference for automating CipherTrust Manager (CM) with
`ksctl`. Each child page maps to a stage of appliance configuration:

| Page | What it covers |
|:-----|:---------------|
| [Installing KSCTL]({% link docs/ksctl-install.md %}) | Downloading `ksctl`, configuring the connection, multiple environments, and parsing JSON output with `jq` |
| [Initial Appliance Setup]({% link docs/ksctl-initial-setup.md %}) | SSH key, web admin password, node name, licenses, login banner, NTP, password policy, appliance admins, quorum |
| [Authentication]({% link docs/ksctl-authentication.md %}) | LDAP directory, LDAP group mapping, and OIDC connections |
| [Certificate Authorities]({% link docs/ksctl-certificate-authorities.md %}) | Adding trusted external CAs and external subordinate CAs |
| [Logging & Alerts]({% link docs/ksctl-logging.md %}) | Local log DB, Loki retention, syslog forwarding, KMIP/NAE records, and alarm/alert definitions |
| [Domains]({% link docs/ksctl-domains.md %}) | Creating domains, domain admins, sub-domains, delegated sub-domains, and syslog redirection |
| [Cluster]({% link docs/ksctl-cluster.md %}) | Creating a cluster and joining nodes with `cluster fulljoin` |
| [Protocol Interfaces]({% link docs/ksctl-protocol-interfaces.md %}) | Adding a hardened NAE-XML interface |
| [CTE (Appendix A)]({% link docs/ksctl-cte.md %}) | CTE admin, client profiles, client logging, record upload, concise logging, and client syslog |

{: .note }
> This section is not a comprehensive reference for every configuration or use
> case. It provides a foundation for automating CipherTrust administration in
> your environment. For full automation, `ksctl` commands generally require some
> form of scripting to provide logical flow and persistence of values —
> scripting is beyond the scope of this guide.

{: .note }
> For additional details and specific examples, refer to the latest official
> documentation at **[Thales Docs Online](https://thalesdocs.com/ctp/cm/latest/index.html)**.

{: .tip }
> Many `ksctl` command options have both a full expanded form and a single
> character short form. This guide favors the expanded form for clarity — for
> example, `--hsm-connection-id` versus `-c`. Use `ksctl <command> --help` to
> see the syntax and short forms for any command.

---

## Source & provenance

The commands and examples in this section are adapted from Thales'
*CipherTrust Automation — Using KSCTL from the Command Line* guide and validated
against the official [Thales product documentation](https://thalesdocs.com/ctp/cm/latest/index.html).
Exact strings (interface names, ports, UUIDs, license strings, and version
numbers) are illustrative and will differ in your environment — treat them as
templates, not literals, and verify version-specific behavior against the
official docs before production use.
