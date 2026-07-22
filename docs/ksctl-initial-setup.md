---
title: Initial Appliance Setup
parent: Automation with KSCTL
nav_order: 2
---

# Initial Appliance Setup
{: .no_toc }

This section covers the basics of appliance configuration using `ksctl`.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

{: .warning }
> **Initial IP address configuration:**
>
> - The appliance boots with a DHCP IP address by default.
>
> A static IP may be configured using the following documentation:
>
> - Using `nmcli`: <https://thalesdocs.com/ctp/cm/latest/get_started/deployment/network-configuration/network-configuration-planning/index.html>
> - Using `cloud-init`: <https://thalesdocs.com/ctp/cm/latest/get_started/deployment/virtual-deployment/cloud-init-deploy/index.html>

## SSH Key for KSADMIN

One of the first tasks to configure a new CipherTrust appliance is setting the
SSH key used to authenticate to the `ksadmin` shell login. This login is used to
apply upgrades to the appliance and to initiate Support sessions on the
appliance as root.

The SSH key is generated externally using **`ssh-keygen`** on Linux or
**PuTTYgen** on Windows.

{: .warning }
> The CM web GUI will not allow logins until this key has been set.

{: .note }
> Store the SSH key securely in a privileged access solution such as CyberArk.

Add the SSH key:

```bash
ksctl ssh keys add --key 'ssh-rsa AA...'
```

## Built-In Web Admin Password

The default web GUI `admin` password must be changed.

```bash
ksctl changepw --url=https://<ip-or-host> --user admin --password <new password>
```

## Node Name

A unique system name may be assigned to identify different nodes in a
CipherTrust cluster. This is not the network host name, but is useful in a large
distributed cluster environment.

{: .note }
> Define a logical appliance node naming convention.

```bash
ksctl system info set --name APAC-Prod1
```

## Licenses

At install, the appliance is configured as a free **Community Edition**
appliance. This is restricted to the **key management**, **Data Protection
Gateway**, and **CTE for Kubernetes** use cases.

### Activate 90 Day Trial License

For testing purposes, the full capabilities of CipherTrust may be unlocked with
unlimited connectors for a 90-day trial period (30 days for Data Discovery).

```bash
ksctl licensing trials activate --id "CipherTrust Manager Full Trial"
```

### Get Lock Codes for Purchased License Activation

For licensed production systems, the licenses must be associated to a unique
**lock code**. This code is used to activate licenses on the **Licensing
Customer Portal**.

{: .note }
> **Licensing Customer Portal:** <https://dplicense.prod.sentinelcloud.com/ecp/auth/login>
>
> **Licensing Documentation:** <https://thalesdocs.com/ctp/cm/latest/get_started/license/>

```bash
ksctl licensing lockdata
```

This returns both the appliance lock code (virtual only — physical appliances
have built-in appliance licenses), as well as the cluster-wide connector lock
code.

Example:

```json
{
   "code": "*1FW KSPH 86WH HJJV",
   "cluster_code": "*17Z QZK5 CMCZ Z7F9"
}
```

To extract just the **connector lock code**, you can use `jq`:

```bash
ksctl licensing lockdata | jq .cluster_code
```

### Install Activated Licenses

Once purchased licenses have been activated on the **Customer Portal**, a license
string is generated that must be installed on the CM cluster.

{: .warning }
> This step requires manual interaction with the Customer License Portal.
> Activate a Trial License to complete the full configuration via automation and
> install the purchased licenses after setup is complete.

Virtual appliance license:

```bash
ksctl licensing licenses add --license "16 Virtual_KeySecure Ni LONG NORMAL STANDALONE..."
```

Connector license:

```bash
ksctl licensing licenses add --bind-type=cluster --license "17 TransparentEncryption Ni LONG NORMAL STANDALONE EXCL 5_CLIENTS..."
```

{: .warning }
> Always copy the license string from the **`lservrc`** file contained in the
> **`license.zip`** email attachment using a text editor. Copying the string
> from the email body or the web page may leave hidden characters that will
> result in an invalid license string.

## Login Banner

Most corporate environments require login banners for compliance reasons.

For short messages:

```bash
ksctl banners update --name pre-auth --message 'Authorized Personnel Only'
```

For longer messages, you can load a pre-defined message file:

```bash
ksctl banners update --name pre-auth --file <banner-message-file>
```

{: .tip }
> **Best practice:** Store a standard, legal-approved login banner message in a
> file and re-use it for each configured appliance.

## NTP Servers

Configure each appliance to sync with a standard time source to avoid
potentially obscure and difficult to troubleshoot errors.

```bash
ksctl ntp servers add --host time.nist.gov
```

For authenticated NTP:

```bash
ksctl ntp servers add --host ntp-b.nist.gov --key secret
```

## Password Policy

Set the appliance password policy to match corporate standards.

{: .note }
> See ThalesDocs for detailed descriptions of the policy options:
> <https://thalesdocs.com/ctp/cm/2.9/admin/cm_admin/password-policy/index.html>

```bash
ksctl users pwdpolicy update --minlength 8 --maxlength 30 --minupper 1 --minlower 1 --minother 1 --mindig 1
```

## Appliance Admins

Add additional named web admin accounts for use by the appliance
administrators. These may be their primary account or a backup in case remote
authentication via LDAP or OIDC fails.

{: .tip }
> **Best Practice:**
> - Use only named admin accounts and not the built-in `admin` account.
> - Set the `admin` user to a strong random password and store it securely in a
>   password management system.

Create the user:

```bash
ksctl users create -n <name> -p <password> -e <email>
```

Example output:

```text
ksctl users create -n my-admin -p Thales123! -e my-admin@thales.lab

{
        "created_at": "2022-08-24T20:03:40.841964Z",
        "email": "my-admin@thales.lab",
        "last_login": null,
        "logins_count": 0,
        "name": "my-admin",
        "nickname": "my-admin",
        "updated_at": "2022-08-24T20:03:40.841964Z",
        "user_id": "local|c4b13478-4dc8-47ed-b119-20a454bcf883",
        "username": "my-admin",
        "failed_logins_count": 0,
        "account_lockout_at": null,
        "failed_logins_initial_attempt_at": null,
        "last_failed_login_at": null,
        "password_changed_at": "2022-08-24T20:03:40.840114Z",
        "password_change_required": false,
        "certificate_subject_dn": "",
        "enable_cert_auth": false,
        "auth_domain": "00000000-0000-0000-0000-000000000000",
        "login_flags": {
                "prevent_ui_login": false
        }
}
```

To add the user to groups so that it has permissions within the appliance, the
`user_id` field must be extracted from the command output.

```bash
ksctl users create -n myadmin -p Thales123! -e my-admin@thales.lab | jq .user_id
```

```text
"local|c4b13478-4dc8-47ed-b119-20a454bcf883"
```

If you need to look up the user ID for an existing user:

```bash
ksctl users list | jq -r '.resources[] | select(.name == "myadmin") | .user_id'
```

Assign the new user to the `admin` group:

```bash
ksctl groups adduser -n admin -u "local|c4b13478-4dc8-47ed-b119-20a454bcf883"
```

Force the user to change password on login:

```bash
ksctl users modify -i "local|c4b13478-4dc8-47ed-b119-20a454bcf883" -r true
```

{: .tip }
> **Best Practice:**
> - Administrative accounts should be assigned the minimum group memberships
>   required to perform their assigned role.
> - Covering separation of duties is beyond the scope of this document.

## Enable Quorum Policy

Use the quorum policy to enforce split responsibility for sensitive key
operations.

{: .note }
> See ThalesDocs for detailed descriptions of the policy options:
> <https://thalesdocs.com/ctp/cm/2.9/admin/cm_admin/password-policy/index.html>

{: .warning }
> In the source *Automation with KSCTL* guide, the command block printed under
> this section duplicates the [Password Policy](#password-policy) example above
> (a copy/paste artifact). Refer to the ThalesDocs link above and
> `ksctl quorum --help` for the correct quorum policy syntax for your CM version.
