# HeartBeat – Windows Failover Cluster Network Management

PowerShell module and Ansible automation for deploying, configuring, validating and rolling back Windows Server Failover Cluster (WSFC) heartbeat network configurations.

## Overview

HeartBeat automates the configuration of dedicated heartbeat networking across Windows Server Failover Cluster nodes, including multi-site and cross-subnet cluster environments.

The project combines PowerShell functions with Ansible playbooks to provide a repeatable and consistent deployment process across cluster nodes.

The solution is designed to:

- Configure dedicated heartbeat network adapters.
- Identify and rename heartbeat network interfaces automatically using DNS.
- Configure static IP addressing using a central CSV configuration file.
- Configure Windows Failover Cluster network names and roles.
- Establish static routes between heartbeat networks across different subnets.
- Validate DNS resolution and network connectivity.
- Capture cluster and network configuration baselines before making changes.
- Disable legacy heartbeat interfaces following successful deployment.
- Provide rollback functionality to restore the previous heartbeat configuration.

## Features

| Feature | Description |
|---|---|
| Automated deployment | Deploy heartbeat configuration across cluster nodes using Ansible. |
| Network identification | Identifies network adapters using DNS records. |
| Dynamic IP configuration | Retrieves heartbeat IP addresses from DNS. |
| CSV configuration | Maintains network, subnet mask and gateway information centrally. |
| Cluster configuration | Renames cluster networks and configures their communication roles. |
| Cross-subnet routing | Creates persistent routes between heartbeat networks. |
| DNS validation | Checks heartbeat DNS records and connectivity. |
| Baseline collection | Exports operating system, adapter, IP, DNS and cluster configuration. |
| Rollback | Restores the previous heartbeat network configuration. |

## Repository Structure

```text
HeartBeat/
│
├── Config/
│   └── HeartBeatNetworks.CSV
│
├── Module/
│   ├── HeartBeatTools.psm1
│   └── HeartBeatTools.psd1
│
├── Scripts/
│
├── playbooks/
│   ├── HeartBeat.yml
│   └── RollBackHeartBeat.yml
│
└── README.md
```

### Config

Contains the central CSV configuration file used by the PowerShell module to determine network addressing and routing information.

### Module

Contains the PowerShell module and manifest.

**HeartBeatTools.psm1** contains the functions responsible for configuring, validating and restoring the heartbeat network.

**HeartBeatTools.psd1** defines the module metadata and PowerShell compatibility requirements.

### Scripts

Contains supporting PowerShell scripts for the project.

### playbooks

Contains the Ansible playbooks used to automate deployment and rollback across Windows cluster nodes.

---

## Prerequisites

### Windows Server

- Windows Server 2016 or later.
- Windows Server Failover Clustering installed and configured.
- Nodes must already be members of a functioning failover cluster.
- PowerShell 5.1 or later.
- Administrative privileges on cluster nodes.
- DNS records configured and resolvable between cluster nodes.
- Dedicated network adapters available for heartbeat connectivity.

Required Windows PowerShell functionality includes:

- FailoverClusters
- NetTCPIP
- NetAdapter
- DnsClient

### DNS Configuration

The solution uses DNS records to identify heartbeat network addresses.

Each cluster node is expected to have the following records:

| DNS record | Purpose |
|---|---|
| `COMPUTERNAME-HB` | Dedicated heartbeat network IP address. |
| `COMPUTERNAME-HBX` | Cross-site heartbeat network IP address. |

The records must resolve to the correct IPv4 addresses configured on the respective network adapters.

### Ansible Controller

- Ansible installed and configured.
- Windows hosts accessible through WinRM.
- `ansible.windows` collection installed.
- Ansible inventory configured with a `cluster_nodes` group.
- Administrative credentials available for Windows remote management.

Install the required Ansible collection:

```bash
ansible-galaxy collection install ansible.windows
```

---

## Network Configuration

Network configuration is maintained in:

`Config/HeartBeatNetworks.CSV`

Example:

```csv
Network,SubnetMask,Gateway,PrefixLength
192.168.10.0,255.255.255.0,192.168.10.254,24
10.10.10.0,255.255.255.0,10.10.10.254,24
10.10.11.0,255.255.255.0,10.10.11.254,24
```

Each entry defines:

| Column | Description |
|---|---|
| Network | Destination network address. |
| SubnetMask | Network subnet mask. |
| Gateway | Gateway used to reach the network. |
| PrefixLength | CIDR prefix length. |

Add the heartbeat network subnets and their corresponding gateways to this file before deployment.

**Important:** The current `Set-HeartBeatNICCSV` implementation derives the network address using a `/24` calculation. Review this logic before using non-/24 heartbeat networks.

The route configuration supports a defined set of subnet masks, which should also be checked before introducing additional subnet sizes.

---

## Installation

Clone the repository onto the Ansible controller.

```bash
git clone https://github.com/BrianHennessy2709/HeartBeat.git
```

The current Ansible playbooks expect the repository to be located at:

```text
/home/hcsadmin/ansible/HeartBeat
```

If using a different location, update the source paths in the playbooks accordingly.

---

## Ansible Deployment

The project includes two Ansible playbooks:

| Playbook | Purpose |
|---|---|
| `HeartBeat.yml` | Deploys the new heartbeat network configuration. |
| `RollBackHeartBeat.yml` | Restores the previous heartbeat network configuration. |

### Inventory Configuration

The playbooks target the `cluster_nodes` inventory group.

Example inventory:

```ini
[cluster_nodes]
A2022-N01.poc.local
B2022-N01.poc.local

[cluster_nodes:vars]
ansible_connection=winrm
ansible_winrm_transport=ntlm
ansible_port=5985
ansible_winrm_server_cert_validation=ignore
```

Configure authentication using your preferred Ansible credential or Vault arrangement.

### Deploy HeartBeat

Run the deployment playbook:

```bash
ansible-playbook playbooks/HeartBeat.yml -i inventory/windows.ini --ask-vault-pass
```

The deployment performs the following operations:

1. Creates the required PowerShell module directory on each cluster node.
2. Creates the configuration directory.
3. Copies the PowerShell module and manifest.
4. Copies the network configuration CSV.
5. Imports the PowerShell module.
6. Executes `Invoke-HeartBeatDeployment`.
7. Disables the legacy heartbeat adapter using `Disable-OldHeartBeatNIC`.
8. Returns the execution output to Ansible.

### Rollback HeartBeat

To restore the previous heartbeat configuration:

```bash
ansible-playbook playbooks/RollBackHeartBeat.yml -i inventory/windows.ini --ask-vault-pass
```

The rollback playbook:

1. Copies the module and configuration to the cluster nodes.
2. Imports the PowerShell module.
3. Validates heartbeat DNS connectivity.
4. Executes `Invoke-HeartBeatRollback`.
5. Re-enables and restores the previous heartbeat configuration.

**Note:** Review the rollback playbook and its execution order before using it in production.

---

## PowerShell Module Functions

The module exposes the following functions.

### Deployment

| Function | Description |
|---|---|
| `Invoke-HeartBeatDeployment` | Executes the complete heartbeat deployment workflow. |
| `Set-XHeartBeatNICName` | Identifies and renames the cross-site heartbeat adapter using DNS. |
| `Set-HeartBeatNIC` | Configures the dedicated heartbeat adapter. |
| `Set-HeartBeatNICCSV` | Configures heartbeat adapter addressing using the CSV configuration. |
| `Set-HeartbeatClusterNetwork` | Configures the cluster heartbeat network. |
| `Set-HeartbeatClusterNetworkCSV` | Configures the cluster heartbeat network using CSV-derived subnet information. |
| `Add-HeartBeatRoute` | Creates a heartbeat route. |
| `Add-HeartBeatRouteCSV` | Creates a heartbeat route using the CSV configuration. |

### Validation and Logging

| Function | Description |
|---|---|
| `Export-ClusterBaseline` | Exports a timestamped snapshot of the current cluster and network configuration. |
| `Test-HeartBeatDNS` | Validates heartbeat DNS records and IP connectivity. |
| `Write-Log` | Writes log entries to the screen and log file. |

### Rollback and Recovery

| Function | Description |
|---|---|
| `Invoke-HeartBeatRollback` | Executes the rollback workflow. |
| `Enable-OldHeartBeatNIC` | Re-enables the legacy heartbeat adapter. |
| `Disable-OldHeartBeatNIC` | Disables the legacy heartbeat adapter. |
| `Restore-HeartBeatNIC` | Restores the previous heartbeat adapter configuration. |
| `Restore-XHeartBeatNICName` | Restores the previous cross-site heartbeat adapter name. |
| `Set-XHeartbeatClusterNetwork` | Restores the previous cross-site cluster network configuration. |

### Additional Utilities

| Function | Description |
|---|---|
| `Remove-HiddenDevices` | Removes hidden devices from Device Manager. |

---

## Manual PowerShell Execution

The module can also be executed directly on a cluster node.

Import the module:

```powershell
Import-Module C:\Windows\Build\Modules\WSFCHeartBeatCSV\HeartBeatTools.psd1 -Force
```

Capture a baseline:

```powershell
Export-ClusterBaseline
```

Execute deployment:

```powershell
Invoke-HeartBeatDeployment
```

Validate heartbeat DNS:

```powershell
Test-HeartBeatDNS
```

Execute rollback:

```powershell
Invoke-HeartBeatRollback
```

---

## Baseline Collection

Before making configuration changes, the deployment automatically executes:

```powershell
Export-ClusterBaseline
```

The baseline is stored under:

```text
C:\Windows\Build\Logs
```

A timestamped folder is created for each execution.

The collection includes:

- Computer and operating system information.
- Cluster service status.
- Network adapter information.
- IP addressing and interface configuration.
- Routing tables.
- Adapter advanced properties.
- Cluster configuration.
- Cluster nodes and networks.
- Cluster network interfaces.
- Cluster groups and resources.
- Quorum configuration.
- Heartbeat and cross-site heartbeat DNS resolution results.

These files provide a reference for troubleshooting and comparing configurations before and after deployment.

---

## Important Considerations

Before running the deployment:

1. Verify all heartbeat DNS records resolve correctly.
2. Confirm that the dedicated heartbeat adapters are available.
3. Check that the CSV contains all required network and gateway entries.
4. Confirm that routing between the heartbeat subnets is possible.
5. Verify that the cluster is healthy and all nodes are online.
6. Review the baseline output before proceeding with production changes.

The deployment functions include validation checks to help prevent accidental modification of existing network adapters and cluster networks.

The solution is designed to run against an existing cluster and does not create the Windows Failover Cluster itself.

---

## Version Information

| Property | Value |
|---|---|
| Project | HeartBeat |
| PowerShell module | HeartBeatTools |
| Module version | 1.1.0 |
| PowerShell compatibility | 5.1 |
| Automation | Ansible |
| Platform | Windows Server |
| Author | Brian Hennessy |

## Project Repository

GitHub: https://github.com/BrianHennessy2709/HeartBeat

---

## Disclaimer

This project is intended to automate Windows Server Failover Cluster heartbeat network configuration.

Always test changes in a non-production environment before deploying to production clusters.

Ensure that a valid baseline and recovery plan are available before modifying cluster networking.
