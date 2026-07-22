---
title: Protocol Interfaces
parent: Automation with KSCTL
nav_order: 8
---

# Configure Protocol Interfaces
{: .no_toc }

CipherTrust allows the creation of additional interfaces for use cases such as
**KMIP** or **NAE-XML**. In this example, we will create a new NAE-XML interface
that **CipherTrust Application Data Protection (CADP)** will interact with, and
set TLS and client certificate authentication requirements.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Add NAE-XML Interface

In this example, a new NAE-XML interface will be set up on TCP port 9001, with a
minimum TLS version of 1.3, and authentication requirements set to:

- verify client cert
- user name taken from client cert
- auth request is optional

```bash
ksctl interfaces create --name nae-secure --type nae --port 9001 --mode tls-cert-pw-opt --tlsversion tls_1_3
```
