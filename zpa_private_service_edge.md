# ZPA Private Service Edge

## ZPA Private Service Edge on Proxmox

### Prerequisites

Retrieve the provisioning key from the [Zscaler Admin Portal](https://console.zscaler.com/private#privateBrokers)

### Download and configure the ZPA Private Service Edge image

All steps below are done on a Proxmox node.

```shell
# Elevate privileges to root
su -

# Download the ZPA Private Service Edge image
wget https://dist.private.zscaler.com/vms/VMware/2024.12/zpa-service-edge-el9-2024.12.ova

# Extract the downloaded OVA file
tar -xvf zpa-service-edge-el9-2024.12.ova

# Set variables for the VM
VM_NAME=zpa-pse-01
VM_ID=9000
VM_STORAGE=local-zfs
NETWORK_BRIDGE=vmbr0
VLAN_TAG=10
OVF_FILE=$(ls *.ovf | head -n 1)

# Import the ZPA Private Service Edge disk
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

# Remove downloaded files (optional)
rm zpa-service-edge*
```

### Enroll the Private Service Edge

```shell
# SSH into the Private Service Edge node
# Start the SSH daemon if not started (must be done via the console)
sudo systemctl start sshd

# Configure correct timezone
sudo timedatectl set-timezone Europe/Oslo

# Add the provision key as an environment variable
ZSCALER_PROVISION_KEY="1|enrollment.zpatwo.net|XXXXXX"

# Stop the ZPA Private Service Edge service
sudo systemctl stop zpa-service-edge

# Create the provision file with correct permissions
sudo touch /opt/zscaler/var/service-edge/provision_key
sudo chmod 644 /opt/zscaler/var/service-edge/provision_key
echo $ZSCALER_PROVISION_KEY | sudo tee /opt/zscaler/var/service-edge/provision_key

# Start the ZPA Private Service Edge service
sudo systemctl start zpa-service-edge
sudo systemctl status zpa-service-edge

# Show the logs for the ZPA Private Edge service
sudo journalctl -u zpa-service-edge -f
```

### Troubleshooting

```shell
# Which App Connectors is the ZPA Private Service Edge connected to
curl localhost:8500/fohh/servers?small
```
