---
title: Install UCP on Azure
description: Learn how to install Docker Universal Control Plane in a Microsoft Azure environment.
keywords: Universal Control Plane, UCP, install, Docker EE, Azure, Kubernetes
redirect_from:
- /ee/ucp/admin/install/install-on-azure/
---

Docker Universal Control Plane (UCP) closely integrates with Microsoft Azure for its Kubernetes Networking
and Persistent Storage feature set. UCP deploys the Calico CNI provider. In Azure,
the Calico CNI leverages the Azure networking infrastructure for data path
networking and the Azure IPAM for IP address management. There are
infrastructure prerequisites required prior to UCP installation for the
Calico / Azure integration.

## Docker UCP Networking

Docker UCP configures the Azure IPAM module for Kubernetes to allocate IP
addresses for Kubernetes pods. The Azure IPAM module requires each Azure virtual
machine which is part of the Kubernetes cluster to be configured with a pool of IP
addresses.

There are two options for provisoning IPs for the Kubernetes cluster on Azure:

- _An automated mechanism provided by UCP which allows for IP pool configuration and maintenance
  for standalone Azure virtual machines._ This service runs within the
  `calico-node` daemonset and provisions 128 IP addresses for each
  node by default. For information on customizing this value, see [Adjust the IP count value](#adjust-the-ip-count-value).
- _Manual provision of additional IP address for each Azure virtual machine._ This
  could be done through the Azure Portal, the Azure CLI `$ az network nic ip-config create`,
  or an ARM template. You can find an example of an ARM template
  [here](#manually-provision-ip-address-pools-as-part-of-an-azure-virtual-machine-scale-set).

## Azure Prerequisites

You must meet the following infrastructure prerequisites to successfully deploy Docker UCP on Azure. **Failure to meet these prerequisites may result in significant errors during the installation process.**

- All UCP Nodes (Managers and Workers) need to be deployed into the same Azure
  Resource Group. The Azure Networking components (Virtual Network, Subnets,
  Security Groups) could be deployed in a second Azure Resource Group.
- The Azure Virtual Network and Subnet must be appropriately sized for your
  environment, as addresses from this pool will be consumed by Kubernetes Pods.
  For more information, see [Considerations for IPAM
  Configuration](#considerations-for-ipam-configuration).
- All UCP worker and manager nodes need to be attached to the same Azure
  Subnet.
- Internal IP addresses for all nodes should be [set to
  Static](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-static-private-ip-arm-pportal),
  rather than the default of Dynamic.
- The Azure Virtual Machine Object Name needs to match the Azure Virtual Machine
  Computer Name and the Node Operating System's Hostname which is the FQDN of
  the host, including domain names. Note that this requires all characters to be in lowercase.
- An Azure Service Principal with `Contributor` access to the Azure Resource
  Group hosting the UCP Nodes. This Service principal will be used by Kubernetes
  to communicate with the Azure API. The Service Principal ID and Secret Key are
  needed as part of the UCP prerequisites. If you are using a separate Resource
  Group for the networking components, the same Service Principal will need
  `Network Contributor` access to this Resource Group.

UCP requires the following information for the installation:

- `subscriptionId` - The Azure Subscription ID in which the UCP
objects are being deployed.
- `tenantId` - The Azure Active Directory Tenant ID in which the UCP
objects are being deployed.
- `aadClientId` - The Azure Service Principal ID.
- `aadClientSecret` - The Azure Service Principal Secret Key.

### Azure Configuration File

For UCP to integrate with Microsoft Azure, all Linux UCP Manager and Linux UCP
Worker nodes in your cluster need an identical Azure configuration file,
`azure.json`.  Place this file within `/etc/kubernetes` on each host. Since the
configution file is owned by `root`, set its permissions to `0644` to ensure
the container user has read access.

The following is an example template for `azure.json`. Replace `***` with real values, and leave the other
parameters as is.

```json
{
    "cloud":"AzurePublicCloud",
    "tenantId": "***",
    "subscriptionId": "***",
    "aadClientId": "***",
    "aadClientSecret": "***",
    "resourceGroup": "***",
    "location": "***",
    "subnetName": "***",
    "securityGroupName": "***",
    "vnetName": "***",
    "useInstanceMetadata": true
}
```

There are some optional parameters for Azure deployments:

- `primaryAvailabilitySetName` - The Worker Nodes availability set.
- `vnetResourceGroup` - The Virtual Network Resource group, if your Azure Network objects live in a
seperate resource group.
- `routeTableName` - If you have defined multiple Route tables within
an Azure subnet.

See the [Kubernetes Azure Cloud Provider Config](https://github.com/kubernetes/cloud-provider-azure/blob/master/docs/cloud-provider-config.md) for more details on this configuration file.

## Guidelines for IPAM Configuration

> **Warning**
>
> You must follow these guidelines and either use the appropriate size network in Azure or take the proper action to fit within the subnet.
> Failure to follow these guidelines may cause significant issues during the
> installation process.

The subnet and the virtual network associated with the primary interface of the
Azure virtual machines need to be configured with a large enough address
prefix/range. The number of required IP addresses depends on the workload and
the number of nodes in the cluster.

For example, in a cluster of 256 nodes, make sure that the address space of the subnet and the
virtual network can allocate at least 128 * 256 IP addresses, in order to run a maximum of 128 pods
concurrently on a node. This would be ***in addition to*** initial IP allocations to virtual machine
NICs (network interfaces) during Azure resource creation.

Accounting for IP addresses that are allocated to NICs during virtual machine bring-up, set
the address space of the subnet and virtual network to `10.0.0.0/16`. This
ensures that the network can dynamically allocate at least 32768 addresses,
plus a buffer for initial allocations for primary IP addresses.

> Azure IPAM, UCP, and Kubernetes
>
> The Azure IPAM module queries an Azure virtual machine's metadata to obtain
> a list of IP addresses which are assigned to the virtual machine's NICs. The
> IPAM module allocates these IP addresses to Kubernetes pods. You configure the
> IP addresses as `ipConfigurations` in the NICs associated with a virtual machine
> or scale set member, so that Azure IPAM can provide them to Kubernetes when
> requested.
{: .important}

## Manually provision IP address pools as part of an Azure virtual machine scale set

Configure IP Pools for each member of the virtual machine scale set during provisioning by
associating multiple `ipConfigurations` with the scale set’s
`networkInterfaceConfigurations`. Here's an example `networkProfile`
configuration for an ARM template that configures pools of 32 IP addresses
for each virtual machine in the virtual machine scale set.

```json
"networkProfile": {
  "networkInterfaceConfigurations": [
    {
      "name": "[variables('nicName')]",
      "properties": {
        "ipConfigurations": [
          {
            "name": "[variables('ipConfigName1')]",
            "properties": {
              "primary": "true",
              "subnet": {
                "id": "[concat('/subscriptions/', subscription().subscriptionId,'/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/virtualNetworks/', variables('virtualNetworkName'), '/subnets/', variables('subnetName'))]"
              },
              "loadBalancerBackendAddressPools": [
                {
                  "id": "[concat('/subscriptions/', subscription().subscriptionId,'/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/loadBalancers/', variables('loadBalancerName'), '/backendAddressPools/', variables('bePoolName'))]"
                }
              ],
              "loadBalancerInboundNatPools": [
                {
                  "id": "[concat('/subscriptions/', subscription().subscriptionId,'/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/loadBalancers/', variables('loadBalancerName'), '/inboundNatPools/', variables('natPoolName'))]"
                }
              ]
            }
          },
          {
            "name": "[variables('ipConfigName2')]",
            "properties": {
              "subnet": {
                "id": "[concat('/subscriptions/', subscription().subscriptionId,'/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/virtualNetworks/', variables('virtualNetworkName'), '/subnets/', variables('subnetName'))]"
              }
            }
          }
          .
          .
          .
          {
            "name": "[variables('ipConfigName32')]",
            "properties": {
              "subnet": {
                "id": "[concat('/subscriptions/', subscription().subscriptionId,'/resourceGroups/', resourceGroup().name, '/providers/Microsoft.Network/virtualNetworks/', variables('virtualNetworkName'), '/subnets/', variables('subnetName'))]"
              }
            }
          }
        ],
        "primary": "true"
      }
    }
  ]
}
```

## UCP Installation

### Adjust the IP Count Value

During a UCP installation, a user can alter the number of Azure IP addresses
UCP will automatically provision for pods. By default UCP will provision 128
addresses, from the same Azure Subnet as the hosts, for each Virtual Machine in
the cluster. However if you have manually attached additional IP addresses
to the Virtual Machines (via an ARM Template, Azure CLI or Azure Portal) or you
are deploying in to small Azure subnet (less than /16), an `--azure-ip-count`
flag can be used at install time. 

> Note: Do not set the `--azure-ip-count` variable to a value of less than 6 if
> you have not manually provisioned additional IP addresses for each Virtual
> Machine. The UCP installation will need at least 6 IP addresses to allocate
> to the core UCP components that run as Kubernetes pods. That is in addition
> to the Virtual Machine's private IP address.

Below are some example scenarios which require the `--azure-ip-count` variable
to be defined. 

**Scenario 1 - Manually Provisioned Addresses**

If you have manually provisioned additional IP addresses for each Virtual
Machine, and want to disable UCP from dynamically provisioning more IP
addresses for you, then you would pass `--azure-ip-count 0` into the UCP
installation command.

**Scenario 2 - Reducing the number of Provisioned Addresses**

If you want to reduce the number of IP addresses dynamically allocated from 128
addresses to a custom value due to:

- Primarily using the Swarm Orchestrator
- Deploying UCP on a small Azure subnet (for example /24)
- Plan to run a small number of Kubernetes pods on each node. 

For example if you wanted to provision 16 addresses per virtual machine, then
you would pass `--azure-ip-count 16` into the UCP installation command. 

If you need to adjust this value post-installation, see
[instructions](https://docs.docker.com/ee/ucp/admin/configure/ucp-configuration-file/) on how to download the UCP
configuration file, change the value, and update the configuration via the API.
If you reduce the value post-installation, existing virtual machines will not
be reconciled, and you will have to manually edit the IP count in Azure.  

### Install UCP

Run the following command to install UCP on a manager node. The `--pod-cidr`
option maps to the IP address range that you have configured for the Azure
subnet, and the `--host-address` maps to the private IP address of the master
node. Finally if you want to adjust the amount of IP addresses provisioned to
each virtual machine pass `--azure-ip-count`.

> **Note**
>
> The `pod-cidr` range must match the Azure Virtual Network's Subnet
> attached the hosts. For example, if the Azure Virtual Network had the range
> `172.0.0.0/16` with Virtual Machines provisioned on an Azure Subnet of
> `172.0.1.0/24`, then the Pod CIDR should also be `172.0.1.0/24`.

```bash
docker container run --rm -it \
  --name ucp \
  --volume /var/run/docker.sock:/var/run/docker.sock \
  {{ page.ucp_org }}/{{ page.ucp_repo }}:{{ page.ucp_version }} install \
  --host-address <ucp-ip> \
  --pod-cidr <ip-address-range> \
  --cloud-provider Azure \
  --interactive
```
