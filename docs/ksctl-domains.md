---
title: Domains
parent: Automation with KSCTL
nav_order: 6
---

# Domains
{: .no_toc }

A typical enterprise environment will have one or more security domains within
CipherTrust to separate administration and key material for different customers,
environments, and use cases.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Create Domain

{: .tip }
> **Best Practice:** Create an admin user for each new domain to support current
> or future separation of duties.

### Create Domain Admin User

In addition to other admin accounts that may need to manage the objects in a
CipherTrust domain, it is best practice to create a dedicated admin for each
domain and grant it full admin rights within that domain.

{: .note }
> The `ksctl domain create` command takes user ID values to assign
> administrators rather than user names. We will need to get the `user_id` of any
> users we wish to assign as admins.

```bash
ksctl users create -n domain1admin -p Thales123! -e domain1admin@thales.lab | jq .user_id
```

```text
"local|e940f36d-6a0b-4857-94f6-b6fb6bf5062e"
```

We may also want the `user_id` value for the `admin` user or a similar
administrative account.

```bash
ksctl users list | jq '.resources[] | select(.name == "admin") | .user_id'
```

```text
"local|003b52e1-b8a8-4b3b-aab4-a9e5d18e5f13"
```

### Create Sub-Domain

This command will create a new CipherTrust domain underneath the `root` domain.
In this model, users will be managed at the root and domain admins will assign
the users to groups within the domain.

```bash
ksctl domains create --name domain1 --admins 'local|e940f36d-6a0b-4857-94f6-b6fb6bf5062e,local|003b52e1-b8a8-4b3b-aab4-a9e5d18e5f13'
```

### Create Delegated Sub-Domain

This command will create a domain with **Allow Subdomain User Management**
enabled. This will allow the domain admin to create domain-specific users and
autonomously control security within the domain, independent of the `root` users
and groups.

For full domain autonomy, the assigned domain admin can create a purely local
admin account and unassign the admin user inherited from the `root` domain.

```bash
ksctl domains create --name domain1 --admins 'local|e940f36d-6a0b-4857-94f6-b6fb6bf5062e' --allow-user-mgmt
```

#### Login to Sub-Domain

```bash
ksctl logout
ksctl login --user domain1admin --password Thales123! --domain domain1
```

#### Add User to Sub-Domain

```bash
ksctl users create --is-domain-user true -n domainlocal -p Thales123! -e domainlocal@thales.lab | jq .user_id
```

```text
"local|141aade0-a50e-4d8b-a5b7-0d615a7b21c2"
```

#### Add Local User to Domain Key Users Group

```bash
ksctl groups adduser -n "Key Users" -u "local|141aade0-a50e-4d8b-a5b7-0d615a7b21c2"
```

## Disable Domain Syslog Forward-to-Parent

By default, a sub-domain will forward logs to the parent domain (typically
`root`) syslog server. For fully autonomous domains, this behavior can be
disabled and a separate syslog connection configured within the domain.

```bash
ksctl domains syslog-redirection disable
```

## Log Back Into Root

Always be aware of what domain you are connected to, as you can authenticate to
one domain and specify a separate domain you have access to for where `ksctl`
will execute the command.

See authorized domains:

```bash
ksctl tokens self-domains | jq .resources[].name
```

```text
"domain1"
"root"
```

Log back into root:

```bash
ksctl logout
ksctl login
```
