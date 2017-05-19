# kube-coreos-bash

## Background
I needed to build an on-prem multi-node k8s cluster on bare metal. BASH scripts were portable and easiest to quickly get up to speed. I know this isn't the best way to spin up a cluster on bare-metal, but it's arguably the easiest way especially if you are a seasoned sysadmin.

The scripts were adapted from [this page](https://github.com/coreos/coreos-kubernetes/tree/master/multi-node/generic). The original tutorial is found [here](https://coreos.com/kubernetes/docs/latest/getting-started.html).

## Why bare-metal?
**Simplicity.** Once it's installed on a host machine, you don't ever need to SSH in to manage Container Linux. That is the beauty of kubernetes on Container Linux.

**Performance.** It is a no-brainer that the more layers of virtualisation below your workload, the poorer it performs. This is a reason why Microsoft developed the [FreeFlow network overlay](https://www.microsoft.com/en-us/research/publication/freeflow-high-performance-container-networking-3/) in the first place.

**Lower Ops Overhead.** This is in comparison to deploying kubernetes on top of a fresh install of VMware vCenter or KVM. If your team uses containers exclusively (ie. don't need VMs), then there is no point installing kubernetes in VMs.

Ultimately, it depends on your infra environment. If your sysadmin will only supply you with VMs, you have little choice. If the idea of managing physical machines makes you cringe, please deploy on the cloud. Nevertheless, you still can use the BASH scripts to deploy your cluster. If bare-metal is an option and you like or have to manage physical infra, then these scripts will work for you.

## Viable alternatives
The only viable alternative for on-prem bare metal installation is [kargo](https://github.com/kubernetes-incubator/kargo), which I am still exploring. It does the job very well, but I need to heavily modify it to fit my requirements. These are:
1. TLS between all control-plane elements
1. RBAC-ready certs for the control-plane elements

**kargo** is a solid deployment tool, but I was on a tight timeline. I wasn't keen on mucking around with the complex Ansible scripts.

Much of the community is focused on **kops**(https://github.com/kubernetes/kops) and **kubeadm**(https://kubernetes.io/docs/getting-started-guides/kubeadm/) which are for cloud platforms and don't support bare-metal.

## The future of this repo
Sad to say, once I firm up the scripts I probably won't update the scripts anymore. It works OOTB for kubernetes v1.6. That being said, I believe it will work out of the box for many kubernetes versions to come. We shall see!

## Why _almost_?
In this setup, it is assumed that `kube-apiserver` is load-balanced by DNS round-robin. This is not a robust solution. Instead, a HTTP load-balancer (eg. nginx) pod should be installed on each worker node that load-balances between the controller nodes. kargo does this.

I will try to scrape some time to work on this, but feel free to PR on the code if you work it out!


# CoreOS Container Linux Installation Guide
The instructions to setup Container Linux won't be thorough. It'll instead point you to the right guides.

## Install CoreOS on the machines
For a trial run of your production environment, you can install CoreOS Container Linux on libvirt. CoreOS has a [wonderful guide](https://coreos.com/os/docs/latest/booting-with-libvirt.html) which still works (tested in May 2017).

Iif you're ready to deploy on bare-metal, use this [guide](https://coreos.com/kubernetes/docs/latest/kubernetes-on-baremetal.html).

## Configure networking in Container Linux

### Set static IPs and routes
You should configure static IPs on the hosts especially if this is a production cluster. Use [this guide](https://coreos.com/os/docs/latest/network-config-with-networkd.html) to configure a static IP.

### Configure a HTTP proxy in Docker
If your machines are behind a corporate proxy, [configure Docker](https://coreos.com/os/docs/latest/customizing-docker.html) to use it.

# Kubernetes Installation Guide
The guide is still a works-in-progress.

## Generate certs
On your device, generate the certs needed by the controller and worker nodes. The scripts uses `ping` to extract the IPs from the hostnames. At minimum, the hostnames of the nodes must exist in `/etc/hosts`.

1. In `gencert.sh`, edit the variables under `#### Config`.
1. Run `gencert.sh`

To send the certs to the hosts, ensure that they are accessible by hostname via SSH.

1. Run `sendcert.sh`

## Install etcd
`etcd` is the configuration storage engine that kubernetes uses.

1. Copy `etcd-install.sh` to each of the controller nodes
1. In `etcd-install.sh`,
    1. Change `ADVERTISE_IP` to the node's IP.
    1. Update `ETCD_ENDPOINTS` to reflect the hostnames and IPs of the controller nodes.
    1. Update `ETCD_VER` to your desired version. Check [here](https://quay.io/coreos/etcd) for the versions available.
1. Run `etcd-install.sh`
    ```
    sudo sh ./etcd-install.sh
    ```
1. Check cluster health. If it doesn't say `cluster is healthy`, you need to rectify this before installing kubernetes.
    ```
    cd /etc/etcd/ssl; sudo etcdctl --ca-file ca.pem --cert-file peer.pem --key-file peer-key.pem cluster-health
    ```

## Setup the kubernetes controller nodes
The controller has 3 control-plane elements: `kube-apiserver`, `kube-controller-manager`, `kube-scheduler`. `kubelet` and `kube-proxy` are also installed althought they are non-essential elements for the controller.

`kubelet` is needed to bootstraps the self-hosted cluster which means that the control-plane elements also run as pods on the kubernetes network. In particular, it runs the `hyperkube` pod to bring up the 3 control-plane elements. W.r.t. cluster deployments, the k8s project is gravitating towards a [self-hosted model](https://coreos.com/blog/self-hosted-kubernetes.html). Since the cluster is self-hosted, `kube-proxy` is needed to connect the pods to the pod network.

1. Copy `controller-install.sh` to each of the controller nodes.
1. In `controller-install.sh`,
    1. Change `ADVERTISE_IP` to the node's IP.
    1. Update `ETCD_ENDPOINTS` to reflect the hostnames and IPs of the controller nodes.
    1. Update `K8S_VER` to your desired version. Check [here](quay.io/coreos/hyperkube) for the versions available.
1. Run `controller-install.sh`
    ```
    sudo sh ./controller-install.sh
    ```

## Setup the kubernetes worker nodes
Two k8s elements run in the worker nodes: `kubelet` and `kube-proxy`. `kubelet` takes instructions from the controller while `kube-proxy` the node and its pods to the overlay network.

1. Copy `worker-install.sh` to each of the worker nodes.
1. In `worker-install.sh`,
    1. Change `ADVERTISE_IP` to the node's IP.
    1. Update `ETCD_ENDPOINTS` to reflect the IPs of the controller nodes.
    1. Update `CONTROLLER_ENDPOINT` to that of your DNS RR record that load-balances your controllers. This should match `gencert.sh`'s config.
    1. Update `K8S_VER` to your desired version. This should match the version used in the controller nodes.
1. Run `controller-install.sh`
    ```
    sudo sh ./controller-install.sh
    ```

## Setup kubectl
Now that your cluster is up, you can mess around with it.

1. Install `kubectl` on your device. The installation method depends on your OS.
1. Copy your generated certs to the `~/.kube` folder.
1. Add the following config `~/.kube/config` and modify accordingly.
    ```
    apiVersion: v1
    clusters:
    - cluster:
        certificate-authority: /Users/eugene/.kube/ca.pem
        server: https://con.kube:6443
      name: my-cluster
    contexts:
    - context:
        cluster: my-cluster
        user: admin
      name: my-cluster-admin
    current-context: my-cluster-admin
    kind: Config
    preferences: {}
    users:
    - name: admin
      user:
        client-certificate: /Users/admin/.kube/client.pem
        client-key: /Users/admin/.kube/client-key.pem
    ```
1. Check out your cluster. If this succeeds, congratulations!
    ```
    kubectl -n kube-system get pod
    ```