---
title: Public Cloud — AWS
parent: Deployment
nav_order: 2
---

# Public Cloud Deployment — AWS
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

This path launches CipherTrust Manager as an **EC2 instance** from its AWS
image — the public-cloud installation, where AWS runs the infrastructure while
you retain full control of the encryption keys. The post-boot configuration is
identical to the VMware path; only the "create the VM" steps differ.

**Prerequisites for this path:** an AWS account with EC2 permissions, a chosen
**VPC/subnet**, an EC2 **key pair** (OpenSSH/RSA), and access to the CM **AMI**
(via AWS Marketplace or provisioned through the Thales Customer Support Portal).
Minimum sizing: **2 vCPU, 16 GB RAM** — an instance type such as `t3.xlarge` or
`m5.xlarge` meets this — plus a **100 GB** (production) EBS system volume.

---

## Step 1 — Obtain the AMI

Get the CipherTrust Manager AMI into your account one of two ways:

- **AWS Marketplace** — subscribe to the *CipherTrust Data Security Platform* /
  *CipherTrust Manager* listing, which makes the AMI launchable in your region.
- **Thales Customer Support Portal** — request provisioning under **Tools →
  Provisioning Requests**; Thales shares an AMI ID to your AWS account.

**What this does:** places a private, launchable machine image in your account so
EC2 can boot it. Confirm you're operating in the **AWS region** where the AMI is
available.

---

## Step 2 — Launch the EC2 instance

In the EC2 console, choose **Launch instance**, then:

1. **Name** the instance (e.g. `ciphertrust-manager-prod`).
2. **Application and OS Images:** select the CM AMI (My AMIs, or from your
   Marketplace subscription).
3. **Instance type:** pick one that meets 2 vCPU / 16 GB RAM — e.g.
   `m5.xlarge`.
4. **Key pair:** select an existing OpenSSH/RSA key pair or create one — you'll
   use it to SSH in as `ksadmin`.
5. **Network settings:** choose the **VPC** and **subnet**; enable a public IP
   only if you truly need direct access (a bastion or VPN is safer).
6. **Configure storage:** set the root EBS volume to **100 GiB** for production
   (30 GiB is acceptable for a short-lived lab).
7. **Security group:** see Step 3.
8. **Advanced → User data:** optionally supply a cloud-init payload to pre-seed
   configuration (same idea as the VMware vApp option).
9. **Launch instance.**

**What this does:** EC2 provisions a VM from the AMI with your CPU/RAM, attaches
the EBS system volume, injects your key pair's public key for `ksadmin`, and
places the instance on your chosen subnet.

---

## Step 3 — Set the security group

Create/attach a security group that opens only what you need. Scope every rule to
known source ranges — never `0.0.0.0/0` for management ports.

| Type | Port | Source | Purpose |
|:-----|:-----|:-------|:--------|
| SSH | 22 | your admin CIDR / bastion | `ksadmin` console management |
| HTTPS | 443 | admin + client CIDRs | Web console, REST, `ksctl`, CTE registration |
| Custom TCP | 5696 | KMIP client CIDRs | KMIP (only if used) |
| Custom TCP | 9000 | app server CIDRs | NAE-XML apps (only if used) |

**What this does:** the security group is AWS's stateful firewall for the
instance. The minimum to bring CM online and reach the console is **22 and 443**;
add 5696/9000 only when you register those client types.

{: .warning }
> Exposing **22** or **443** to the whole internet is a common and serious
> mistake for a key manager. Restrict to your corporate ranges, a bastion, or a
> VPN, and prefer private subnets.

---

## Step 4 — Connect as `ksadmin` (SSH)

Once the instance shows **running** and passes status checks, SSH in as
**`ksadmin`** using your key pair and the instance's public DNS or IP:

```bash
ssh -i /path/to/your-key.pem ksadmin@ec2-203-0-113-25.compute-1.amazonaws.com
```

Sample first-connection output:

```
The authenticity of host 'ec2-203-0-113-25...' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added 'ec2-203-0-113-25...' (ED25519) to the list of known hosts.

CipherTrust Manager
ksadmin@ciphertrust:~$
```

**What this does:** logs you into the appliance OS as the `ksadmin` system
account, authenticated by your EC2 key pair (no password). In AWS the instance
usually already has an IP from the subnet, so you typically don't need the manual
`nmcli` static-IP step from the VMware path — though you can still set the
hostname and other system options here (see
[Initial Configuration]({% link docs/initial-configuration.md %})).

{: .tip }
> On Windows, use PuTTY: convert the `.pem` to `.ppk` with PuTTYgen, set the
> username to `ksadmin`, and load the key under *Connection → SSH → Auth*.

---

## Step 5 — Reach the web console

In a browser, go to the instance's address over HTTPS:

```
https://<instance-public-or-private-IP>
```

Accept the self-signed-certificate warning (expected for now) and log in to the
CipherTrust Manager application:

- **Username:** `admin`
- **Password:** `admin`

You'll be forced to set a new password immediately.

**What this does:** confirms the appliance is serving the web console over 443 and
hands you to the application login. For a private-subnet instance, reach this
through your bastion/VPN rather than a public IP.

{: .warning }
> As with any deployment, the very next actions are to rotate the default
> `admin` password and harden access — proceed to
> [Initial Configuration]({% link docs/initial-configuration.md %}).

---

Next: [Initial Configuration →]({% link docs/initial-configuration.md %})
