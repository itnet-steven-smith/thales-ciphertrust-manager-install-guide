---
title: Installing KSCTL
parent: Automation with KSCTL
nav_order: 1
---

# Installing KSCTL
{: .no_toc }

The `ksctl` utility ships with CipherTrust Manager and is available for Linux,
Windows, and macOS.

## On this page
{: .no_toc .text-delta }

1. TOC
{:toc}

---

{: .warning }
> **KSCTL is updated with every CipherTrust release.** While older `ksctl`
> versions may be used with newer CM versions, it is best practice to always
> update `ksctl` when a new version of CM is deployed.
>
> **Only the new version will support new features or updates to existing
> APIs.** Upgrade procedures for CM should include steps to distribute the
> updated `ksctl` executable to CM administrators.

## Download KSCTL

To access the `ksctl` download and documentation, click **API & CLI
Documentation** in the lower right of the web login screen, or click the **API**
button in the upper right when logged into the web UI.

This connects to the **API Playground** by default, so click the **CLI Guide**
button at the top left of the screen. This is the `ksctl` documentation — refer
back to this page for specific syntax. `ksctl` also has the `--help` switch to
provide syntax for any command.

To download the executable package, click the **CLI** download button in the
upper right of the screen.

Extract the appropriate binary for your system from the **`ksctl_images.zip`**
file along with the **`config_sample.yaml`** file. Rename the binary to `ksctl`
or `ksctl.exe` as appropriate and place it in your system path.

## Configure KSCTL Connection

By default, `ksctl` looks for a configuration file at
**`<home directory>/.ksctl/config.yaml`**. Create the `.ksctl` directory and
copy the sample config file into the directory as `config.yaml`.

Edit the `config.yaml` file and set these fields:

```yaml
KSCTL_URL: https://<your-cm-host>:443
KSCTL_USERNAME: <your-admin-login-name>
```

Delete the `KSCTL_PASSWORD` field and you will be prompted for it at initial
login.

Now, log in to the CM appliance:

```bash
ksctl login
```

This command authenticates to CM, obtains a JWT and refresh token, and stores
the tokens in the config file. Going forward, you may run `ksctl` commands
without explicitly logging into CM.

## Using Multiple KSCTL Configurations

In a large-scale enterprise environment, there will be multiple CM nodes in a
cluster, separate clusters for Dev, Test, and Prod environments, or even hybrid
environments between on-prem and cloud-based installations.

`ksctl` supports using multiple configuration files with the `--configfile`
switch. For example, you may have one file for the on-prem CM node and another
for a node running in AWS. Specify the appropriate config file when running a
command:

```bash
ksctl --configfile .\config-local.yaml keys list
ksctl --configfile .\config-aws.yaml keys list
```

For extensive work in one system, copy the appropriate config file to the
default `config.yaml` for convenience. A more advanced option is to use a batch
file or shell script to select the environment or node, and then write the
appropriate parameters to the default config file.

{: .note }
> Verify the server node you are connected to with the `ksctl system info get`
> command. See [Node Name]({% link docs/ksctl-initial-setup.md %}#node-name)
> for details on setting the appliance name.

## Options for Parsing JSON Output

`ksctl` is a command-line client to the CipherTrust REST APIs. The APIs return
data in JSON format, and often there is information you will want to extract for
use in following commands — object IDs, for example.

For cross-platform JSON parsing at the CLI, both **PowerShell** and **`jq`** are
available. This guide uses the `jq` command when JSON parsing is required,
although both options have their benefits.

{: .note }
> **`jq` Download:** <https://stedolan.github.io/jq/>
>
> **`jq` Tutorial:** <https://stedolan.github.io/jq/tutorial/>

{: .warning }
> **Command shell character escaping requirements for double-quotes:**
>
> Each command shell has different requirements for use of strings containing
> double quotes. Many `ksctl` commands include parameters with embedded JSON
> strings, so you will need to be familiar with the character escaping
> requirements of your chosen command shell. For example:
>
> - **BASH:** `ksctl users list | jq '.resources[] | select(.name == "admin")'`
> - **PowerShell:** `ksctl users list | jq '.resources[] | select(.name == """admin""")'`
> - **cmd.exe:** `ksctl users list | jq ".resources[] | select(.name == \"admin\")"`
