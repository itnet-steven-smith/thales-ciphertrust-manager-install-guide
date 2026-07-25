---
title: REST API Automation (Python & PowerShell)
parent: Automation with KSCTL
nav_order: 10
---

# REST API Automation — Python &amp; PowerShell
{: .no_toc }

`ksctl` is a convenient wrapper around the CipherTrust Manager REST API. For
custom automation, CI/CD pipelines, or integration with existing tooling, you
can call the same `/api/v1/…` endpoints directly. This page shows the common
pattern — **authenticate, then manage keys** — in **Bash**, **Python**, and
**PowerShell**, and finishes by wrapping `ksctl` itself from a script.
{: .fs-6 .fw-300 }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The request pattern

Every REST call follows the same three steps:

1. **Authenticate** — `POST /api/v1/auth/tokens` with a grant type and
   credentials; the response returns a short-lived **JWT**.
2. **Authorize** — send that JWT as an `Authorization: Bearer <jwt>` header on
   every subsequent request.
3. **Act** — call the resource endpoint, e.g. `GET /api/v1/vault/keys2/` to list
   keys or `POST` to the same path to create one.

{: .note }
> **TLS verification.** In production, trust the CipherTrust Manager's
> CA-signed certificate and verify TLS on every call. The `-k` /
> `verify=<ca>` / `-SkipCertificateCheck` options below are shown for lab
> appliances using self-signed certificates — point them at your CA bundle
> instead of disabling verification when you go to production.

{: .warning }
> Never hard-code credentials. The examples read the password from an
> environment variable (`CM_PASSWORD`); in a pipeline, source it from a secret
> store (Vault, AWS Secrets Manager, GitHub Actions secrets, etc.).

---

## Bash (curl + jq)

The lightest-weight option — ideal for a quick script or a CI step where `ksctl`
is not installed.

```bash
#!/usr/bin/env bash
set -euo pipefail

CM_HOST="https://cm.thales.lab"
CM_USER="admin"
: "${CM_PASSWORD:?Set CM_PASSWORD in the environment}"

# 1. Authenticate — capture the JWT
JWT=$(curl -sk -X POST "$CM_HOST/api/v1/auth/tokens" \
  -H "Content-Type: application/json" \
  -d "{\"grant_type\":\"password\",\"username\":\"$CM_USER\",\"password\":\"$CM_PASSWORD\"}" \
  | jq -r '.jwt')

# 2. List the first 10 keys
curl -sk "$CM_HOST/api/v1/vault/keys2/?limit=10" \
  -H "Authorization: Bearer $JWT" | jq -r '.resources[].name'

# 3. Create an AES-256 key
curl -sk -X POST "$CM_HOST/api/v1/vault/keys2/" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{"name":"app-data-key","algorithm":"AES","size":256}'
```

---

## Python (requests)

A small session class keeps the token on the `requests.Session` so it is sent
automatically with every call — a good base for larger automation.

```python
import os
import requests

CM_HOST = "https://cm.thales.lab"
# Path to the CipherTrust Manager CA bundle. Use False for lab appliances only.
CM_VERIFY = os.getenv("CM_CA_BUNDLE", "/etc/ssl/certs/cm-ca.pem")


class CipherTrust:
    def __init__(self, host, verify=CM_VERIFY):
        self.host = host.rstrip("/")
        self.verify = verify
        self.session = requests.Session()

    def login(self, username, password):
        r = self.session.post(
            f"{self.host}/api/v1/auth/tokens",
            json={"grant_type": "password", "username": username, "password": password},
            verify=self.verify, timeout=30,
        )
        r.raise_for_status()
        self.session.headers["Authorization"] = f"Bearer {r.json()['jwt']}"

    def list_keys(self, limit=10):
        r = self.session.get(
            f"{self.host}/api/v1/vault/keys2/",
            params={"limit": limit}, verify=self.verify, timeout=30,
        )
        r.raise_for_status()
        return [k["name"] for k in r.json().get("resources", [])]

    def create_key(self, name, algorithm="AES", size=256):
        r = self.session.post(
            f"{self.host}/api/v1/vault/keys2/",
            json={"name": name, "algorithm": algorithm, "size": size},
            verify=self.verify, timeout=30,
        )
        r.raise_for_status()
        return r.json()


if __name__ == "__main__":
    cm = CipherTrust(CM_HOST)
    cm.login("admin", os.environ["CM_PASSWORD"])
    print("Existing keys:", cm.list_keys())
    cm.create_key("app-data-key")
```

---

## PowerShell (Invoke-RestMethod)

For Windows-centric environments. `Invoke-RestMethod` parses the JSON response
into objects automatically. `-SkipCertificateCheck` requires PowerShell 6+.

```powershell
$CmHost = "https://cm.thales.lab"
$CmUser = "admin"
if (-not $env:CM_PASSWORD) { throw "Set CM_PASSWORD in the environment" }

# 1. Authenticate
$body = @{ grant_type = "password"; username = $CmUser; password = $env:CM_PASSWORD } | ConvertTo-Json
$auth = Invoke-RestMethod -Method Post -Uri "$CmHost/api/v1/auth/tokens" `
    -ContentType "application/json" -Body $body -SkipCertificateCheck
$headers = @{ Authorization = "Bearer $($auth.jwt)" }

# 2. List the first 10 keys
$keys = Invoke-RestMethod -Uri "$CmHost/api/v1/vault/keys2/?limit=10" `
    -Headers $headers -SkipCertificateCheck
$keys.resources | Select-Object -ExpandProperty name

# 3. Create an AES-256 key
$new = @{ name = "app-data-key"; algorithm = "AES"; size = 256 } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "$CmHost/api/v1/vault/keys2/" `
    -Headers $headers -ContentType "application/json" -Body $new -SkipCertificateCheck
```

---

## Wrapping `ksctl` from a script

For bulk work it is often simpler to shell out to `ksctl` — it already handles
authentication, sessions, and domains — and parse its JSON output. This example
exports a key inventory to CSV, a common step when gathering **audit evidence**.

```python
import csv
import json
import subprocess

# `ksctl` uses the session/config established by `ksctl configure`
raw = subprocess.run(
    ["ksctl", "keys", "list"],
    capture_output=True, text=True, check=True,
).stdout
keys = json.loads(raw).get("resources", [])

with open("key_inventory.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["name", "algorithm", "state", "created"])
    for k in keys:
        writer.writerow([k.get("name"), k.get("algorithm"), k.get("state"), k.get("createdAt")])

print(f"Exported {len(keys)} keys to key_inventory.csv")
```

{: .note }
> Field names such as `state` and `createdAt` are illustrative — run
> `ksctl keys list` once and inspect the JSON to confirm the exact keys returned
> by your CipherTrust Manager version.

---

## Where to go next

- [Protocol Interfaces]({% link docs/ksctl-protocol-interfaces.md %}) — expose
  KMIP/NAE interfaces the keys above are served over.
- [SIEM Integration (Splunk)]({% link docs/siem-splunk.md %}) — forward the audit
  records these automation calls generate.
