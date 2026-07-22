---
title: Authentication
parent: Automation with KSCTL
nav_order: 3
---

# Configure Authentication
{: .no_toc }

A typical enterprise environment will use external authentication for
CipherTrust platform users.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## LDAP Directory

An Active Directory connection allows user accounts to be managed outside of
CipherTrust. Below is a typical configuration for a Windows Server 2019 domain
controller. The specifics will vary.

```bash
ksctl connections create ldap -n "DC1" -u "ldap://dc1.thales.lab" -i "sAMAccountName" -r "OU=Demo,DC=thales,DC=lab" -b "CN=cmbind,OU=Demo,DC=thales,DC=lab" -p "Thales123!" -w "OU=Demo,DC=thales,DC=lab" -e "sAMAccountName" -t "(objectClass=group)" -o "cn" -m "member"
```

{: .note }
> **For LDAPS:**
> - The domain controller's root CA must be a trusted external CA.
> - The root CA cert is provided when creating the connection.
> - Add `--root-ca-file` or `--insecure-skip-verify`.

## LDAP Group Mapping

LDAP group mapping allows Active Directory users to inherit CipherTrust rights
membership in an AD group that is logically mapped to a CipherTrust group.

```bash
ksctl groupmaps create -c "DC1" -n "cmadmins" -k "admin"
```

## OIDC Connection

Open ID Connect allows an external identity provider to authenticate CipherTrust
users. This is the typical method of adding multi-factor authentication to
logins.

```bash
ksctl connections create oidc --name "SafeNet Trusted Access" --client-id <my client id> --redirect-uri https://<my-cm-host>/api/v1/auth/oidc-callback --discovery-uri https://idp.us.safenetid.com/auth/realms/QBBB6BI5JA-STA/.well-known/openid-configuration
```
