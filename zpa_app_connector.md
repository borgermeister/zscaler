# ZPA App Connector

## ZPA App Connector on Kubernetes

### Prerequisites

Install Helm with `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash`

> [!NOTE]
> Sometimes Curl doesn't like IPv6. You can fix this by creating a config file telling Curl to use IPv4.  
> `echo ipv4 >> ~/.curlrc`

Retrieve the provisioning key from the [Zscaler Access Admin Portal](https://console.zscaler.com/private#connectors)

### Add the Zscaler Private Access Connector Helm repository

This chart deploys the Zscaler Private Access Connector on Kubernetes.

```shell
# Add the Zscaler Private Access Connector Helm repository
helm repo add zpa-helm-repo https://dist.private.zscaler.com/helm-charts/k8s/

# Search for available charts in the repository
helm search repo zpa-helm-repo

# Show values for the chart
helm show values zpa-helm-repo/zpa-app-connector-kubernetes > values.yaml
```

### Modify the values file

You can create a new `values.yaml` file and change the default values for the Helm chart.
For example, you can change the memory and CPU requests and limits.

```yaml
---
zsglobal:
  autoscaling:
    enabled: false
    minReplicas: 2
    maxReplicas: 8
  replicaCount: 2
  resources:
    limits:
      memory: "1Gi"
      cpu: "800m"
      # memory: "8000Mi" # default Helm chart value
      # cpu: "2000m" # default Helm chart value
    requests:
      memory: "512Mi"
      cpu: "200m"
      # memory: "4000Mi" # default Helm chart value
      # cpu: "2000m" # default Helm chart value
```

> [!NOTE]
> If you want to control how many pods to run disable **autoscaling**.

### Install the Chart

To install the chart with the release name `zpa-connector` in the `zscaler-zpa` namespace, run the following command:

```shell
# Add the provision key as an environment variable
ZSCALER_PROVISION_KEY="1|enrollment.zpatwo.net|XXXXXX"
 
# Create the namespace and label it
kubectl create namespace zscaler-zpa
kubectl label namespace zscaler-zpa app.kubernetes.io/managed-by=Helm --overwrite
kubectl annotate namespace zscaler-zpa meta.helm.sh/release-name=zpa-connector meta.helm.sh/release-namespace=zscaler-zpa --overwrite

# Install the chart
helm install zpa-connector zpa-helm-repo/zpa-app-connector-kubernetes -n zscaler-zpa -f values.yaml --set zsglobal.provisionkey.value=$ZSCALER_PROVISION_KEY

# The -f flag is optional and can be used to override the default values in the chart.

# Upgrade
helm upgrade zpa-connector zpa-helm-repo/zpa-app-connector-kubernetes -n zscaler-zpa -f values.yaml
```

### Uninstall the Chart

To uninstall the chart with the release name `zpa-connector` from the `zscaler-zpa` namespace, run the following command:

```shell
helm uninstall zpa-connector -n zscaler-zpa
```

## ZPA App Connector on Proxmox

### Prerequisites

Retrieve the provisioning key from the [Zscaler Access Admin Portal](https://console.zscaler.com/private#connectors)

### Download and configure the ZPA connector image

All steps below are done on a Proxmox node.

```shell
# Download the ZPA connector image
wget https://dist.private.zscaler.com/vms/VMware/2024.12/zpa-connector-el9-2024.12.ova

# Extract the downloaded OVA file
tar -xvf zpa-connector-el9-2024.12.ova

# Set variables for the VM
VM_NAME=zpa-connector-01
VM_ID=8000
VM_STORAGE=local-zfs
NETWORK_BRIDGE=vmbr0
VLAN_TAG=10
OVF_FILE=$(ls -lt *.ovf | head -n 1)

# Import the ZPA connector disk
qm importovf $VM_ID $OVF_FILE $VM_STORAGE

# Set the name of the VM
qm set $VM_ID --name $VM_NAME

# Assign SCSI controller
qm set $VM_ID --scsihw pvscsi

# Assign network adapter
qm set $VM_ID --net0 virtio,bridge=$NETWORK_BRIDGE,tag=$VLAN_TAG

# Assign CPU Architecture
qm set $VM_ID --cpu cputype=x86-64-v2-AES

# Start the VM
qm start $VM_ID

# Check the status of the VM
qm status $VM_ID
```

[Continue to Enroll the App Connector](#enroll-the-app-connector)

## ZPA App Connector on Proxmox LCX

### Prerequisites

Retrieve the provisioning key from the [Zscaler Access Admin Portal](https://console.zscaler.com/private#connectors)

### Download the LCX template

All steps below are done on a Proxmox node.

```shell
pveam update
pveam available
pveam download local centos-9-stream-default_20240828_amd64.tar.xz
```

>[!TIP]
>Rocky Linux 9 has also been verified to work using this guide:  
>rockylinux-9-default_20240912_amd64.tar.xz

### Install ZPA App Connector LXC

```shell
LXC_ID=8000
HOSTNAME=zpa-connector-01
LXC_STORAGE=local-zfs
NETWORK_BRIDGE=vmbr0
DISKSIZE=8
MEMORY=1024
SWAP=512
CORES=2
VID=2
IP4=10.100.2.100/24
GW4=10.100.2.1
IP6=2001:db8:dead:beef::64/64
GW6=2001:db8:dead:beef::1
NAMESERVER=10.100.2.10,10.100.2.11
SEARCHDOMAIN=example.cloud
PASSWORD=zscaler123
pct create $LXC_ID local:vztmpl/centos-9-stream-default_20240828_amd64.tar.xz \
  --description 'Zscaler - ZPA App Connector' \
  --hostname $HOSTNAME \
  --storage $LXC_STORAGE \
  --rootfs $LXC_STORAGE:$DISKSIZE \
  --memory $MEMORY \
  --swap $SWAP \
  --cores $CORES \
  --net0 name=eth0,bridge=$NETWORK_BRIDGE,tag=$VID,ip=$IP4,gw=$GW4,ip6=$IP6,gw6=$GW6 \
  --nameserver $NAMESERVER \
  --searchdomain $SEARCHDOMAIN \
  --password $PASSWORD \
  --onboot true
```

If you're not running IPv6 in your network you can change the line `--net0` into this:  
`--net0 name=eth0,bridge=$NETWORK_BRIDGE,tag=$VID,ip=$IP4,gw=$GW4,ip6=auto`

> [!NOTE]
> To use ZPA App Connector in the container you will have to modify the configuration file.  
> Note that this will remove some security features from the LXC so use with caution!
>
> `echo "lxc.cap.drop:" >> /etc/pve/lxc/$LXC_ID.conf`

```shell
# Start the LXC container
pct start $VM_ID
```

### Add Zscaler repository

All steps below are done on the ZPA App Connector.

`sudo vi /etc/yum.repos.d/zscaler.repo`

```shell
# Content of /etc/yum.repos.d/zscaler.repo
[zscaler]
name=Zscaler Private Access Repository
baseurl=https://yum.private.zscaler.com/yum/el9
enabled=1
gpgcheck=1
gpgkey=https://yum.private.zscaler.com/yum/el9/gpg
```

```shell
# Install the ZPA App Connector package
sudo yum install zpa-connector
```

## Enroll the App Connector

All steps below are done on the ZPA App Connector.

```shell
# Add the provision key as an environment variable
ZSCALER_PROVISION_KEY="1|enrollment.zpatwo.net|XXXXXX"

# Stop the ZPA Connector
sudo systemctl stop zpa-connector

# Create the provision file with correct permissions
sudo touch /opt/zscaler/var/provision_key
sudo chmod 644 /opt/zscaler/var/provision_key
echo $ZSCALER_PROVISION_KEY | sudo tee /opt/zscaler/var/provision_key

# Start the ZPA Connector
sudo systemctl start zpa-connector
sudo systemctl status zpa-connector

# Show the logs for the ZPA Connector service
sudo journalctl -u zpa-connector -f
```
