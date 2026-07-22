---
title: Connecting Endpoints
nav_order: 7
---

# Connecting Endpoints (KMIP, CTE, CTE-U & more)
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

A key manager is only useful once things consume its keys. This page summarizes
how to connect the main endpoint types to your freshly installed CipherTrust
Manager. These are integration overviews to get you oriented — each connector has
a full Thales deployment guide with product-specific detail.

{: .note }
> Most connectors are licensed features. In **Community Edition** you can explore
> the workflow; production use of CTE, CTE-U, KMIP at scale, and application
> connectors requires the appropriate license applied to the appliance.

## KMIP clients

**KMIP (Key Management Interoperability Protocol)** is the OASIS standard that
lets third-party products request and use keys from CM with no proprietary agent.
Storage arrays, backup software, databases, and VMware vSphere/vSAN encryption all
speak KMIP.

Typical flow:

1. In the web console, add a **KMIP-enabled interface** (listening on TCP
   **5696**) if one isn't already present.
2. Establish trust: upload the client's CA certificate to CM and register the
   client so CM will accept its certificate.
3. Create a **KMIP client/registration token** or certificate for the endpoint.
4. On the endpoint, configure CM as the KMIP key server: host = appliance IP,
   port = 5696, and present the client certificate.

`ksctl` equivalents exist for scripting, e.g. listing the interfaces:

```bash
ksctl interfaces list
```

**What this does:** stands up a mutually-authenticated TLS channel on 5696 over
which the endpoint asks CM to create, fetch, and manage keys using standard KMIP
operations — so the array/backup product encrypts data while CM owns the keys.

## CTE — CipherTrust Transparent Encryption

**CTE** is a **kernel-level agent** that transparently encrypts files and volumes
on a host and enforces access policies (which users/processes may see cleartext),
with keys and policies coming from CM. Applications keep running unmodified.

Typical flow:

1. In CM, create a **registration token** for CTE clients (web console → CTE, or
   `ksctl`).
2. Install the CTE agent on the host (Linux/Windows/AIX) and register it to CM
   using the appliance address and token:

   ```bash
   # On the protected host, after installing the CTE agent
   register_host --primary <ciphertrust-manager-ip> --regtoken <token>
   ```

3. In CM, define an **encryption key**, a **policy** (who/what may access
   plaintext), and a **GuardPoint** (the path/volume to protect).
4. Apply the GuardPoint; CTE begins transparently encrypting data at that path.

**What this does:** the agent intercepts I/O in the kernel and applies
encryption + access control based on policy fetched from CM, so data at rest is
protected and only authorized identities ever see cleartext — transparently to
the application.

## CTE-U — CipherTrust Transparent Encryption UserSpace

**CTE-U** does the same job as CTE but runs in **userspace** using Linux **FUSE**
(Filesystem in Userspace) — with **no kernel module**. That makes it ideal for
the many Linux distributions and, especially, for **containers and Kubernetes**,
and it lets you patch the OS without rebuilding a kernel module.

Typical flow:

1. In CM, create a **registration token** (as for CTE).
2. Install the **CTE-U** agent on the Linux host or into your container image.
3. Register the agent to CM with the appliance address and token.
4. Define keys, policies, and GuardPoints in CM exactly as for CTE.

**What this does:** provides the same policy-based transparent encryption and
access control as CTE, but through a userspace filesystem layer — trading a small
amount of performance for far simpler deployment and container/K8s friendliness.

{: .tip }
> **CTE vs CTE-U in one line:** choose **CTE** (kernel) for maximum performance on
> supported OS/kernel combinations; choose **CTE-U** (userspace/FUSE) for broad
> Linux coverage, containers/Kubernetes, and patch-without-rebuild simplicity.

## Application-layer encryption & tokenization

For encrypting or tokenizing specific fields inside an application, CM exposes
developer interfaces: **REST**, **PKCS#11**, **JCE (Java)**, **.NET**, and the
legacy **NAE-XML** protocol (TCP **9000**). Developers call CM to encrypt,
decrypt, or tokenize data at the field level while the keys remain in CM.

**What this does:** puts protection at the point where data is created, so
sensitive fields (card numbers, PII) are encrypted/tokenized in the app tier and
the keys never live in the application.

## Database TDE

CM can hold the **Transparent Data Encryption** master key for **Microsoft SQL
Server**, **Oracle**, and other databases via the appropriate connector/EKM
provider. The database encrypts its own data files while CM centrally manages and
protects the TDE master key.

---

Next: [SIEM Integration (Splunk) →]({% link docs/siem-splunk.md %})
