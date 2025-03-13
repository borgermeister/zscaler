# ZPA App Connector

## ZPA App Connector on Kubernetes

### Prerequisites

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
OVF_FILE=$(ls *.ovf | head -n 1)

# Import the ZPA connector disk
qm importovf $VM_ID $OVF_FILE $VM_STORAGE

# Set the name of the VM
qm set $VM_ID --name $VM_NAME

# Assign SCSI controller
qm set $VM_ID --scsihw pvscsi

# Assign network adapter
qm set $VM_ID --net0 virtio,bridge=vmbr0,tag=10

# Assign CPU Architecture
qm set $VM_ID --cpu cputype=x86-64-v2-AES

# Start the VM
qm start $VM_ID

# Check the status of the VM
qm status $VM_ID
```

### Enroll the App Connector

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

sudo systemctl start zpa-connector
sudo systemctl status zpa-connector
```
