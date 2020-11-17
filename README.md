# Getting up and running with Calico on AKS

This guide looks into a few options to create an AKS cluster and explores networking and policy options offered in AKS. The examples include Azure CLI provisioned cluster and a cluster defined via ARM template.  
For more details about AKS networking options refer to [Everything you need to know about  Kubernetes networking on Azure](https://www.projectcalico.org/everything-you-need-to-know-about-kubernetes-networking-on-azure/) video recording.

`Calico open source` and `Calico Enterprise` versions are used in this guide. Refer to [Calico documentation](https://docs.projectcalico.org/calico-enterprise/) for more information on how one compares to the other.

For more information on how Azure policies compare to Calico policies refer to the [official AKS documentation](https://docs.microsoft.com/en-us/azure/aks/use-network-policies#differences-between-azure-and-calico-policies-and-their-capabilities).

## Networking options for AKS cluster

AKS offers several networking options with and without network policy engine.

- Using [kubenet](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/#kubenet) networking plugin. It provides a very basic network configuration with Host-Local IPAM and `/24` routes in the VNET associated with the host. On its own `kubenet` doesn't configure Node-to-Node communications. However, if AKS cluster provisioned with this option has multiple nodes, the routes for Node-to-Node communications will be automatically configured by AKS provisioner.  
The `kubenet` leverages Linux bridge called `cbr0` to facilitate POD-to-POD communications.

  >This option has no `Calico` involved in cluster configuration.

  ```bash
  az aks create --network-plugin kubenet ...
  ```

- Using `kubenet + Calico` networking plugin and network policy. This option is a bit misleading in its naming as it suggests that `kubenet` is used while in reality the cluster is configured to use `Calico CNI` with Host-Local IPAM and `Calico network policy` engine. Similar to pure `kubenet` option, you get `/24` routes for PODs in the POD-network VNET.  
For example in a cluster with node subnet `10.240.0.0/16` each node get an IP from that subnet (e.g. node1 `10.240.0.4`, node2 `10.240.0.5`, etc.). Each POD gets an IP from a POD-network (e.g. `10.244.0.0/16`) but from a CIDR that is associated with a node. Each node gets `/24` route configured for its PODs in the route table. E.g. PODs on node1 get `10.244.0.0/24`, PODs on node2 get `10.244.1.0/24`, and so on.

  >Note that `Calico` version in this configuration is managed by AKS controller and cannot be replaced. Any attempt to change the installed `Calico` version will be rolled back by AKS controller.

  ```bash
  az aks create --network-plugin kubenet --network-policy calico ...
  ```

- Using `azure-cni` with `azure` network policy. This option configures the cluster with `azure-cni` and `azure` network policy engine. This policy engine implements basic [Kubernetes network policies](https://kubernetes.io/docs/concepts/services-networking/network-policies/). The bridge is used on each host to facilitate POD-to-POD communications.

  ```bash
  az aks create --network-plugin azure --network-policy azure ...
  ```

- Using `azure-cni` with `Calico` network policy. In this configuration nodes and PODs use IPs from the same underlying VNET, and the POD IPs are configured as secondary IPs on the VM's Azure network interfaces. In this case the underlying VNET is fully aware of the pod IP addresses, and can route POD traffic without needing User Defined Routes. In other words, the POD-network is routable. You can tell whether the POD network is routable if you cannot see a `routetable` resource in your cluster's auxiliary resource group.

  >It is important to size the VNET properly for AKS cluster as IP exhaustion is a common issue for AKS clusters with routable POD-network.

  Such configuration allows for other Azure services to communicate with cluster PODs directly by using their IPs. `Calico` is used for network policy enforcement.

  >The installed `Calico` version is maintained by AKS controller and cannot be changed.

  ```bash
  az aks create --network-plugin azure --network-policy calico ...
  ```

- Using `azure-cni` with `transparent` network mode and `Calico` network policy. This configuration uses `azure-cni` as described in previous two options and configures `transparent` network mode that is compatible with `Calico Enterprise` which is not managed by AKS controller.  
To get AKS cluster configured with `transparent` network mode, you can either user `ARM template` or install the cluster with `--network-plugin azure` option and then use a [utility pod](https://github.com/ivansharamok/istioworkshop/blob/master/03-TigeraSecure-Install/bridge-to-transparent.yaml) to enforce `transparent` network mode on AKS nodes.

  >The `ARM template` deployment is discussed later in this guide.

  ```bash
  az aks create --network-plugin azure ...
  ```

## Configure Azure CLI extension and register a feature

Many AKS preview features including preview versions of Kubernetes are available to be used with `aks-preview` extension. It's a common practice to use this extension to get access to the latest AKS features. Refer to Azure documentation for more [details on extensions](https://docs.microsoft.com/en-us/cli/azure/azure-cli-extensions-overview?view=azure-cli-latest), [features](https://docs.microsoft.com/en-us/cli/azure/feature?view=azure-cli-latest), and [providers](https://docs.microsoft.com/en-us/cli/azure/provider?view=azure-cli-latest).

Steps to configure the extension:

- List all extensions to see if `aks-extension` already installed or has update available.

  ```bash
  az extension list-available --output table
  ```

- If the extension is already installed and has an update available, update it.

  ```bash
  az extension update --name aks-preview
  ```

- If the extension isn't installed, install it.

  ```bash
  az extension add --name aks-preview
  ```

- When you want to use a particular feature, you need to register it. This example registers `AKSNetworkModePreview` preview feature to allow configuration of `transparent` network mode for AKS cluster.

  ```bash
  # register the feature
  az feature register -n AKSNetworkModePreview --namespace Microsoft.ContainerService
  # must register with the provider
  az provider register -n Microsoft.ContainerService
  ```

- Verify the feature was registered successfully. It can take a few moments for the provider to register the feature.

  ```bash
  az feature list -o table | grep -i aksnetworkmode
  ```

## Configure service principal

AKS cluster requires a service principal to setup access to necessary cluster resources (e.g. `vnet`, `routetable`, etc.).

Configure a service principal to be used with the AKS cluster:

- Check if service principal with a specific name already exists.

  ```bash
  # define var for service principal name
  SP='calico-aks-sp'
  # list service principal
  az ad sp list --spn http://$SP --query "[].{id:appId,tenant:appOwnerTenantId,displayName:displayName,appDisplayName:appDisplayName,homepage:homepage,spNames:servicePrincipalNames}"
  ```

- Create the service principal and capture its password.

  >Note, the password may contain special characters that may need to be escaped. Alternatively, you can recreate the service principal account in attempt to avoid character escaping.

  ```bash
  SP_PASSWORD=$(az ad sp create-for-rbac --name "http://$SP" --skip-assignment | jq '.password' | sed -e 's/^"//' -e 's/"$//')
  ```

## Using `az aks` to install `Calico` on AKS

This example uses [az aks](https://docs.microsoft.com/en-us/cli/azure/aks?view=azure-cli-latest) command of [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest) to create an AKS cluster using `Calico CNI` with Host-Local IPAM and `Calico network policy` engine managed by AKS controller.  
If you want to use any AKS preview features configure [aks-preview](#configure-azure-cli-extension-and-register-a-feature) extension first.

>This example refers to `SP` and `SP_PASSWORD` variables defined in [service principal section](#configure-service-principal).

Create AKS cluster:

- Check supported k8s versions for the region

  ```bash
  # list supported k8s versions
  az aks get-versions --location $LOCATION --output table
  ```

- Login into `azure` and set helper variables.

  ```bash
  # login into az-cli
  az login
  ### set vars
  RG='calico-wbnr'
  LOCATION='centralus'
  CLUSTER_NAME='calico-cni'
  SP='calico-aks-sp'
  ROLE='Contributor'
  NET_ROLE='Network Contributor'
  K8S_VERSION=1.18.10
  ```

- Create the resource group and configure service principal roles on it

  ```bash
  # create resoruce group
  az group create --name $RG --location $LOCATION
  # get resource group ID
  RG_ID=$(az group show -n $RG --query 'id' -o tsv)
  # get service principal client/app Id
  CLIENT_ID=$(az ad sp list --display-name $SP --query '[].appId' -o tsv)
  # set service principal Contributor role on resource group
  az role assignment create --role $ROLE --assignee $CLIENT_ID --scope $RG_ID
  # [optional] if Contributor role cannot be used, use 'Network Contributor' role which provides minimum required permissions for AKS resources
  az role assignment create --role $NET_ROLE --assignee $CLIENT_ID --scope $RG_ID
  ```

- Deploy AKS cluster.

  >By default AKS cluster uses `VirtualMachineScaleSets` for its nodes. You can change it via `--vm-set-type` parameter. See `az aks create --help` for details.

  ```bash
  # var to use existing SSH key
  SSH_KEY='/path/to/ssh_key.pub'
  # deploy AKS cluster using Calico CNI w/ Host-Local IPAM and Calico net policy
  az aks create \
    --resource-group $RG \
    --name $CLUSTER_NAME \
    --kubernetes-version $K8S_VERSION \
    --nodepool-name 'nix' \
    --node-count 2 \
    --network-plugin kubenet \
    --network-policy calico \
    --service-cidr 10.0.0.0/16 \
    --dns-service-ip 10.0.0.10 \
    --docker-bridge-address 172.17.0.1/16 \
    --service-principal $CLIENT_ID \
    --client-secret $SP_PASSWORD \
    --node-osdisk-size 50 \
    --node-vm-size Standard_D2s_v3 \
    --output table \
    --ssh-key-value $SSH_KEY
  ```

- View cluster state.

  ```bash
  # list aks clusters
  az aks list --resource-group $RG --output table
  ```

- Once cluster is provisioned, retrieve `kubeconfig` info to communicate with the cluster and install `kubectl` if needed.

  ```bash
  # if needed install kubectl
  az aks install-cli
  # retrieve kubeconfig
  az aks get-credentials --resource-group $RG --name $CLUSTER_NAME --file ./kubeconfig
  ```

At this point cluster should be ready for use. See [demo section](#demo) for example app and policies to deploy onto the cluster.

## Using `ARM template` to install `Calico Enterprise` on AKS

>This example refers to `SP` and `SP_PASSWORD` variables defined in [service principal section](#configure-service-principal).

This example uses the `ARM template` and its `parameters` file located at [arm](arm/) folder in this repo. Before you can deploy the template, set required parameters `servicePrincipalClientId`, `servicePrincipalClientSecret`, and `sshRSAPublicKey` in the [aks.parameters.json](arm/aks.parameters.json) file and adjust others if needed.

>Make sure `AKSNetworkModePreview` feature is registered in your `Azure CLI` before deploying the cluster. Refer to [register a feature section](#configure-azure-cli-extension-and-register-a-feature) for more details.

- Retrieve `servicePrincipalClientId` value.

  ```bash
  CLIENT_ID=$(az ad sp list --display-name $SP --query '[].appId' -o tsv)
  ```

- Set `servicePrincipalClientSecret` using value from `SP_PASSWORD` variable defined in [service principal section](#configure-service-principal)
- Set `sshRSAPublicKey` value that represents your SSH public key. It start with `ssh-rsa ...`
- Check supported k8s versions for the region

  ```bash
  # list supported k8s versions
  az aks get-versions --location $LOCATION --output table
  ```

- Login into `azure` and set helper variables.

  ```bash
  # login into az-cli
  az login
  ### set vars
  RG='calient-wbnr'
  LOCATION='centralus'
  SP='calico-aks-sp'
  ROLE='Contributor'
  NET_ROLE='Network Contributor'
  CLUSTER_NAME='calient-azcni'
  K8S_VERSION=1.18.10
  ```

- Create resource group and set service principal role on it.

  ```bash
  # create resoruce group
  az group create --name $RG --location $LOCATION
  # get resource group ID
  RG_ID=$(az group show -n $RG --query 'id' -o tsv)
  # get service principal client/app Id
  CLIENT_ID=$(az ad sp list --display-name $SP --query '[].appId' -o tsv)
  # set service principal Contributor role on resource group
  az role assignment create --role $ROLE --assignee $CLIENT_ID --scope $RG_ID
  # [optional] if Contributor role cannot be used, use 'Network Contributor' role which provides minimum required permissions for AKS resources
  az role assignment create --role $NET_ROLE --assignee $CLIENT_ID --scope $RG_ID
  ```

- Deploy the cluster.

  ```bash
  # validate the template and parameters
  az deployment group validate --resource-group $RG --template-file arm/aks-vmss.json --parameters @arm/aks.parameters.json clusterName=$CLUSTER_NAME servicePrincipalClientId=$CLIENT_ID servicePrincipalClientSecret="$SP_PASSWORD" kubernetesVersion=$K8S_VERSION sshRSAPublicKey="$(cat $SSH_KEY)"
  # deploy the template with parameters
  az deployment group create --resource-group $RG --template-file arm/aks-vmss.json --parameters @arm/aks.parameters.json clusterName=$CLUSTER_NAME servicePrincipalClientId=$CLIENT_ID servicePrincipalClientSecret=$SP_PASSWORD kubernetesVersion=$K8S_VERSION sshRSAPublicKey="$(cat $SSH_KEY)"
  ```

- View cluster state.

  ```bash
  # list aks clusters
  az aks list --resource-group $RG --output table
  ```

- Once cluster is provisioned, retrieve `kubeconfig` info to communicate with the cluster and install `kubectl` if needed.

  ```bash
  # if needed install kubectl
  az aks install-cli
  # retrieve kubeconfig
  az aks get-credentials --resource-group $RG --name $CLUSTER_NAME --file ./kubeconfig
  ```

- Refer to [Tigera official documentation](https://docs.tigera.io/getting-started/kubernetes/managed-public-cloud/aks#install-calico-enterprise) to install Calico Enterprise on AKS

At this point cluster should be ready for use. See [demo section](#demo) for example app and policies to deploy onto the cluster.

## Remove AKS cluster

When done using the cluster, remove it and clean up related resources.

Remove the cluster using `az aks` command.

```bash
# delete AKS cluster
az aks delete -n $CLUSTER_NAME -g $RG --yes #--no-wait
```

Remove the cluster using `ARM template`.

```bash
# remove cluster using ARM template
az deployment group create --resource-group $RG --template-file arm/aks.cleanup.json --mode Complete
```

Remove resource group and service principal account.

```bash
# delete resource group
az group delete -n $RG --yes
# delete SP account
az ad sp delete --id $(az ad sp list --display-name $SP --query '[].appId' -o tsv)
```

## Demo

The demo scenario uses several PODs and a few policy examples. The folders `30-*` and `35-*` are intended for Calico Enterprise as they use policy tiers.

Use [calicoctl](https://docs.projectcalico.org/getting-started/clis/calicoctl/install) to work with policies in `Calico OSS` cluster. You can use `kubectl` for `Calico Enterprise` cluster.

### `Calico OSS` demo scenario

>Calico OSS demo scenario can be used in both Calico OSS and Calico Enterprise clusters.

- Deploy sample application.

  ```bash
  # deploy app components
  kubectl apply -f demo/10-app/
  # view deployed components
  kubectl -n demo get pod
  ```

  The sample app consists of `nginx` deployment, `centos` standalone POD, and `netshoot` standalone POD. The `nginx` instances serve static HTML page. The `centos` instance queries an external resource, `www.google.com`. The `netshoot` instance queries the `nginx` service with name `open-nginx`.

- Attach to the log stream of `centos` and `netshoot` PODs to confirm that both processes can get HTTP 200 response. Then deploy Kubernetes default deny policy for `demo` namespace.

  ```bash
  # attach to PODs log streams
  kubectl -n demo logs -f centos
  kubectl -n demo logs -f netshoot
  # deploy k8s default deny policy
  kubectl apply -f demo/20-default-deny/policy-default-deny-k8s.yaml
  ```

  After the policy is applied, both `centos` and `netshoot` processes should not be able to query the targeted resources.

- Allow `centos` to query external resource.

  ```bash
  # deploy centos allow policy
  DATASTORE_TYPE=kubernetes calicoctl apply -f demo/25-sample-policy/allow-centos-egress.yaml
  ```

  Once policy is applied, the `centos` POD should be able to access `www.google.com` resource.

- Allow DNS lookups for entire `demo` namespace and `nginx` service access for `netshoot` component.

  ```bash
  # allow cluster DNS lookups
  DATASTORE_TYPE=kubernetes calicoctl apply -f demo/25-sample-policy/allow-cluster-dns.yaml
  # allow nginx ingress
  DATASTORE_TYPE=kubernetes calicoctl apply -f demo/25-sample-policy/allow-nginx-ingress.yaml
  # allow netshoot egress
  DATASTORE_TYPE=kubernetes calicoctl apply -f demo/25-sample-policy/allow-port80-egress.yaml
  ```

  Once all three policies are applied, the `netshoot` POD should be able to get response from the `open-nginx` cluster service.

### `Calico Enterprise` demo scenario

>Calico Enterprise demo scenario can be used only in Calico Enterprise clusters as it uses enterprise features.

- Deploy sample application.

  ```bash
  # deploy app components
  kubectl apply -f demo/10-app/
  # view deployed components
  kubectl -n demo get pod
  ```

  The sample app consists of `nginx` deployment, `centos` standalone POD, and `netshoot` standalone POD. The `nginx` instances serve static HTML page. The `centos` instance queries an external resource, `www.google.com`. The `netshoot` instance queries the `nginx` service with name `open-nginx`.

- Deploy a new policy tier.

  ```bash
  # deploy security tier
  kubectl apply -f demo/30-calient-tier/
  ```

- Attach to the log stream of `centos` and `netshoot` PODs to confirm that both processes can get HTTP 200 response. Then deploy Calico default deny policy for `demo` namespace.

  ```bash
  # attach to PODs log streams
  kubectl -n demo logs -f centos
  kubectl -n demo logs -f netshoot
  # you can deploy Calico default deny policy which applies the same default deny rules to demo namespace as was used in a standard K8s policy in Calico OSS demo scenario
  kubectl apply -f demo/35-dns-policy/policy-default-deny-calico.yaml
  ```

  Once default deny policy takes affect, both `centos` and `netshoot` instances, should not be able to reach the targeted resources.

- Deploy DNS policy to allow access to external resource

  ```bash
  # deploy network sets resources
  kubectl apply -f demo/35-dns-policy/global-netset.yaml
  kubectl apply -f demo/35-dns-policy/public-ip-netset.yaml
  # deploy DNS policy
  kubectl apply -f demo/35-dns-policy/policy-allow-dns-netset-egress.yaml
  ```

  Once the DNS policy is deployed, `netshoot` pod would not be able to communicate with the `nginx` pod because the DNS policy does not explicitly define a rule to allow this communication. To fix this issue, either user Calico Enterprise Manager UI to add a `Pass` rule to the DNS policy or uncomment the `Pass` action in the `demo/35-dns-policy/policy-allow-dns-netset-egress.yaml` file and re-deploy the policy.

- Test `www.google.com` access

  ```bash
  # curl www.google.com from centos POD
  kubectl -n demo exec -t $(kubectl get pod -l app=centos -n demo -o jsonpath='{.items[*].metadata.name}') -- curl -ILs http://www.google.com | grep -i http
  # curl www.google.com from netshoot POD
  kubectl -n demo exec -t $(kubectl get pod -l app=netshoot -n demo -o jsonpath='{.items[*].metadata.name}') -- curl -ILs http://www.google.com | grep -i http
  ```

  Try to `curl` any other external resource. You should not be able to access it as `allow-dns-netset-egress` policy explicitly denies access to public IP ranges listed in `public-ip-range` global network set.

## Configure SSH access to AKS nodes

For more details on how to configure SSH access to the nodes refer to the [official AKS documentation](https://docs.microsoft.com/en-us/azure/aks/ssh#create-the-ssh-connection).

```bash
# create ssh helper pod
kubectl run --generator=run-pod/v1 -it --rm aks-ssh --image=debian
# if cluster is mixed, use this to pin pod to Linux
kubectl run -it --rm aks-ssh --image=debian --overrides='{"apiVersion":"apps/v1","spec":{"template":{"spec":{"nodeSelector":{"beta.kubernetes.io/os":"linux"}}}}}'
# install ssh client in the helper pod
apt-get update && apt-get install openssh-client -y

# set vars
SSH_KEY='/path/to/ssh_key'
# open a new terminal and run
kubectl cp $SSH_KEY $(kubectl get pod -l run=aks-ssh -o jsonpath='{.items[0].metadata.name}'):/id_rsa

# get node IP to use within aks-ssh POD
NODE_NAME='aks_node_name'
echo "NODE_IP=$(kubectl get node $NODE_NAME -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}')"

# run these commands inside of aks-ssh POD session
chmod 0600 id_rsa
ssh -i id_rsa azureuser@$NODE_IP
```
