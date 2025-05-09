# Installing k0rdent

The process of installing k0rdent is straightforward, and involves the following steps:

1. Create a Kubernetes cluster to act as the management cluster
2. Install k0rdent into the management cluster
3. Add the necessary credentials and templates to work with the providers in your infrastructure.

## Create and prepare the Management Kubernetes cluster

The first step is to create the Kubernetes cluster.

### CNCF-certified Kubernetes

k0rdent is designed to run on any CNCF-certified Kubernetes. This gives you great freedom in setting up k0rdent for convenient, secure, and reliable operations.

* Proof-of-concept scale k0rdent implementations can run on a beefy Linux desktop or laptop, using a single-node-capable CNCF Kubernetes distribution like [k0s](https://k0sproject.io), running natively as bare processes, or inside a container with [KinD](https://kind.sigs.k8s.io/).
* More capacious implementations can run on multi-node Kubernetes clusters on bare metal or in clouds.
* Production users can leverage cloud provider Kubernetes variants like Amazon EKS or Azure AKS to quickly create highly-reliable and scalable k0rdent management clusters.
  
### Mixed-use management clusters

k0rdent management clusters can also be mixed-use. For example, a cloud Kubernetes k0rdent management cluster can also be used as a 'mothership' environment for [k0smotron](https://k0smotron.io) hosted control planes managed by k0rdent, integrated with workers bootstrapped (also by k0rdent) on adjacent cloud VMs or on remote VMs, bare metal servers, or edge nodes.

### Where k0rdent management clusters can live

k0rdent management clusters can live anywhere, so long as they are connected via internet to the clusters they manage. Unlike prior generations of Kubernetes cluster managers, there's no technical need to co-locate a k0rdent manager with managed clusters on a single infrastructure. In fact, k0rdent is being built to manage multiple IDPs and platforms, providing a single point of control and visibility. Deciding where to put a management cluster (or multiple clusters) is best done by assessing the requirements of your use case. Considered in the abstract, a k0rdent management cluster should be:

* Resilient and available
* Accessible and secure
* Easy to network
* Scalable (particularly for mixed-use implementations)
* Easy to operate with minimum overhead
* Monitored and observable
* Equipped for backup and disaster recovery

And of course, where can these requirements be procured at reasonable cost? In many cases the simplest and most cost-effective way to run these clusters is on a provider such as Amazon Elasti Kubernetes Service or Azure Kubernetes Service, where many fo these functions are handled for you.

### Minimum requirements

[[[ TODO ]]]

### Recommended requirements for production

[[[ TODO ]]]

### Create and prepare a Kubernetes cluster with k0s

If you already have a Kubernetes cluster into which you want to install k0rdent, skip to the next step. Otherwise, follow these steps to install and prepare a [k0s kubernetes](https://k0sproject.io) management cluster:

1. Deploy a Kubernetes cluster

    The first step is to create the actual cluster itself. Again, the actual distribution used for the management cluster isn't important, as long as it's a CNCF-compliant distribution. That means you can use an existing EKS cluster, or whatever is your normal corporate standard. To make things simple this guide uses k0s, a small, convenient, and fully-functional distribution:

    ```shell
    curl --proto '=https' --tlsv1.2 -sSf https://get.k0s.sh | sudo sh
    sudo k0s install controller --single
    sudo k0s start
    ```

    k0s includes its own preconfigured version of `kubectl` so make sure the cluster is running:

    ```shell
    sudo k0s kubectl get nodes
    ```

    You should see a single node with a status of `Ready`, as in:

    ```shell
    NAME              STATUS   ROLES    AGE   VERSION
    ip-172-31-29-61   Ready    <none>   46s   v1.31.2+k0s
    ```

2. Install kubectl

    Everything you do in k0rdent is done by creating and manipulating Kubernetes objects, so you'll need to have `kubectl` installed:

    ```shell
    sudo apt-get update
    sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
    sudo chmod 644 /etc/apt/keyrings/kubernetes-apt-keyring.gpg # allow unprivileged APT programs to read this keyring
    echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
    sudo chmod 644 /etc/apt/sources.list.d/kubernetes.list   # helps tools such as command-not-found to work correctly
    sudo apt-get update
    sudo apt-get install -y kubectl
    ```

3. Get the kubeconfig

    In order to access the management cluster you will, of course, need the kubeconfig. Again, if you're using another Kubernetes distribution follow those instructions to get the kubeconfig, but for k0s, the process involves simply copying the existing file and adding it to an environment varable so `kubectl` knows where to find it.

    ```shell
    sudo cp /var/lib/k0s/pki/admin.conf KUBECONFIG
    sudo chmod +r KUBECONFIG
    export KUBECONFIG=./KUBECONFIG
    ```
    Now you should be able to use the non-k0s `kubectl` to see the status of the cluster:

    ```shell
    kubectl get nodes
    ```

    Again, you should see the single k0s node, but by this time it should have had its role assigned, as in:

    ```console
    NAME              STATUS   ROLES           AGE   VERSION
    ip-172-31-29-61   Ready    control-plane   25m   v1.31.2+k0s
    ```

    Now the cluster is ready for installation, which we'll do using Helm.

4. Install Helm

    The easiest way to install k0rdent is through its Helm chart, so let's get Helm installed:

    ```shell
    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```

    Helm will be installed into `/usr/local/bin/helm`.

## Install k0rdent

The actual management cluster is a Kubernetes cluster with the k0rdent application installed. The simplest way to install k0rdent is through its Helm chart.  You can find the latest release [here](https://github.com/k0rdent/kcm/tags), and from there you can deploy the Helm chart, as in:

```shell
helm install kcm oci://ghcr.io/k0rdent/kcm/charts/kcm --version 0.0.7 -n kcm-system --create-namespace
```
```console
WARNING: Kubernetes configuration file is group-readable. This is insecure. Location: ./KUBECONFIG
WARNING: Kubernetes configuration file is world-readable. This is insecure. Location: ./KUBECONFIG
Pulled: ghcr.io/mirantis/hmc/charts/hmc:0.0.3
Digest: sha256:1f75e8e55c44d10381d7b539454c63b751f9a2ec6c663e2ab118d34c5a21087f
NAME: hmc
LAST DEPLOYED: Mon Dec  9 00:32:14 2024
NAMESPACE: hmc-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

(Make sure to specify the correct version number.)

While the installation process appears to be done at this point, it's not; multiple pods and templates are being created in the background, and the entire process takes a few minutes.  

To understand whether installation is complete, start by making sure all pods are ready in the `kcm-system` namespace. There should be 15:

```shell
kubectl get pods -n kcm-system
```
```console
NAME                                                           READY   STATUS
azureserviceoperator-controller-manager-86d566cdbc-rqkt9       1/1     Running
capa-controller-manager-7cd699df45-28hth                       1/1     Running
capi-controller-manager-6bc5fc5f88-hd8pv                       1/1     Running
capv-controller-manager-bb5ff9bd5-7dsr9                        1/1     Running
capz-controller-manager-5dd988768-qjdbl                        1/1     Running
helm-controller-76f675f6b7-4d47l                               1/1     Running
kcm-cert-manager-7c8bd964b4-nhxnq                              1/1     Running
kcm-cert-manager-cainjector-56476c46f9-xvqhh                   1/1     Running
kcm-cert-manager-webhook-69d7fccf68-s46w8                      1/1     Running
kcm-cluster-api-operator-79459d8575-2s9jc                      1/1     Running
kcm-controller-manager-64869d9f9d-zktgw                        1/1     Running
k0smotron-controller-manager-bootstrap-6c5f6c7884-d2fqs        2/2     Running
k0smotron-controller-manager-control-plane-857b8bffd4-zxkx2    2/2     Running
k0smotron-controller-manager-infrastructure-7f77f55675-tv8vb   2/2     Running
source-controller-5f648d6f5d-7mhz5                             1/1     Running
```

State management is handled by Project Sveltos, so you'll want to make sure that all 9 pods are running in the `projectsveltos` namespace:

```shell
kubectl get pods -n projectsveltos
```
```console
NAME                                     READY   STATUS    RESTARTS   AGE
access-manager-cd49cffc9-c4q97           1/1     Running   0          16m
addon-controller-64c7f69796-whw25        1/1     Running   0          16m
classifier-manager-574c9d794d-j8852      1/1     Running   0          16m
conversion-webhook-5d78b6c648-p6pxd      1/1     Running   0          16m
event-manager-6df545b4d7-mbjh5           1/1     Running   0          16m
hc-manager-7b749c57d-5phkb               1/1     Running   0          16m
sc-manager-f5797c4f8-ptmvh               1/1     Running   0          16m
shard-controller-767975966-v5qqn         1/1     Running   0          16m
sveltos-agent-manager-56bbf5fb94-9lskd   1/1     Running   0          15m
```

If any of these pods are missing, simply give k0rdent more time. If there's a problem, you'll see pods crashing and restarting, and you can see what's happening by describing the pod, as in:

```shell
kubectl describe pod classifier-manager-574c9d794d-j8852 -n projectsveltos
```

As long as you're not seeing pod restarts, you just need to wait a few minutes.

Next verify whether the kcm templates have been successfully installed and reconciled.  Start with the `ProviderTemplate` objects:

```shell
kubectl get providertemplate -n kcm-system
```
```console
NAME                                   VALID
cluster-api-0-0-6                      true
cluster-api-provider-aws-0-0-4         true
cluster-api-provider-azure-0-0-4       true
cluster-api-provider-openstack-0-0-1   true
cluster-api-provider-vsphere-0-0-5     true
k0smotron-0-0-6                        true
kcm-0-0-7                              true
projectsveltos-0-45-0                  true
```

Make sure that all templates are not just installed, but valid. Again, this may take a few minutes.

You'll also want to make sure the `ClusterTemplate` objects are installed and valid:

```shell
kubectl get clustertemplate -n kcm-system
```
```console
NAME                                VALID
adopted-cluster-0-0-2               true
aws-eks-0-0-3                       true
aws-hosted-cp-0-0-4                 true
aws-standalone-cp-0-0-5             true
azure-aks-0-0-2                     true
azure-hosted-cp-0-0-4               true
azure-standalone-cp-0-0-5           true
openstack-standalone-cp-0-0-2       true
vsphere-hosted-cp-0-0-5             true
vsphere-standalone-cp-0-0-5         true
```

Finally, make sure the `ServiceTemplate` objects are installed and valid:

```shell
kubectl get servicetemplate -n kcm-system
```
```console
NAME                      VALID
cert-manager-1-16-2       true
dex-0-19-1                true
external-secrets-0-11-0   true
ingress-nginx-4-11-0      true
ingress-nginx-4-11-3      true
kyverno-3-2-6             true
velero-8-1-0              true
```

## Backing up a k0rdent management cluster

In a production environment, you will always want to ensure that your management cluster is backed up. You can back it up just 
as you would any other cluster, or you can prepare your cluster for [use Velero as a backup provider](admin-backup.md).