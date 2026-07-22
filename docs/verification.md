---
title: Verification & Next Steps
nav_order: 11
---

# Verification & Next Steps
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Use this checklist to confirm the appliance is healthy and correctly configured
before you depend on it, then follow the pointers to production-readiness.

## Post-install verification checklist

| # | Check | How to verify | Expected result |
|:--|:------|:--------------|:----------------|
| 1 | Appliance reachable | Browse `https://<appliance-ip>` | CM login page loads over HTTPS |
| 2 | Default password rotated | Try logging in with `admin`/`admin` | **Rejected** — old default no longer works |
| 3 | CLI authenticates | `ksctl login --user admin --password '<pass>'` | Returns a JWT/bearer token |
| 4 | API works end-to-end | `ksctl users list` | Lists your accounts without error |
| 5 | Networking correct | `nmcli conn show <iface> \| grep IP4.ADDRESS` | Shows your intended static IP/CIDR |
| 6 | Hostname/time set | `kscfg system hostname get`; check NTP | Correct hostname; clock accurate |
| 7 | TLS cert installed | Reload the console in a browser | No certificate warning (trusted CA) |
| 8 | Root of trust (prod) | `ksctl hsm servers list` | Your HSM server is listed |
| 9 | Backup taken | System → Backup (or `ksctl`) | A recent, stored backup exists |
| 10 | SIEM receiving data | Splunk: `index=cm sourcetype=cm-st \| head` | CM audit events appear |

### Quick health commands

```bash
# Authenticate and confirm the API path
ksctl login --user admin --password '<pass>'
ksctl users list

# Confirm networking and identity
nmcli conn show ens4 | grep IP4.ADDRESS
kscfg system hostname get

# Confirm the root of trust (production)
ksctl hsm servers list
```

**What these do:** together they prove the three things that matter — you can
authenticate to the API, the appliance has the network identity you intended,
and (for production) the HSM root of trust is attached.

## Common gotchas

{: .warning }
> - **Can't reach the console?** Check the firewall/security group allows **443**
>   from your location, and that the static IP actually applied
>   (`nmcli conn show`).
> - **`ksctl` TLS errors?** You're still on the self-signed cert — use
>   `--nosslverify` until you [install a trusted certificate]({% link docs/initial-configuration.md %}#step-5--install-a-trusted-tls-certificate-recommended),
>   then remove the flag.
> - **Planning an HSM later?** Remember `ksctl hsm setup` **resets the
>   appliance** — do it before loading keys, or budget for a reload.
> - **No SIEM data?** Confirm the CM syslog protocol/port exactly matches the
>   Splunk data input, and that NTP is set so timestamps line up.

## Next steps toward production

Once the single node is verified, the usual road to production is:

1. **Cluster** two or more CM nodes for high availability, sharing keys and
   policies.
2. **Create named admin accounts** and define **RBAC** roles; stop using the
   shared `admin` for day-to-day work.
3. **Set up domains** if you need multi-tenant isolation.
4. **Register your endpoints** — [KMIP clients, CTE, CTE-U, apps, and TDE]({% link docs/connectors.md %}).
5. **Establish key-rotation policies** and schedules.
6. **Wire up monitoring and the SIEM** ([Splunk]({% link docs/siem-splunk.md %})) and alert on
   sensitive events (failed logins, policy changes, key deletions).
7. **Schedule regular encrypted backups** and test a restore.

## Authoritative references

Always confirm exact steps against Thales' official documentation for your
version:

- CipherTrust Manager deployment — Thales Docs (`thalesdocs.com/ctp/cm`)
- Root of Trust / HSM — Thales Docs
- Log Forwarding & Splunk integration — Thales Docs
- Thales Security Intelligence app — Splunkbase (app 5594)

See the [Sources & references]({% link docs/sources.md %}) page for direct links.

---

That completes the initial installation and configuration. Back to
[Home]({% link index.md %}).
