---
title: Prerequisites
nav_order: 4
---

# Prerequisites
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Gather everything below before you start. Having the image, sizing, network
details, and access ready turns the deployment into a smooth 20-minute exercise
instead of a stop-and-start one.

## 1. The CipherTrust Manager image

You download the appliance image from Thales, tied to your entitlement:

- **VMware / private cloud** — an **OVA** template file (for example
  `CipherTrust-Manager-<version>.ova`).
- **AWS / public cloud** — an **AMI**, either subscribed through **AWS
  Marketplace** or provisioned to your AWS account through the **Thales Customer
  Support Portal** (Tools → Provisioning Requests).

Download from the Thales Customer Support Portal (support.thalesgroup.com) using
your account. The free **Community Edition** is included and is a good way to
learn the workflow before applying a license.

{: .note }
> This guide uses CipherTrust Manager 2.x and the **k170v** virtual-appliance
> sizing. Newer releases (2.19, 2.23 LTS, and later) follow the same install
> flow; confirm exact figures in the release notes for your version.

## 2. Sizing (minimum resources)

The k170v virtual appliance's minimum requirements are the same across
hypervisors and clouds:

| Resource | Evaluation | Production |
|:---------|:-----------|:----------|
| vCPUs | 2 | 2 (more for load) |
| Memory | 16 GB | 16 GB |
| System disk | 50 GB | **100 GB** |
| NICs | 1 | 1 (or more) |
| Hypervisor | VMware vSphere/ESXi **5.1+** | — |

{: .tip }
> Give production appliances the full **100 GB** system volume. The 50 GB option
> is intended for short-lived evaluation only, and resizing later is more work
> than provisioning correctly up front.

## 3. Network planning

Decide these before first boot so the console session goes quickly:

- A **static IP address**, **netmask** (in CIDR form, e.g. `/22`), and
  **default gateway** for the appliance.
- One or more **DNS servers**.
- An **NTP server** (accurate time is important for certificates and for
  correlating audit logs in your SIEM).
- A **hostname** / FQDN and a matching DNS record.
- **Firewall / security-group rules** opening only the ports you need:

| Port | Protocol | Purpose |
|:-----|:---------|:--------|
| 22 | TCP | SSH console management (`ksadmin`) |
| 443 | TCP | Web console, REST API, `ksctl`, CTE/CTE-U registration |
| 5696 | TCP | KMIP clients (only if you use KMIP) |
| 9000 | TCP | NAE-XML applications (only if you use them) |
| 514 / 5514 / 6514 | UDP/TCP | Outbound syslog to your SIEM (only if used) |

## 4. Access and tools

- **VMware:** vSphere Client access to an ESXi host or vCenter with permission to
  deploy OVF templates; optionally the **VMware OVF Tool** (`ovftool`) if your
  client can't import an OVA directly.
- **AWS:** an account with EC2 permissions, a chosen **VPC/subnet**, and an EC2
  **key pair** (OpenSSH/RSA format).
- **An SSH client** — native `ssh` on macOS/Linux, or PuTTY on Windows — to log
  in as `ksadmin`.
- **A modern web browser** to reach the CM web console at `https://<appliance-ip>`.
- Optional: the **`ksctl`** CLI (downloadable from the appliance) on your admin
  workstation for scripting.

## 5. Credentials you'll use

You will encounter two identities during setup — don't confuse them:

| Identity | Where it's used | Initial value |
|:---------|:----------------|:--------------|
| **`ksadmin`** | The OS/console account, over SSH or the VM console. Manages networking and system config. | Set an SSH key / password at first console login |
| **`admin`** | The CipherTrust *application* administrator, in the **web console** and `ksctl`. | Username `admin`, password `admin` — **must be changed at first login** |

{: .warning }
> The application administrator ships as **`admin` / `admin`**. Changing this
> password (and replacing the default SSH key) is the very first thing you do
> after the appliance is reachable. See
> [Initial Configuration]({% link docs/initial-configuration.md %}).

---

Choose your deployment path next:

- [Private Cloud — VMware vSphere/ESXi →]({% link docs/deploy-vmware.md %})
- [Public Cloud — AWS →]({% link docs/deploy-aws.md %})
