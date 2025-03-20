# ZPA Private Service Edge

## Prerequisites

Retrieve the provisioning key from the [Zscaler Admin Portal](https://console.zscaler.com/private#privateBrokers)

## ZPA Private Service Edge on Proxmox

All steps below are done on a Proxmox node.

### Download and configure the ZPA Private Service Edge image

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

## ZPA Private Service Edge on Proxmox LXC

All steps below are done on a Proxmox node.

### Download the LCX template

```shell
pveam update
pveam available
pveam download local centos-9-stream-default_20240828_amd64.tar.xz
```

### Install ZPA Private Service Edge LXC

```shell
LXC_ID=9000
HOSTNAME=zpa-pse-01
LXC_STORAGE=local-zfs
NETWORK_BRIDGE=vmbr0
DISKSIZE=8
MEMORY=2048
SWAP=512
CORES=2
VID=2
IP4=10.100.2.253/24
GW4=10.100.2.1
IP6=2a0b:4e07:c25:ff02::fd/64
GW6=2a0b:4e07:c25:ff02::1
NAMESERVER=10.100.2.10,10.100.2.11
SEARCHDOMAIN=home.borgermeister.cloud
PASSWORD=zscaler123
pct create $LXC_ID local:vztmpl/centos-9-stream-default_20240828_amd64.tar.xz \
  --description 'Zscaler - ZPA Private Service Edge' \
  --hostname $HOSTNAME \
  --storage $LXC_STORAGE \
  --rootfs $LXC_STORAGE:$DISKSIZE \
  --memory $MEMORY \
  --swap $SWAP \
  --cores $CORES \
  --net0 name=eth0,bridge=$NETWORK_BRIDGE,tag=$VID,ip=$IP4,gw=$GW4,ip6=$IP6,gw6=$GW6 \
  --nameserver $NAMESERVER \
  --searchdomain $SEARCHDOMAIN \
  --password $PASSWORD
```

If you're not running IPv6 in your network you can change the line `--net0` into this:  
`--net0 name=eth0,bridge=$NETWORK_BRIDGE,tag=$VID,ip=$IP4,gw=$GW4,ip6=auto`

> [!NOTE]
> To use ZPA Private Service Edge in the container you will have to modify the configuration file.  
> Note that this will remove some security features from the LXC so use with caution!
>
> `echo "lxc.cap.drop:" >> /etc/pve/lxc/$LXC_ID.conf`

```shell
# Start the LXC container
pct start $LXC_ID
```

### Add Zscaler repository

All steps below are done on the ZPA Private Service Edge.

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
# Install the ZPA Private Service Edge package
sudo yum install zpa-service-edge
```

## Enroll the Private Service Edge

All steps below are done on the ZPA Private Service Edge.

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
