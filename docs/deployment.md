---
title: Deployment
nav_order: 5
has_children: true
---

# Deployment

CipherTrust Manager ships as the *same* virtual appliance for every platform, so
the post-boot configuration is identical no matter where you run it. Only the way
you get the VM created differs. Pick the path that matches your environment:

- **[Private Cloud — VMware vSphere/ESXi]({% link docs/deploy-vmware.md %})** — deploy the OVA
  template. This is the most common on-premises path and demonstrates the
  private-cloud use case.
- **[Public Cloud — AWS]({% link docs/deploy-aws.md %})** — launch the Marketplace/AMI image on
  EC2. This demonstrates the public-cloud use case, where the cloud runs the
  infrastructure but you keep control of the keys.

Both paths converge on the same next step —
[Initial Configuration]({% link docs/initial-configuration.md %}) — where you set networking, rotate
the default credentials, and reach the web console and `ksctl` CLI.

{: .note }
> You do **not** need to do both. Follow one path end-to-end. The two are
> presented together only so this guide covers both the private- and public-cloud
> scenarios you asked about.
