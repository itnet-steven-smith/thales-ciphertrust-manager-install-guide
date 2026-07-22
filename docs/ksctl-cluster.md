---
title: Cluster
parent: Automation with KSCTL
nav_order: 7
---

# Configure Cluster
{: .no_toc }

All enterprise deployments of CipherTrust should be part of a replicated
cluster.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

{: .note }
> **Cluster Node Limitation** — The maximum number of cluster nodes is 20.

{: .note }
> For proper cluster functionality, every node must be synchronized to the same
> time.

{: .warning }
> For proper cluster functionality, every node must be synchronized to the same
> time. Otherwise, communication between nodes fails. You must manually configure
> the same NTP server for each node. The NTP server configuration is not
> replicated, so you must configure each node independently.

## Create Cluster

First, create the cluster.

{: .warning }
> If using host names instead of IP addresses, each CipherTrust node must be
> able to resolve its own host name as well as the names of other nodes.

```bash
ksctl cluster new --host=test.thales.lab
```

## Join a Node to the Cluster

Other nodes are now able to join the cluster. There are three steps to join a
node to the cluster, but these can be automated using the `ksctl cluster
fulljoin` command.

{: .warning }
> All data on the **joining** node will be erased. The join operation might take
> some time to complete, depending on network speed and database size.
> Completion times of 30 minutes are not unusual in production environments.

{: .note }
> For the `cluster fulljoin` command, you are required to input a member's IP or
> DNS, the joining node's IP or DNS, and either the joining node's configuration
> file, or its username, password, and URL.

```bash
ksctl cluster fulljoin --member=192.168.86.22 --newnodehost=192.168.86.33 --newnodeconfig=new-node-config.yaml
```

Or:

```bash
ksctl cluster fulljoin --member=192.168.86.22 --newnodehost=192.168.86.33 --newnodepass=joiningNodePass --newnodeuser=joiningNodeUser --newnodeurl="https://192.168.86.33"
```
