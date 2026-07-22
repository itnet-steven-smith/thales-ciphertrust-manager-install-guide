---
title: Private Cloud — VMware vSphere/ESXi
parent: Deployment
nav_order: 1
---

# Private Cloud Deployment — VMware vSphere/ESXi
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

This path deploys the CipherTrust Manager **OVA** onto VMware vSphere/ESXi — the
typical on-premises / private-cloud installation. Each step lists the exact
action or command, sample output where relevant, and an explanation of what it
does.

**Prerequisites for this path:** the CM **OVA** file, vSphere Client access with
permission to deploy OVF templates, and the [network details]({% link docs/prerequisites.md %})
you planned (static IP, gateway, DNS). Minimum sizing: **2 vCPU, 16 GB RAM, 100
GB disk, 1 NIC**, ESXi **5.1+**.

---

## Step 1 — (If needed) Convert the OVA with `ovftool`

Most vSphere Client versions import an `.ova` directly, so you can usually skip
this step. If your client can't, use the **VMware OVF Tool** to unpack the OVA
into its component `.ovf`, `.vmdk`, and `.mf` files:

```bash
ovftool --lax CipherTrust-Manager-2.x.ova CipherTrust-Manager-2.x.ovf
```

Sample output:

```
Opening OVA source: CipherTrust-Manager-2.x.ova
Writing OVF package: CipherTrust-Manager-2.x.ovf
Disk progress: 100%
Completed successfully
```

**What this does:** `ovftool` reads the single-file OVA archive and writes out
the separate descriptor (`.ovf`), virtual disk (`.vmdk`), and manifest (`.mf`).
The `--lax` flag relaxes strict validation so minor schema differences between
tool versions don't block the export. You then point "Deploy OVF Template" at the
`.ovf`.

---

## Step 2 — Deploy the OVF template in vSphere

In the vSphere Client:

1. Right-click the target **host**, **cluster**, or **resource pool** and choose
   **Deploy OVF Template…**
2. In **Source**, select the local `.ova` (or the `.ovf` from Step 1) — or paste
   a URL if the image is hosted.
3. Give the VM a **name** and choose the **inventory folder**.
4. Select the **compute resource** (host/cluster).
5. Review the **details**, then accept sizing: **2 vCPU / 16 GB RAM / 100 GB**.
6. Choose the **datastore** and a disk format (Thin provisioning is fine for
   evaluation; Thick for production performance).
7. Map the OVA network to the correct **port group / VLAN** for the appliance.
8. Click **Finish**.

**What this does:** vSphere reads the OVF descriptor, creates a VM shell with the
CPU/memory/disk the template specifies, imports the virtual disk, and attaches
the VM's NIC to your chosen network. Nothing boots yet — you're assembling the VM.

{: .tip }
> Put the appliance on a management network/VLAN that can reach your DNS, NTP,
> and (later) your HSM and SIEM, and that your KMIP/CTE clients can reach on the
> [ports from the architecture page]({% link docs/architecture.md %}#how-endpoints-connect).

---

## Step 3 — (Optional) Pre-seed configuration with cloud-init

You can inject network settings, the admin SSH key, and disk-encryption choices
at first boot using a **cloud-init** payload placed in the VM's **vApp Options**.
This automates what you'd otherwise type at the console in Step 5.

1. Author a cloud-init `user-data` file (YAML) with your network and SSH-key
   settings.
2. Base64-encode it:

   ```bash
   openssl base64 -in user-data -out user-data.b64
   ```

3. In the VM's **Edit Settings → vApp Options**, paste the base64 string into the
   user-data property, then boot.

**What this does:** cloud-init runs once on first boot and applies the supplied
configuration, so the appliance can come up already networked and hardened.
`openssl base64` encodes the YAML because the vApp property expects base64 text.
After boot you can confirm it ran by checking `/var/log/cloud-init.log`.

{: .note }
> Cloud-init is optional. If you skip it, you'll simply set networking by hand at
> the console in Step 5 — both approaches reach the same place.

---

## Step 4 — Power on and open the console

Power on the VM, then open its console: select the VM → **Summary** tab → **Launch
Web Console** (or **Launch Remote Console**).

**What this does:** the appliance boots its hardened Linux OS and the CipherTrust
services. The console is your out-of-band way in before networking is set — the
equivalent of a crash cart or serial cable.

You'll see the appliance reach a login prompt.

---

## Step 5 — First console login as `ksadmin` and set the IP

Log in at the console as **`ksadmin`** and follow the prompt to create a secure
password (first login only). Then assign a static IP with NetworkManager's
`nmcli`. Replace the interface name (`ens4` here), addresses, and MAC to match
your VM.

```bash
# 1) Create a connection profile bound to the NIC's MAC address
nmcli conn add type ethernet con-name ens4 ifname '' -- \
  ethernet.mac-address 00:50:56:99:3F:55 ipv4.method auto ipv6.method ignore

# 2) Switch it to a static (manual) address, gateway, and DNS
nmcli conn modify ens4 ipv4.method manual \
  ipv4.addresses 10.121.105.18/22 \
  ipv4.gateway 10.121.104.1 \
  ipv4.dns 8.8.8.8,8.8.4.4

# 3) Ignore any DNS handed out by DHCP
nmcli conn modify ens4 ipv4.ignore-auto-dns yes

# 4) Bring the connection up with the new settings
nmcli conn up ens4
```

Verify the address took effect:

```bash
nmcli conn show ens4 | grep IP4.ADDRESS
```

Sample output:

```
IP4.ADDRESS[1]:                         10.121.105.18/22
```

**What each command does:**

- `nmcli conn add … mac-address …` creates a persistent network profile pinned to
  the interface's hardware (MAC) address, so it survives reboots and NIC
  reordering.
- `nmcli conn modify … ipv4.method manual …` replaces DHCP with the static IP,
  gateway, and DNS you planned.
- `ipv4.ignore-auto-dns yes` stops a DHCP server from overriding your DNS entries.
- `nmcli conn up ens4` activates the profile immediately.
- The `grep IP4.ADDRESS` check confirms the interface now holds your static
  address before you rely on it.

{: .tip }
> Find the exact interface name and MAC first with `nmcli device status` and
> `nmcli device show`. Interface names vary (`ens4`, `ens32`, `eth0`) by
> hypervisor and hardware.

---

## Step 6 — Reach the web console

From a browser on the same network, go to:

```
https://10.121.105.18
```

(substitute your appliance's IP). You'll get a TLS warning for the appliance's
self-signed certificate — expected at this stage; you'll replace the certificate
later. Log in to the CipherTrust Manager application with the default
credentials:

- **Username:** `admin`
- **Password:** `admin`

The appliance immediately requires you to set a new password.

**What this does:** confirms the appliance is reachable over HTTPS and hands you
off to the application login. From here, all remaining setup is platform-neutral.

{: .warning }
> Don't stop at "it loads." Go straight to
> [Initial Configuration]({% link docs/initial-configuration.md %}) to rotate the default `admin`
> password, replace the default SSH key, set the hostname/NTP, and (for
> production) establish the [HSM root of trust]({% link docs/architecture.md %}#root-of-trust-with-an-hsm).

---

Next: [Initial Configuration →]({% link docs/initial-configuration.md %})
