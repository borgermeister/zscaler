# K3s with Cilium as CNI

## Introduction

This Getting Started guide will help you deploy a 3 node K3s cluster running Cilium as CNI. The cluster will also be configured to be dual-stacked.  
This establishes the prerequisites for ZPA Connector installation on Kubernetes.

> [!NOTE]
> The IPv6 prefix *2001:db8:dead::/48* is reserved for documentation purposes as per RFC3849 and should be replaced with an appropriate prefix from your IP plan.

## Installation

### Install the Ubuntu host on Proxmox

Logon to one of your Proxmox hosts and elevate your permission to the root user account.

```bash
# Create variables used to create the virtual machine
VM_ID=2000
VM_NAME=k3s01-cilium
VM_MEMORY=4096
DISK_SIZE=32
OS_TYPE=l26
NETWORK_BRIDGE=vmbr0
VLAN_TAG=2
STORAGE_POOL=local-zfs
ISO_PATH=local:iso/ubuntu-24.04.2-live-server-amd64.iso

# Create the virtual machine using the provided variables
qm create $VM_ID \
  --name $VM_NAME \
  --agent enabled=1 \
  --machine q35 \
  --bios seabios \
  --numa 0 \
  --cpu x86-64-v2-AES \
  --ostype $OS_TYPE \
  --cores 2 \
  --sockets 1 \
  --memory $VM_MEMORY \
  --balloon 1024 \
  --serial0 socket \
  --scsihw virtio-scsi-pci \
  --virtio0 $STORAGE_POOL:$DISK_SIZE \
  --ide2 $ISO_PATH,media=cdrom \
  --boot order='virtio0;ide2' \
  --hotplug disk,network,usb,cpu \
  --net0 virtio,bridge=$NETWORK_BRIDGE,tag=$VLAN_TAG
```

| OS Type | Description                     |
| ------- | ------------------------------- |
| other   | unspecified OS                  |
| wxp     | Microsoft Windows XP            |
| w2k     | Microsoft Windows 2000          |
| w2k3    | Microsoft Windows 2003          |
| w2k8    | Microsoft Windows 2008          |
| wvista  | Microsoft Windows Vista         |
| win7    | Microsoft Windows 7             |
| win8    | Microsoft Windows 8/2012/2012r2 |
| win10   | Microsoft Windows 10/2016/2019  |
| win11   | Microsoft Windows 11/2022/2025  |
| l24     | Linux 2.4 Kernel                |
| l26     | Linux 2.6 - 6.X Kernel          |

### Preparing the Ubuntu host

If you have cloned an existing Ubuntu server from a template, there are a few steps you should take first.

I prefer to use NeoVim as a text editor. Install it using the command: `sudo apt install neovim`

```shell
# Check that 127.0.0.1 resolves to current hostname
sudo nvim /etc/hosts

# Create new SSH keys for OpenSSH
sudo rm /etc/ssh/ssh_host_*
sudo dpkg-reconfigure openssh-server
sudo systemctl restart ssh
```


> [!NOTE]
> If K3s is running on Ubuntu, you may have to set DNSStubListener to `no` to make DNS resolution work from within a container.
>
> ```shell
> sudo sed -i '/DNSStubListener=yes/c\DNSStubListener=no' /etc/systemd/resolved.conf
> sudo systemctl restart systemd-resolved
> ```

You should also set `accept-ra` to `false` in your `/etc/netplan/*.yaml` configuration to prevent Ubuntu from getting temporary IPv6 addresses.

```shell
network:
    version: 2
    ethernets:
        ens18:
            dhcp4: false
            dhcp6: false
            accept-ra: false
            addresses:
                - 10.100.2.224/24
                - "2001:db8:dead:beef::224/64"
            nameservers:
                addresses:
                    - 10.100.2.10
                    - 10.100.2.11
                    - "2001:db8:dead:beef::1:53"
                    - "2001:db8:dead:beef::2:53"
                search:
                    - k3slab.internal
            routes:
                  - to: default
                    via: 10.100.2.1
                  - to: "::/0"
                    via: "2001:db8:dead:beef::1"
                    on-link: true
```

```shell
#  When you run the following command you should only see the static configured IP addresses
hostname -I
10.100.2.224  2001:db8:dead:beef::224
```

### Install the first K3s control-plane node

The config file will achieve the following:

- `cluster-init: true` will initialize the cluster. *Only used on the first node*
- `flannel-backend: "none"` disables the default Flannel CNI
- `disable-kube-proxy: true` is used when the CNI provides its own proxy functionality
- `disable-network-policy: true` disables the default network policy controller
- `tls-san` specifies additional Subject Alternative Names(SAN) for the Kubernetes API server
- We will also disable both Traefik and ServiceLB (formerly Klipper LoadBalancer)

```shell
# Create K3s config directory
sudo mkdir -p /etc/rancher/k3s
```

> [!IMPORTANT]
> The largest supported `service-cidr` mask is /12 for IPv4, and /112 for IPv6

`sudo nvim /etc/rancher/k3s/config.yaml`

```bash
# Content of /etc/rancher/k3s/config.yaml
cluster-init: true
flannel-backend: "none"
disable-kube-proxy: true
disable-network-policy: true
tls-san:
  - "k3s-lab.k3slab.internal"
  - "k3s-lab01.k3slab.internal"
  - "k3s-lab02.k3slab.internal"
  - "k3s-lab03.k3slab.internal"
  - "10.100.2.224"
  - "10.100.2.225"
  - "10.100.2.226"
disable:
  - servicelb
  - traefik
kube-apiserver-arg:
  - "--service-cluster-ip-range=10.43.0.0/16,2001:db8:dead:ff::/112"
kube-controller-manager-arg:
  - "--cluster-cidr=10.42.0.0/16,2001:db8:dead:100::/56"
```

```shell
# Install K3s with a specific version and reference to a configuration file
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.31.6+k3s1 sh -s - --config=/etc/rancher/k3s/config.yaml
```

#### Kube-config

After the cluster is provisioned you'll have to copy `kube-config` to the default location to be able to use `kubectl` against the cluster.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/rancher/k3s/k3s.yaml $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
echo "export KUBECONFIG=$HOME/.kube/config" >> $HOME/.bashrc
source $HOME/.bashrc
```

> [!TIP]
> Since you'll be using `kubectl`quite a lot, it can be smart to create an alias:
> 
> ```shell
> echo "alias k=kubectl" >> ~/.bashrc
> source ~/.bashrc
> ```

Verify that the first control-plane node is working. Since we're missing a CNI the status will be **NotReady**

```shell
kubectl get node
NAME        STATUS     ROLES                       AGE     VERSION
k3s-lab01   NotReady   control-plane,etcd,master   5m18s   v1.31.6+k3s1
```

If you only plan to install one control-plane node you can skip all the way down to "**Install Cilium CLI**"

#### Install additional control-plane nodes (optional but recommended)

Get the join token from the first control-plane node: `/var/lib/rancher/k3s/server/token`

> [!IMPORTANT]
> Remember to first create the `config.yaml` file for K3s, but without the `cluster-init` statement.
> If you don't want to specify the `server` and `token` as part of the installation command, you can add them to `config.yaml` instead.
> Then the installation command will look like this:
> `curl -sfL https://get.k3s.io | sh -s - server --config=/etc/rancher/k3s/config.yaml`

Explanation of the installation command:

- **`sh -s - server`**: The `server` argument is passed to the K3s installation script, indicating that the node should be set up as a control-plane node. (the `server` statement may not be needed)
- **`--token`**: Specifies the join token for the cluster.
- **`--server`**: Specifies the URL of the existing K3s server's API server.
- **`--config`**: Specifies the path to the K3s configuration file.

```bash
K3S_TOKEN=<TOKEN>
API_SERVER_IP=<IP>  # IP address of the first control-plane node
API_SERVER_PORT=<PORT>  # Default port is 6443
curl -sfL https://get.k3s.io | sh -s - server \
  --config=/etc/rancher/k3s/config.yaml \
  --token ${K3S_TOKEN} \
  --server "https://${API_SERVER_IP}:${API_SERVER_PORT}"
```

#### Install worker nodes (optional)

```shell
K3S_TOKEN=<TOKEN>
API_SERVER_IP=<IP>  # IP address of the first control-plane node
API_SERVER_PORT=<PORT>  # IP address of the first control-plane node
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.31.6+k3s1 \
  K3S_URL=https://${API_SERVER_IP}:${API_SERVER_PORT} \
  K3S_TOKEN=${K3S_TOKEN} sh -
```

The environment variable `K3S_URL` tells the installation script to install K3s as an agent.

### Upgrading K3s

Before upgrading you should safely drain the node for pods:

`kubectl drain --ignore-daemonsets <node name>`

```shell
K3S_TOKEN=<TOKEN>
API_SERVER_IP=<IP>
API_SERVER_PORT=<PORT>

# Control-plane nodes
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.31.6+k3s1 sh -s - --config=/etc/rancher/k3s/config.yaml

# Worker nodes
curl -sfL https://get.k3s.io | INSTALL_K3S_VERSION=v1.31.6+k3s1 K3S_URL=https://${API_SERVER_IP}:${API_SERVER_PORT}" K3S_TOKEN=${K3S_TOKEN} sh -
```

After the upgrade is successfully done you can resume scheduling new pods to the node:

`kubectl uncordon <node name>`

## Install Cilium CLI

```bash
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
CLI_ARCH=amd64

if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi

curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum

sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin

rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
```

## Deploy Cilium to the cluster with Cilium CLI

> [!NOTE]
> Although Cilium itself is installed into the `kube-system` namespace, the `cilium` namespace will be used for Cilium CRD resources.

```bash
# Create cilium namespace
kubectl create ns cilium

# Deploy Cilium to the cluster
API_SERVER_IP=10.100.2.224
API_SERVER_PORT=6443
cilium install \
  --namespace=kube-system \
  --set k8sServiceHost=${API_SERVER_IP} \
  --set k8sServicePort=${API_SERVER_PORT} \
  --set ipam.operator.clusterPoolIPv4PodCIDRList="10.42.0.0/16" \
  --set ipam.operator.clusterPoolIPv4ServiceCIDRList="10.43.0.0/16" \
  --set ipam.operator.clusterPoolIPv4MaskSize=24 \
  --set ipam.operator.clusterPoolIPv6PodCIDRList="2001:db8:dead:100::/56" \
  --set ipam.operator.clusterPoolIPv6MaskSize=64 \
  --set ipam.operator.clusterPoolIPv6ServiceCIDRList="2001:db8:dead:ff::/112" \
  --set kubeProxyReplacement=true \
  --set l2announcements.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set bpf.masquerade=true \
  --set ipv4.enabled=true \
  --set ipv6.enabled=true \
  --set enableIPv6Masquerade=false \
  --set bgpControlPlane.enabled=true
```

```shell
# Check the status of the deployment
cilium status
cilium config view
```

### Configure Cilium

#### LoadBalancer IPAM

[https://docs.cilium.io/en/stable/network/lb-ipam/](https://docs.cilium.io/en/stable/network/lb-ipam/)

> [!NOTE]
> When using `start` and `stop` blocks, Cilium will not advertise the IP allocated for LoadBalancing.
> Use `cidr` to have Cilium announce the address via BGP.

`nvim cilium-ip-pool.yaml`

```bash
# cilium-ip-pool.yaml
---
apiVersion: cilium.io/v2alpha1
kind: CiliumLoadBalancerIPPool
metadata:
  name: cilium-ip-pool
  namespace: cilium
spec:
  allowFirstLastIPs: 'No'
  blocks:
    - cidr: "10.10.0.0/24"
    - cidr: "2001:db8:dead:dad::/64"
    # - start: '10.100.1.227'
    #   stop: '10.100.1.229'
    # - start: '2001:db8:dead:beef::ff00'
    #   stop: '2001:db8:dead:beef::ffff'
```

`kubectl apply -f  cilium-ip-pool.yaml`

#### Announcement policy

`nvim cilium-announcement-policy.yaml`

```bash
# cilium-announcement-policy.yaml
---
apiVersion: cilium.io/v2alpha1
kind: CiliumL2AnnouncementPolicy
metadata:
  name: default-l2-announcement-policy
  namespace: cilium
spec:
  externalIPs: true
  loadBalancerIPs: true
```

`kubectl apply -f cilium-announcement-policy.yaml`

#### Assign static IP to service (for reference)

[https://docs.cilium.io/en/stable/network/lb-ipam/](https://docs.cilium.io/en/stable/network/lb-ipam/)

Services can share the same IP as long as the services doesn't have conflicting ports

```yaml
---
apiVersion: v1
kind: Service
metadata:
  name: service-blue
  namespace: example
  label:
    color: blue
  annotations:
    lbipam.cilium.io/ips: "10.100.1.225,2001:db8:dead:beef::ff01"
    lbipam.cilium.io/sharing-key: "1234"
```

### Cilium BGP configuration

[https://docs.cilium.io/en/latest/network/bgp-control-plane/bgp-control-plane-v2/](https://docs.cilium.io/en/latest/network/bgp-control-plane/bgp-control-plane-v2/)

#### Enable IP forwarding

```shell
# Ubuntu / Debian
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.d/99-ip-forwarding.conf > /dev/null
echo "net.ipv6.conf.all.forwarding=1" | sudo tee -a /etc/sysctl.d/99-ip-forwarding.conf > /dev/null  
sudo sysctl -p
```

#### BGP Peering Policy

> [!NOTE]
> This policy requires that the node has the label `bgp_enable=true`

`kubectl label node k3s-lab01 bgp_enabled=true`

`nvim cilium-bgp.yaml`

```yaml
# cilium-bgp.yaml
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPClusterConfig
metadata:
  name: bgp-cluster-config
  namespace: cilium
spec:
  nodeSelector:
    matchLabels:
      bgp_enabled: "true"
  bgpInstances:
    - name: "instance-64512"
      localASN: 64512
      peers:
        - name: "peer4-64512-k3s-test"
          peerASN: 64512
          peerAddress: "10.100.2.1"
          peerConfigRef:
            name: "peer-config"
        - name: "peer6-214373-k3s-test"
          peerASN: 64512
          peerAddress: "2001:db8:dead:beef::250"
          peerConfigRef:
            name: "peer-config"
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPPeerConfig
metadata:
  name: peer-config
  namespace: cilium
spec:
  families:
    - afi: ipv4
      safi: unicast
      advertisements:
        matchLabels:
          advertise: bgp
    - afi: ipv6
      safi: unicast
      advertisements:
        matchLabels:
          advertise: bgp
  gracefulRestart:
    enabled: true
    restartTimeSeconds: 120
  # ebgpMultihop: 2
  # timers:
  #   connectRetryTimeSeconds: 12
  #   holdTimeSeconds: 9
  #   keepAliveTimeSeconds: 3
---
apiVersion: "cilium.io/v2alpha1"
kind: CiliumBGPAdvertisement
metadata:
  name: bgp-advertisement
  namespace: cilium
  labels:
    advertise: bgp
spec:
  advertisements:
    - advertisementType: "Service"
      service:
        addresses:
          - ClusterIP
          - ExternalIP
          - LoadBalancerIP
      selector:  # <-- select all services
        matchExpressions:
          - {key: somekey, operator: NotIn, values: ['never-used-value']}
      attributes:
        localPreference: 150
        communities:
          large: [ "64512:100:100" ]
    - advertisementType: "PodCIDR"
      attributes:
        localPreference: 150
        communities:
          large: [ "64512:200:200" ]
```

`kubectl apply -f cilium-bgp.yaml`

#### Verify BGP Peering

```shell
cilium bgp peers

Node       Local AS   Peer AS    Peer Address               Session State
k3s-test   64512      64512      10.100.2.1                 established
           64512      64512      2001:db8:dead:beef::250    established

cilium bgp routes advertised ipv4

Node       VRouter   Peer           Prefix         NextHop
k3s-test   64512     10.100.2.1     10.42.0.0/24   10.100.2.224

cilium bgp routes advertised ipv6

Node       VRouter   Peer                      Prefix                   NextHop
k3s-test   64512     2001:db8:dead:beef::250   2001:db8:dead:100::/64   2001:db8:dead:beef::224
```
