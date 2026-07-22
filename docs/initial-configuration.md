---
title: Initial Configuration
nav_order: 6
---

# Initial Configuration
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

At this point the appliance is deployed and reachable at `https://<appliance-ip>`
(from either the [VMware]({% link docs/deploy-vmware.md %}) or [AWS]({% link docs/deploy-aws.md %}) path). Now you
harden it and make it usable: rotate the default credentials, replace the SSH
key, set identity/time, install the `ksctl` CLI, and — for production — establish
the [HSM root of trust]({% link docs/architecture.md %}#root-of-trust-with-an-hsm).

## Step 1 — Change the default administrator password

Log in to the web console as `admin` / `admin`. The appliance forces a new
password on first login. Choose one that meets the policy:

| Rule | Requirement |
|:-----|:------------|
| Length | 8–30 characters |
| Uppercase | at least 1 |
| Lowercase | at least 1 |
| Digit | at least 1 |
| Special character | at least 1 |

**What this does:** removes the single most dangerous default on the box. Until
this is changed, anyone who can reach 443 owns your key manager. You can also do
this from `ksctl` once it's set up (Step 4):

```bash
ksctl users modify -n admin --password '<NewStr0ng!Pass>'
```

{: .tip }
> Store the new `admin` password in your team's secrets manager immediately, and
> create **named administrator accounts** per person rather than sharing `admin`.
> Named accounts make the audit trail meaningful.

## Step 2 — Replace the default SSH public key

Replace the appliance's default SSH key with your own (OpenSSH format) so console
access is tied to keys you control. This is done in the web console under the
system/administration settings (and can be pre-seeded via cloud-init at deploy
time).

**What this does:** ensures that OS-level (`ksadmin`) access uses a key only your
team holds, closing another well-known default.

## Step 3 — Set hostname, time, and DNS (system config)

Over SSH as `ksadmin`, the **`kscfg`** system-configuration utility manages
OS-level identity and networking. Useful commands:

```bash
# Show available command groups (net, system, help)
kscfg --help

# List and inspect network interfaces
kscfg net interfaces list
kscfg net interfaces get -n eth0

# View and set the hostname (default is "ciphertrust")
kscfg system hostname get
kscfg system hostname set -n ctm-prod-01
```

Sample output:

```
$ kscfg system hostname get
ciphertrust
$ kscfg system hostname set -n ctm-prod-01
Hostname updated. Restart services for the change to take full effect.
```

**What each command does:**

- `kscfg --help` lists the tool's command groups.
- `kscfg net interfaces list` / `get` show the current NIC configuration — handy
  to confirm the address you set during deployment.
- `kscfg system hostname get`/`set` read and change the appliance hostname
  (written to `/etc/hostname`); set a meaningful FQDN-style name and add a
  matching DNS record.

{: .note }
> **Time matters.** Configure **NTP** so the appliance clock is accurate.
> Certificates, key validity windows, and — importantly — the timestamps that
> your [SIEM]({% link docs/siem-splunk.md %}) correlates all depend on correct time. Set NTP in the
> web console's network/system settings (or via your cloud-init payload).

{: .warning }
> `kscfg system reset` wipes all data, and `kscfg system factory-reset` (k470/k570)
> reverts to factory state. These are recovery-only, last-resort commands — don't
> run them on a configured appliance.

## Step 4 — Install and log in with `ksctl` (CLI)

`ksctl` is the command-line client for CipherTrust Manager's REST API. It's the
fastest way to script setup and is required for tasks like HSM root-of-trust
configuration. Download the `ksctl` bundle for your OS from the appliance
(available in the web console downloads, or at
`https://<appliance-ip>/downloads/ksctl_images.zip`), unpack it, then configure
and log in:

```bash
# Point ksctl at your appliance and log in as admin
ksctl config set --url https://10.121.105.18 --nosslverify
ksctl login --user admin --password '<NewStr0ng!Pass>'
```

Sample output:

```
{
  "duration": 300000,
  "jwt": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9....",
  "token_type": "Bearer"
}
```

Confirm connectivity by listing users:

```bash
ksctl users list
```

**What this does:** `ksctl config set` records the appliance URL (the
`--nosslverify` flag tolerates the self-signed cert until you install a trusted
one); `ksctl login` authenticates and caches a bearer token so subsequent
commands are authorized. A successful `users list` proves the API path works
end-to-end.

## Step 5 — Install a trusted TLS certificate (recommended)

Replace the appliance's self-signed web certificate with one signed by your
enterprise CA (or a public CA), via the web console's **CA / Interface**
certificate settings for the `web` interface.

**What this does:** removes browser warnings and, more importantly, lets clients
and administrators cryptographically trust they're talking to the real appliance
rather than an impostor.

## Step 6 — Establish the HSM root of trust (production)

For production, anchor the key hierarchy in hardware now — **before** you load
keys or register clients, because the operation resets the appliance. Using
`ksctl` as configured above:

```bash
# Example: attach a Luna Network HSM partition (DESTRUCTIVE — resets CM)
ksctl hsm setup luna \
  --reset \
  --partition-name my_partition \
  --partition-password '<partition-password>' \
  --server '<hsm-ip>' \
  --server-cert-file /path/to/server.pem

# Verify the HSM server is registered
ksctl hsm servers list
```

**What this does:** moves the top of the KEK chain into the HSM so the master
keys live in tamper-resistant hardware (see
[Architecture & Root of Trust]({% link docs/architecture.md %}#root-of-trust-with-an-hsm)). Because
`--reset` wipes the appliance, do this on a fresh install or from a backup.

{: .warning }
> Skipping the HSM is fine for a lab (CM will use a software root of trust), but
> for anything holding real keys, plan the HSM step up front — retrofitting it
> later means another destructive reset and a full key reload.

## Step 7 — Take a backup

Once the appliance is configured the way you want, create a **backup** (web
console → System → Backup, or via `ksctl`) and store it securely. Backups are
encrypted and are your recovery path for everything below the root of trust.

**What this does:** captures configuration, keys, and policies so you can restore
after a failure or rebuild — essential before you start depending on the
appliance.

---

Next: [Connecting Endpoints →]({% link docs/connectors.md %})
