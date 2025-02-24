---
title: Create a private Azure Kubernetes Service cluster
description: Learn how to create a private Azure Kubernetes Service (AKS) cluster
services: container-service
ms.topic: article
ms.date: 8/30/2021

---

# Create a private Azure Kubernetes Service cluster

In a private cluster, the control plane or API server has internal IP addresses that are defined in the [RFC1918 - Address Allocation for Private Internet](https://tools.ietf.org/html/rfc1918) document. By using a private cluster, you can ensure network traffic between your API server and your node pools remains on the private network only.

The control plane or API server is in an Azure Kubernetes Service (AKS)-managed Azure subscription. A customer's cluster or node pool is in the customer's subscription. The server and the cluster or node pool can communicate with each other through the [Azure Private Link service][private-link-service] in the API server virtual network and a private endpoint that's exposed in the subnet of the customer's AKS cluster.

## Region availability

Private cluster is available in public regions, Azure Government, and Azure China 21Vianet regions where [AKS is supported](https://azure.microsoft.com/global-infrastructure/services/?products=kubernetes-service).

> [!NOTE]
> Azure Government sites are supported, however US Gov Texas isn't currently supported because of missing Private Link support.

## Prerequisites

* The Azure CLI version 2.2.0 or later
* The Private Link service is supported on Standard Azure Load Balancer only. Basic Azure Load Balancer isn't supported.  
* To use a custom DNS server, add the Azure DNS IP 168.63.129.16 as the upstream DNS server in the custom DNS server.

## Create a private AKS cluster

### Create a resource group

Create a resource group or use an existing resource group for your AKS cluster.

```azurecli-interactive
az group create -l westus -n MyResourceGroup
```

### Default basic networking 

```azurecli-interactive
az aks create -n <private-cluster-name> -g <private-cluster-resource-group> --load-balancer-sku standard --enable-private-cluster  
```
Where `--enable-private-cluster` is a mandatory flag for a private cluster. 

### Advanced networking  

```azurecli-interactive
az aks create \
    --resource-group <private-cluster-resource-group> \
    --name <private-cluster-name> \
    --load-balancer-sku standard \
    --enable-private-cluster \
    --network-plugin azure \
    --vnet-subnet-id <subnet-id> \
    --docker-bridge-address 172.17.0.1/16 \
    --dns-service-ip 10.2.0.10 \
    --service-cidr 10.2.0.0/24 
```
Where `--enable-private-cluster` is a mandatory flag for a private cluster. 

> [!NOTE]
> If the Docker bridge address CIDR (172.17.0.1/16) clashes with the subnet CIDR, change the Docker bridge address appropriately.

## Configure Private DNS Zone 

The following parameters can be leveraged to configure Private DNS Zone.

- "system", which is also the default value. If the --private-dns-zone argument is omitted, AKS will create a Private DNS Zone in the Node Resource Group.
- "none", defaults to public DNS which means AKS will not create a Private DNS Zone.  
- "CUSTOM_PRIVATE_DNS_ZONE_RESOURCE_ID", which requires you to create a Private DNS Zone in this format for Azure global cloud: `privatelink.<region>.azmk8s.io`. You will need the Resource ID of that Private DNS Zone going forward.  Additionally, you will need a user assigned identity or service principal with at least the `private dns zone contributor`  and `vnet contributor` roles.
  - If the Private DNS Zone is in a different subscription than the AKS cluster, you need to register Microsoft.ContainerServices in both the subscriptions.
  - "fqdn-subdomain" can be utilized with "CUSTOM_PRIVATE_DNS_ZONE_RESOURCE_ID" only to provide subdomain capabilities to `privatelink.<region>.azmk8s.io`

### Create a private AKS cluster with Private DNS Zone

```azurecli-interactive
az aks create -n <private-cluster-name> -g <private-cluster-resource-group> --load-balancer-sku standard --enable-private-cluster --enable-managed-identity --assign-identity <ResourceId> --private-dns-zone [system|none]
```

### Create a private AKS cluster with a Custom Private DNS Zone

```azurecli-interactive
az aks create -n <private-cluster-name> -g <private-cluster-resource-group> --load-balancer-sku standard --enable-private-cluster --enable-managed-identity --assign-identity <ResourceId> --private-dns-zone <custom private dns zone ResourceId> --fqdn-subdomain <subdomain-name>
```

## Create a private AKS cluster with a Public FQDN

Prerequisites:

* Azure CLI >= 2.28.0 or Azure CLI with aks-preview extension 0.5.29 or later.
* If using ARM or the rest API, the AKS API version must be 2021-05-01 or later.

The Public DNS option can be leveraged to simplify routing options for your Private Cluster.  

![Public DNS](https://user-images.githubusercontent.com/50749048/124776520-82629600-df0d-11eb-8f6b-71c473b6bd01.png)

1. When you provision a private AKS cluster, AKS by default creates an additional public FQDN and corresponding A record in Azure public DNS. The agent nodes still use the A record in the private DNS zone to resolve the private IP address of the private endpoint for communication to the API server.

2. If you use `--private-dns-zone none`, the cluster will only have a public FQDN. When using this option, no Private DNS Zone is created or used for the name resolution of the FQDN of the API Server. The IP of the API is still private and not publicly routable.

3. If the public FQDN is not desired, you could use `--disable-public-fqdn` to disable it ("none" private dns zone is not allowed to disable public FQDN).

```azurecli-interactive
az aks create -n <private-cluster-name> -g <private-cluster-resource-group> --load-balancer-sku standard --enable-private-cluster --enable-managed-identity --assign-identity <ResourceId> --private-dns-zone <private-dns-zone-mode> --disable-public-fqdn
az aks update -n <private-cluster-name> -g <private-cluster-resource-group> --disable-public-fqdn
```

## Options for connecting to the private cluster

The API server endpoint has no public IP address. To manage the API server, you'll need to use a VM that has access to the AKS cluster's Azure Virtual Network (VNet). There are several options for establishing network connectivity to the private cluster.

* Create a VM in the same Azure Virtual Network (VNet) as the AKS cluster.
* Use a VM in a separate network and set up [Virtual network peering][virtual-network-peering].  See the section below for more information on this option.
* Use an [Express Route or VPN][express-route-or-VPN] connection.
* Use the [AKS Run Command feature](#aks-run-command).

Creating a VM in the same VNET as the AKS cluster is the easiest option.  Express Route and VPNs add costs and require additional networking complexity.  Virtual network peering requires you to plan your network CIDR ranges to ensure there are no overlapping ranges.

### AKS Run Command

Today when you need to access a private cluster, you must do so within the cluster virtual network or a peered network or client machine. This usually requires your machine to be connected via VPN or Express Route to the cluster virtual network or a jumpbox to be created in the cluster virtual network. AKS run command allows you to remotely invoke commands in an AKS cluster through the AKS API. This feature provides an API that allows you to, for example, execute just-in-time commands from a remote laptop for a private cluster. This can greatly assist with quick just-in-time access to a private cluster when the client machine is not on the cluster private network while still retaining and enforcing the same RBAC controls and private API server.

### Prerequisites

* The Azure CLI version 2.24.0 or later

### Use AKS Run Command

Simple command

```azurecli-interactive
az aks command invoke -g <resourceGroup> -n <clusterName> -c "kubectl get pods -n kube-system"
```

Deploy a manifest by attaching the specific file

```azurecli-interactive
az aks command invoke -g <resourceGroup> -n <clusterName> -c "kubectl apply -f deployment.yaml -n default" -f deployment.yaml
```

Deploy a manifest by attaching a whole folder

```azurecli-interactive
az aks command invoke -g <resourceGroup> -n <clusterName> -c "kubectl apply -f deployment.yaml -n default" -f .
```

Perform a Helm install and pass the specific values manifest

```azurecli-interactive
az aks command invoke -g <resourceGroup> -n <clusterName> -c "helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update && helm install my-release -f values.yaml bitnami/nginx" -f values.yaml
```
> [!NOTE]
> Secure access to the AKS Run Command by creating a Custom role with the "Microsoft.ContainerService/managedClusters/runcommand/action", "Microsoft.ContainerService/managedclusters/commandResults/read" permissions and assign to specific users and/or groups in combination with Just-in-Time access or Conditional Access policies. 

## Virtual network peering

As mentioned, virtual network peering is one way to access your private cluster. To use virtual network peering, you need to set up a link between virtual network and the private DNS zone.
    
1. Go to the node resource group in the Azure portal.  
2. Select the private DNS zone.   
3. In the left pane, select the **Virtual network** link.  
4. Create a new link to add the virtual network of the VM to the private DNS zone. It takes a few minutes for the DNS zone link to become available.  
5. In the Azure portal, navigate to the resource group that contains your cluster's virtual network.  
6. In the right pane, select the virtual network. The virtual network name is in the form *aks-vnet-\**.  
7. In the left pane, select **Peerings**.  
8. Select **Add**, add the virtual network of the VM, and then create the peering.  
9. Go to the virtual network where you have the VM, select **Peerings**, select the AKS virtual network, and then create the peering. If the address ranges on the AKS virtual network and the VM's virtual network clash, peering fails. For more information, see  [Virtual network peering][virtual-network-peering].

## Hub and spoke with custom DNS

[Hub and spoke architectures](/azure/architecture/reference-architectures/hybrid-networking/hub-spoke) are commonly used to deploy networks in Azure. In many of these deployments, DNS settings in the spoke VNets are configured to reference a central DNS forwarder to allow for on-premises and Azure-based DNS resolution. When deploying an AKS cluster into such a networking environment, there are some special considerations that must be taken into account.

![Private cluster hub and spoke](media/private-clusters/aks-private-hub-spoke.png)

1. By default, when a private cluster is provisioned, a private endpoint (1) and a private DNS zone (2) are created in the cluster-managed resource group. The cluster uses an A record in the private zone to resolve the IP of the private endpoint for communication to the API server.

2. The private DNS zone is linked only to the VNet that the cluster nodes are attached to (3). This means that the private endpoint can only be resolved by hosts in that linked VNet. In scenarios where no custom DNS is configured on the VNet (default), this works without issue as hosts point at 168.63.129.16 for DNS that can resolve records in the private DNS zone because of the link.

3. In scenarios where the VNet containing your cluster has custom DNS settings (4), cluster deployment fails unless the private DNS zone is linked to the VNet that contains the custom DNS resolvers (5). This link can be created manually after the private zone is created during cluster provisioning or via automation upon detection of creation of the zone using event-based deployment mechanisms (for example, Azure Event Grid and Azure Functions).

> [!NOTE]
> If you are using [Bring Your Own Route Table with kubenet](./configure-kubenet.md#bring-your-own-subnet-and-route-table-with-kubenet) and Bring Your Own DNS with Private Cluster, the cluster creation will fail. You will need to associate the [RouteTable](./configure-kubenet.md#bring-your-own-subnet-and-route-table-with-kubenet) in the node resource group to the subnet after the cluster creation failed, in order to make the creation successful.

## Limitations 
* IP authorized ranges can't be applied to the private api server endpoint, they only apply to the public API server
* [Azure Private Link service limitations][private-link-service] apply to private clusters.
* No support for Azure DevOps Microsoft-hosted Agents with private clusters. Consider to use [Self-hosted Agents](/azure/devops/pipelines/agents/agents?tabs=browser). 
* For customers that need to enable Azure Container Registry to work with private AKS, the Container Registry virtual network must be peered with the agent cluster virtual network.
* No support for converting existing AKS clusters into private clusters
* Deleting or modifying the private endpoint in the customer subnet will cause the cluster to stop functioning. 

<!-- LINKS - internal -->
[az-provider-register]: /cli/azure/provider#az_provider_register
[az-feature-register]: /cli/azure/feature#az_feature_register
[az-feature-list]: /cli/azure/feature#az_feature_list
[az-extension-add]: /cli/azure/extension#az_extension_add
[az-extension-update]: /cli/azure/extension#az_extension_update
[private-link-service]: ../private-link/private-link-service-overview.md#limitations
[virtual-network-peering]: ../virtual-network/virtual-network-peering-overview.md
[azure-bastion]: ../bastion/tutorial-create-host-portal.md
[express-route-or-vpn]: ../expressroute/expressroute-about-virtual-network-gateways.md
[devops-agents]: /azure/devops/pipelines/agents/agents
[availability-zones]: availability-zones.md
