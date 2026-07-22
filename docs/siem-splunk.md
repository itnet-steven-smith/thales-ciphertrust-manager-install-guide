---
title: SIEM Integration (Splunk)
nav_order: 9
---

# SIEM Integration — Splunk
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

CipherTrust Manager's logs are high-value security telemetry — who used which
key, who changed a policy, who logged in and from where. Forwarding them to a
**SIEM** puts that activity next to the rest of your security data for
correlation, alerting, and retention. This page covers the two flows that matter
most: **CM server/audit logs** and **CTE client access logs**, both into
**Splunk**.

## What you can forward, and where

CipherTrust Manager's **log-forwarding** capability can send records to:

- **Syslog** servers (UDP, TCP, or TLS) — the most common path, and what Splunk
  ingests.
- **Loki** and **Elasticsearch** back ends (for non-Splunk stacks).

For Splunk specifically, Thales publishes the **Thales Security Intelligence**
app on Splunkbase (app ID 5594). It ships the indexes, source types, and
dashboards that turn raw CipherTrust events into searchable, visualized security
data.

{: .note }
> Accurate **NTP** on the appliance (see
> [Initial Configuration]({% link docs/initial-configuration.md %}#step-3--set-hostname-time-and-dns-system-config))
> is a prerequisite for good SIEM data — event correlation falls apart if the
> CipherTrust clock drifts from Splunk's.

---

## Part A — Forward CipherTrust Manager server logs to Splunk

This sends the appliance's own audit and server records to Splunk.

### Step 1 — Install the Thales Security Intelligence app in Splunk

From Splunkbase, install **Thales Security Intelligence** on your Splunk search
head / indexer. Installing it creates the CipherTrust source types (e.g.
`cm-st`) and dashboards.

**What this does:** teaches Splunk how to parse and present CipherTrust events,
and provides ready-made dashboards instead of you building parsing from scratch.

### Step 2 — Create a Splunk index

In Splunk: **Settings → Indexes → New Index**.

- **Index Name:** `cm` (or a name of your choice)
- **App Context:** *Thales Security Intelligence*
- Save.

**What this does:** creates a dedicated bucket to store CipherTrust Manager
events, keeping them organized and separately retainable/searchable.

### Step 3 — Create a Splunk data input

In Splunk: **Settings → Data inputs → TCP** (or **UDP**) → **New**.

- **Port:** `UDP/5514` or `TCP/6514` (pick one; TCP is more reliable)
- **Source type:** `cm-st` (created by the app)
- **App Context:** *Thales Security Intelligence*
- **Method:** IP
- **Index:** `cm` (from Step 2)
- Submit.

**What this does:** opens a listener on Splunk that receives the syslog stream
from CM, tags it with the `cm-st` source type, and routes it into the `cm` index.

### Step 4 — Grant your role access to the index

In Splunk: **Settings → Roles →** your role → **Indexes** tab → select `cm`,
check **Default**, save.

**What this does:** ensures analysts can actually search the new index (and that
it's included in default searches).

### Step 5 — Point CipherTrust Manager at Splunk

In the CM web console: **Admin Settings → Syslog → Add Syslog Server** (in newer
releases this lives under **Log Forwarding / Connection Manager → Syslog**).

- **Host:** the Splunk server IP
- **Port:** `5514` or `6514` (match the input you created)
- **Protocol:** UDP or TCP (match the input)
- Add.

**What this does:** tells CipherTrust Manager to stream its server/audit records
to the Splunk listener you configured. Within moments, key operations, policy
changes, and logins begin arriving in the `cm` index.

{: .tip }
> Prefer **TCP** (or TLS) over UDP for a key manager's audit trail — you don't
> want security events silently dropped. If your Splunk input is TLS, supply the
> matching certificates on the CM syslog connection.

---

## Part B — Forward CipherTrust Transparent Encryption (CTE) logs to Splunk

CTE produces per-client **access logs** — every allowed/denied attempt to read
protected data. These stream to the *same* Thales Security Intelligence app,
independently of the CM server logs above.

### Step 1 — Enable syslog in the CTE client profile

In the CM web console, edit the CTE **Profile** used by your clients:

- **CLIENT LOGGING CONFIGURATION →** enable **Syslog**.
- **CLIENT SYSLOG CONFIGURATION →** add the Splunk server:
  - **Server:** Splunk hostname/IP and port
  - **Message Format:** `RFC5424`
  - **Transport:** **TLS** (recommended), TCP, or UDP
  - For TLS, provide the **CA certificate**, **client certificate**, and
    **private key**.

**What this does:** instructs the CTE agents (via the profile they inherit from
CM) to emit their access logs as RFC 5424 syslog messages to Splunk — encrypted
in transit when you choose TLS.

### Step 2 — Configure the Splunk syslog input (`inputs.conf`)

On the Splunk side, define a listener for the CTE stream. Add to
`/opt/splunk/etc/system/local/inputs.conf` (Linux) or
`C:\Program Files\Splunk\etc\system\local\inputs.conf` (Windows):

```ini
[default]
host = <Splunk Server IP Address>

[tcp-ssl:<port>]
listenOnIPv6 = yes
acceptFrom = *
sourcetype = rfc5424_syslog

[SSL]
password = password
requireClientCert = false
serverCert = <path to server.pem>
```

Copy the server certificate to the `serverCert` path, then **restart Splunk**.

**What this does:** opens a TLS syslog listener that accepts the CTE messages and
tags them with the `rfc5424_syslog` source type so the Thales app parses them
correctly. Restarting Splunk loads the new input.

### Step 3 — Verify events are arriving

In Splunk search, confirm data is flowing:

```spl
index=cm sourcetype=cm-st | head 20
index=* sourcetype=rfc5424_syslog | head 20
```

**What this does:** a quick sanity check — if CM audit events appear under
`cm-st` and CTE access events under `rfc5424_syslog`, both pipelines are working.
From here, the app's dashboards visualize key usage and access decisions.

---

## Summary

| Log source | CM-side config | Splunk-side config | Splunk source type |
|:-----------|:---------------|:-------------------|:-------------------|
| CM server/audit | Admin Settings → Syslog → Add Syslog Server | Index `cm` + TCP/UDP data input | `cm-st` |
| CTE client access | CTE Profile → Client (Sys)log Config, RFC5424 | `inputs.conf` TLS/TCP listener | `rfc5424_syslog` |

Both feed the **Thales Security Intelligence** app for dashboards and alerting.

---

Next: [Verification & Next Steps →]({% link docs/verification.md %})
