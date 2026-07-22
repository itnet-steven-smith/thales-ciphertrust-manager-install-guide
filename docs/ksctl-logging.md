---
title: Logging & Alerts
parent: Automation with KSCTL
nav_order: 5
---

# Configure Logging
{: .no_toc }

As of version 2.9, CipherTrust has two local logging mechanisms. One is the
legacy local database, which is displayed under **Records > Server Records** and
is replicated between all nodes in the cluster. However, this is very
inefficient in moderate to large enterprises. A new logging micro-service based
on **Grafana Loki** has been added which is easier to read, supports an advanced
query language — **LogQL** — and is no longer replicated between all nodes to
reduce traffic and load within a cluster.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## Disable Local Log DB (Use embedded Loki)

{: .tip }
> **Best Practice:** Disable the local log database and use the embedded Loki
> micro-service.

```bash
ksctl properties modify -n ENABLE_RECORDS_DB_STORE -p false
```

## Extend Loki Retention

By default, Loki will retain records for one week. This can be extended, but be
aware of the impact of additional logging on local storage.

```bash
ksctl loki config modify --retention-time "336h"
```

## Syslog

To send logs to a SIEM or other log analytics solution, first create a syslog
connection assigned to the `logger` service, and then create a log forwarder
that uses that connection.

{: .tip }
> **Best Practice:** Always forward security logs to a central log aggregator or
> analytics platform.

{: .warning }
> Syslog servers configured as log forwarders can forward client audit records,
> while syslog servers configured through the legacy Admin Settings cannot.

### Add Syslog Connection

```bash
ksctl connectionmgmt log-forwarder syslog create --name splunk --host splunk.thales.lab --port 514 --transport udp --products "logger" | jq .id
```

```text
"cef6a38e-0bd1-4f8f-ac78-c60454426024"
```

### Add Syslog Forwarder for Server and Client Records

```bash
ksctl log-forwarders add syslog --name splunk --connection-id "cef6a38e-0bd1-4f8f-ac78-c60454426024" --forward-server-audit-records true --forward-client-audit-records true | jq .id
```

```text
"6bcc74ce-b050-498a-89ea-4c8178569d9b"
```

### Enable and Forward KMIP and NAE-XML Records

{: .warning }
> Excessive logging can impact performance.

```bash
ksctl properties modify --name ENABLE_KMIP_ACTIVITY_LOGS --value true
```

```bash
ksctl properties modify --name ENABLE_NAE_ACTIVITY_LOGS --value true
```

```bash
ksctl log-forwarders modify syslog --id "6bcc74ce-b050-498a-89ea-4c8178569d9b" --forward-logs-activity-kmip true --forward-logs-activity-nae true
```

## Alerts

Alerts are defined from queries that compare values to specific elements in a
log message. If the log matches the query, the alarm is triggered based on the
log message threshold and time interval, if provided. For more information on
creating alerts, refer to **thalesdocs.com**.

{: .warning }
> Each operating system or shell environment handles the escaping of quotes in a
> string parameter differently. To avoid this complication, put the alarm
> condition query in a text file and pass that using the `-f` switch.

### Keys Management

#### Weak RSA Key

```text
# alarm-weak-rsa.txt
# input.success; input.message == "Create Key"; input.details.algorithm == "RSA"; input.details.size < 2048
```

```bash
ksctl records alarm-configs create --name "Weak RSA Key" --description "RSA key should be 2048 bits or larger" --severity warning -f alarm-weak-rsa.txt
```

#### Weak AES Key

```text
# alarm-weak-aes.txt
# input.success; input.message == "Create Key"; input.details.algorithm == "AES"; input.details.size < 256
```

```bash
ksctl records alarm-configs create --name "Weak AES Key" --description "AES key should be 256 bits or larger" --severity warning -f alarm-weak-aes.txt
```

#### Key Deleted

```text
# alarm-key-deleted.txt
# input.success; input.message == "Delete Key"
```

```bash
ksctl records alarm-configs create --name "Key Deleted" --description "A key has been deleted" --severity critical -f alarm-key-deleted.txt
```

#### Key Modified

```text
# alarm-key-update.txt
# input.success; input.message == "Update Key"
```

```bash
ksctl records alarm-configs create --name "Key Updated" --description "A key has been modified" --severity warning -f alarm-key-update.txt
```

#### Keys Exported

```text
# alarm-key-export.txt
# input.success; input.message == "Exporting Keys"
```

```bash
ksctl records alarm-configs create --name "Multiple Keys Exported" --description "More than 5 keys have been exported in 5 seconds" --severity warning -t 10 -p 5 -f alarm-key-export.txt
```

### Certificate Authority

#### Local CA Created

```text
# alarm-ca-create-local.txt
# input.success; input.message == "Create Local CA"
```

```bash
ksctl records alarm-configs create --name "Local CA Created" --description "New Local CA created" --severity info -f alarm-ca-create-local.txt
```

#### Local CA Deleted

```text
# alarm-ca-delete-local.txt
# input.success; input.message == "Delete Local CA"
```

```bash
ksctl records alarm-configs create --name "Local CA Deleted" --description "Local CA deleted" --severity critical -f alarm-ca-delete-local.txt
```

#### External CA Added

```text
# alarm-ca-create-external.txt
# input.success; input.message == "Upload External CA"
```

```bash
ksctl records alarm-configs create --name "External CA trusted" --description "New external CA trusted" --severity warning -f alarm-ca-create-external.txt
```

#### External CA Deleted

```text
# alarm-ca-delete-external.txt
# input.success; input.message == "Delete External CA"
```

```bash
ksctl records alarm-configs create --name "External CA deleted" --description "An external trusted CA has been deleted" --severity critical -f alarm-ca-delete-external.txt
```

### Protocol Interfaces

#### Interface Created

```text
# alarm-port-create.txt
# input.success; input.message == "Create Interface Configuration"
```

```bash
ksctl records alarm-configs create --name "Protocol Interface Created" --description "New protocol interface created" --severity warning -f alarm-port-create.txt
```

#### Interface Deleted

```text
# alarm-port-delete.txt
# input.success; input.message == "Delete Interface Configuration"
```

```bash
ksctl records alarm-configs create --name "Protocol Interface Deleted" --description "A protocol interface has been deleted" --severity warning -f alarm-port-delete.txt
```

#### Interface Modified

```text
# alarm-port-update.txt
# input.success; input.message == "Update Interface Configuration"
```

```bash
ksctl records alarm-configs create --name "Protocol Interface Modified" --description "A protocol interface has been modified" --severity critical -f alarm-port-update.txt
```

### CipherTrust Transparent Encryption

#### CTE GuardPoint Deleted

```text
# alarm-cte-gp-deleted.txt
# input.success; input.message == "Delete CTE GuardPoint"
```

```bash
ksctl records alarm-configs create --name "CTE GuardPoint Deleted" --description "A guardpoint has been deleted from a CTE client" --severity warning -f alarm-cte-gp-deleted.txt
```

#### CTE Client Deleted

```text
# alarm-cte-client-deleted.txt
# input.success; input.message == "Delete CTE Client"
```

```bash
ksctl records alarm-configs create --name "CTE Client Deleted" --description "A CTE client has been deleted" --severity critical -f alarm-cte-client-deleted.txt
```

#### Error Updating CTE Client

```text
# alarm-cte-client-error.txt
# input.success=false; input.message == "Update CTE Client"
```

```bash
ksctl records alarm-configs create --name "Cannot Update CTE Client" --description "There has been an error updating a CTE client" --severity info -f alarm-cte-client-error.txt
```
