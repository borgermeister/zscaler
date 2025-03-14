# Zscaler GRE Tunnels

## Zscaler Configuration

### Prerequisites

Find the public IP of the site sourcing the GRE tunnels.

### Add Static IP and GRE Tunnel

- **Infrastructure** -> **Internet & SaaS** -> **Traffic Forwarding** -> **Location Management** -> **Static IPs & GRE Tunnel**
	- Static IP -> Add Static IP
		- Static IP Address: `<public IP of the site>`
		- Description: `Something descriptive`
		- Selection: `Automatic`
		- Review and save the static IP configuration
	- GRE Tunnels -> Add GRE Tunnel
		- Static IP Address: `Choose the configured IP from the previous step`
		- Description: `Something descriptive`
		- Data Center: `Default values is recommended`
		- Internal IP Range: `Select a /29 prefix to be used on the GRE tunnels`
		- Take note of the Primary and Secondary Data Center VIP
		- Review and save the GRE Tunnel configuration

> [!NOTE]
> The /29 prefix you select will automatically be split into two /30 prefixes to be used on each GRE tunnel. The first IP in each prefix is assigned to your site and the last IP is used by Zscaler.

## OPNsense Configuration

### Add GRE interfaces/tunnels

You need to create two GRE interfaces/tunnels.

- **Interfaces** -> **Devices** -> **GRE** -> **Add**
	- Local address: `WAN interface`
	- Remote address: `Primary or Secondary Data Center VIP`
	- Tunnel local address: `First IP from the Internal IP Range - first or second /30 prefix`
	- Tunnel remote address: `Last IP from the Internal IP Range - first or second /30 prefix`
	- Tunnel netmask / prefix: `/30`
	- Description: `Something descriptive`

### Assign Interface

- **Interfaces** -> **Assignments** -> **Assign a new interface**
	- Device: `Choose the correct GRE interface from the list`
	- Description: `This name will be shown in the interfaces list and your firewall/NAT rules`

### Routing

There are two ways route traffic over the GRE tunnels. The best way is to edit the GRE tunnels under **System** -> **Gateways** -> **Configuration**. By lowering the **Priority** you can influence which gateway is being used. You can also monitor the peer IP on the gateway to make sure the tunnel is up.

Another option to route traffic over the GRE tunnels is to create static routes - one route for each tunnel.

> [!NOTE]
> Depending on your setup you may have to create a firewall rule allowing GRE Protocol 47 on the WAN interface.
