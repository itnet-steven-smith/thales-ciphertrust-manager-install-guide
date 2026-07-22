---
title: Keys & Endpoint Integration
parent: HSM Root of Trust (Luna)
nav_order: 6
---

# Keys & Endpoint Integration — What Is (and Isn't) Transmitted
{: .no_toc }

A frequent point of confusion is *which* keys actually move around. The root of
trust never leaves the HSM, but the keys that endpoints consume behave
differently depending on the protocol and the key's attributes. This page traces
the full hierarchy and states, for each endpoint type, what crosses the boundary.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## The key hierarchy, top to bottom

```text
  ┌──────────────────────────────────────────────────────────────┐
  │ Root of Trust key            LIVES IN: Luna HSM               │
  │                              LEAVES?  Never (clone-only,      │
  │                                       same cloning domain)    │
  ├──────────────────────────────────────────────────────────────┤
  │ CipherTrust KEK chain        LIVES IN: CM database, wrapped   │
  │ (master / key-encryption     LEAVES?  No — unwrapped only by  │
  │  keys)                                 calling the HSM         │
  ├──────────────────────────────────────────────────────────────┤
  │ User / data keys             LIVES IN: CM database, wrapped   │
  │ (AES, RSA, EC, HMAC …)       LEAVES?  Only per key attributes │
  │                                       and the endpoint mode   │
  │                                       (see below)             │
  └──────────────────────────────────────────────────────────────┘
```

Two invariants hold regardless of endpoint:

1. **The root-of-trust key never leaves the HSM boundary in the clear.** It is
   used *inside* the HSM to wrap/unwrap the top of CM's KEK chain, and can only be
   *cloned* to another HSM in the same [cloning domain]({% link docs/hsm-redundancy.md %}).
2. **CM's KEK chain never leaves CM.** It sits wrapped in CM's database and is
   unwrapped only by calling the HSM.

What differs between endpoints is how the **user/data keys** at the bottom are
used.

## The deciding factor: the key's exportability

Whether a user/data key's material can be delivered to an endpoint at all is
governed by the key's attributes in CM:

- **Exportable / retrievable keys** can be returned to a properly authenticated
  endpoint (e.g. via a KMIP *Get*, or delivered to a CTE agent).
- **Non-exportable keys** never leave CM as key material — they can only be
  *used* through a server-side cryptographic operation. Requests to retrieve them
  are refused.

{: .tip }
> **Design principle.** If an endpoint only needs data encrypted/decrypted and
> never needs to hold the key, create the key as **non-exportable** and use
> server-side crypto. Reserve exportable keys for cases where the endpoint must
> perform crypto locally (e.g. high-throughput file or database encryption).

## Per-endpoint behavior

| Integration | Does key material leave CM? | How it works |
|:------------|:----------------------------|:-------------|
| **KMIP** (storage arrays, backup appliances, databases) | **Sometimes** | A KMIP client can *retrieve* a key with **Get** (key material is delivered over the mutually authenticated TLS channel) — **only if the key is exportable**. Alternatively the client asks CM to **encrypt/decrypt/sign** server-side, and the key stays in CM. Non-exportable keys support the latter only. |
| **CTE / CTE-U** (Transparent Encryption agents) | **Yes (protected)** | The CTE agent encrypts files **locally** on the host, so it must receive the GuardPoint key material from CM. Keys are delivered over the agent's secure, mutually authenticated channel and held protected on the client. Versioned keys let CM rotate the material. |
| **NAE-XML / CADP** (application data protection) | **Usually no** | The application sends data to CM's NAE interface and CM performs the cryptographic operation, returning the result — **the key stays in CM**. (CADP can also cache exportable keys for local crypto where that mode is explicitly used.) |
| **TDE** (Oracle / SQL Server via CM as external key manager) | **Depends on mode** | CM manages/stores the TDE **master (wrapping) key** that protects the database's data-encryption keys. Depending on the integration (KMIP retrieval vs. HSM-resident wrapping), the master key is either delivered to the database instance to unwrap DEKs, or used to unwrap in place. The bulk **data-encryption keys are wrapped** by that master key. |

{: .note }
> These are the general models. The exact behavior for any given integration
> depends on the key's attributes (exportable vs non-exportable), the connector,
> and the endpoint's configuration. Confirm the specifics for your protocol and
> CM version against the Thales documentation and the
> [Connecting Endpoints]({% link docs/connectors.md %}) section.

## Everything is protected in transit

Whenever key material *is* delivered to an endpoint (an exportable KMIP Get, a
CTE GuardPoint key), it travels over a **mutually authenticated TLS channel** and
is protected on arrival — it is never exposed in the clear on the wire. Endpoints
authenticate to CM with certificates, and CM authenticates to the endpoint, so
both sides of every key exchange are verified.

## How this ties back to the root of trust

Every user/data key CM issues — whether it stays in CM or is delivered to an
endpoint — is ultimately wrapped by CM's KEK chain, whose top is protected by the
**HSM root-of-trust key**. Compromising an endpoint exposes only the keys that
endpoint was authorized to hold; the root of trust, and every key still wrapped
under it, remains protected inside the Luna HSM.

## Sources

- CipherTrust Manager — Root of Trust HSM: <https://thalesdocs.com/ctp/cm/latest/admin/cm_admin/hardware-security-module/index.html>
- See also this guide's [Connecting Endpoints]({% link docs/connectors.md %}) and
  [Architecture & Root of Trust]({% link docs/architecture.md %}) pages.
