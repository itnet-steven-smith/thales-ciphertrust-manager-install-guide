---
title: Concepts & Architecture
parent: HSM Root of Trust (Luna)
nav_order: 1
---

# Concepts & Architecture
{: .no_toc }

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## What a Hardware Security Module is

A **Hardware Security Module (HSM)** is a hardened, tamper-resistant appliance
that generates, stores, and uses cryptographic keys inside a certified security
boundary (FIPS 140-2/140-3). Keys that are marked as sensitive **never leave the
HSM in the clear** — cryptographic operations that use them (sign, wrap, unwrap,
decrypt) are executed *inside* the HSM, and only the results cross the boundary.
The **Thales Luna Network HSM** is a network-attached HSM appliance that
applications and platforms — including CipherTrust Manager — reach over an
authenticated network channel.

## The root-of-trust model in CipherTrust Manager

CipherTrust Manager protects everything it stores with an internal chain of
**key encryption keys (KEKs)**. User keys are wrapped by KEKs, which are wrapped
by higher KEKs, up to a top-level key. Where that top-level key lives determines
the strength of the whole system:

- **Without an HSM**, CM's KEK chain is anchored to a software-protected key held
  by CM itself.
- **With an HSM root of trust**, CM generates and uses a set of keys **on the
  Luna partition** that protect the top of the KEK chain. The HSM-held key
  becomes the **root of trust**: CM must call the HSM to unwrap its hierarchy, so
  no one can use CM's key material without the HSM.

{: .note }
> **Where the keys actually live.** CM keeps its key hierarchy (user keys, KEK
> chain) in its **own database**. The HSM holds only the root key(s) that protect
> the top of that chain. CM's operational keys are *not* individually stored on
> the HSM — the HSM anchors and protects them.

```text
     CipherTrust Manager                         Luna Network HSM
  ┌──────────────────────────┐               ┌───────────────────────┐
  │  User / endpoint keys     │  wrapped by   │                       │
  │        ▲                  │               │   Root of Trust key   │
  │        │ wrapped by       │  ── calls ──▶ │   (never leaves the   │
  │   KEK chain (in CM DB)    │   HSM to      │    HSM boundary)      │
  │        ▲                  │   unwrap top  │                       │
  │   Top-level KEK ──────────┼───────────────┤  wrap / unwrap here   │
  └──────────────────────────┘               └───────────────────────┘
```

{: .important }
> Because CM's KEK chain is protected by the HSM-held root key, CM must be able
> to reach the HSM (or an HA member of it) to unwrap and use its key hierarchy.
> If **all** HSM access is lost, CM cannot access the protected material — which
> is exactly why the HSM should be **redundant** (see
> [Redundant HSMs]({% link docs/hsm-redundancy.md %})). The precise runtime
> behavior on HSM loss should be confirmed against Thales documentation and your
> support team for your CM version.

## Does the root key ever leave the HSM?

The Luna partition backing the root of trust operates in **cloning mode**. In
this mode, private and secret key material is not permitted to exist outside a
trusted Luna HSM that belongs to the same **cloning domain**. The root-of-trust
key is therefore **not exportable in the clear** — it can only be *cloned* to
another Luna HSM that shares the same cloning domain (this is precisely the
mechanism that makes redundancy possible). Wrap and unwrap operations happen
**inside** the HSM.

{: .note }
> The "never leaves the boundary except via same-domain cloning" behavior is the
> defining property of Luna cloning-mode partitions. Treat the exact
> extractability attributes as version-specific and confirm against the Luna
> firmware documentation you are deploying.

## Supported HSMs

CipherTrust Manager supports a range of HSMs as a root of trust, including:

- **Luna Network HSM** versions 5, 6, and 7 (this guide targets **Luna 7**)
- Thales TCT **Luna T-Series** Network HSM
- **Luna PCIe HSM** and TCT Luna PCIe HSM
- **DPoD Luna Cloud HSM** service (Data Protection on Demand)
- AWS CloudHSM, IBM HPCS, TCT k160, and Entrust nShield Connect

{: .warning }
> CipherTrust Manager is **not** compatible with a Luna **Functionality Module
> (FM)** configuration, and a Luna Network HSM can be added as a root of trust
> only at the CM **root domain**.

## Target design: redundant HSMs for a clustered CM across two data centers

The reference architecture for this guide is a **highly available, two-data-center**
deployment:

- **DC1:** 2 CipherTrust Managers + 2 Luna Network HSMs
- **DC2:** 2 CipherTrust Managers + 2 Luna Network HSMs

All four CipherTrust Managers form **one CM cluster** (replicated key metadata).
All four Luna HSMs share a common **cloning domain** so the single root-of-trust
key can be cloned to every HSM, and each CM node reaches the HSMs through a Luna
**HA group** — so any CM can anchor to any surviving HSM.

```text
                      ┌───────────────  ONE CIPHERTRUST MANAGER CLUSTER  ───────────────┐
                      │                                                                 │
     ── DATA CENTER 1 ───────────────────┐        ┌─────────────────── DATA CENTER 2 ──
     │                                    │        │                                    │
     │   CM-1a        CM-1b               │        │        CM-2a        CM-2b          │
     │     │            │                 │        │          │            │            │
     │     └─────┬──────┘   NTLS/STC      │        │          └─────┬──────┘            │
     │           ▼          client links  │        │                ▼                   │
     │   ┌───────────────┐                │        │        ┌───────────────┐           │
     │   │  Luna HA grp  │                │        │        │  Luna HA grp  │           │
     │   │ Luna-1a Luna-1b│               │        │        │ Luna-2a Luna-2b│          │
     │   └───────┬───────┘                │        │        └───────┬───────┘           │
     │           │                        │        │                │                   │
     └───────────┼────────────────────────┘        └────────────────┼───────────────────
                 │                                                   │
                 └───────────  shared CLONING DOMAIN  ───────────────┘
                        (root-of-trust key cloned to all 4 HSMs)
```

The remaining pages build this design from the ground up:
[roles & authority]({% link docs/hsm-roles.md %}) →
[initializing each Luna]({% link docs/hsm-initialize.md %}) →
[HA & cloning across DCs]({% link docs/hsm-redundancy.md %}) →
[wiring it to the CM cluster]({% link docs/hsm-cm-integration.md %}).

## Sources

- CipherTrust Manager — Root of Trust HSM: <https://thalesdocs.com/ctp/cm/latest/admin/cm_admin/hardware-security-module/index.html>
- Luna Network HSM 7 — HA groups & cloning: <https://thalesdocs.com/gphsm/luna/7/docs/network/Content/admin_partition/ha/ha.htm>
