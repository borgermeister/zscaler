# Zscaler Source IP Anchoring (SIPA)

Some cloud applications or web services restrict access based on the source IP address of the traffic. When using Zscaler Internet Access the traffic normally has a source IP address belonging to Zscaler. Source IP Anchoring, or SIPA, allows you to anchor the source IP address to a specific range of IP addresses. This allows you to bypass the Zscaler source IP address and use your own IP address instead.  

> [!NOTE]
> To get SIPA working you need to make configuration changes to both ZPA and ZIA.

## Application Segment

Start by creating a new application segment and enable it for SIPA.  

- Policies -> Private Applications -> App Segments -> Add Application Segment
	- Name: `SIPA`
	- Source IP Anchor: `Enabled`
	- Applications: `whatismyip.akamai.com`
	- TCP Port Ranges: `80` and `443`
	- Bypass: `Use Client Forwarding Policy`
	- Add Segment Group
		- Name: `SIPA`
	- Add Server Group
		- Name: `SIPA`
		- App Connector Groups: Choose App Connector group
	- Save the new application segment

## Access Policy

Create an access policy allowing traffic from a ZIA Service Edge to the newly created application segment.  

- Policies -> Private Applications -> Policies -> Access Policy -> Add
	- Name: `SIPA`
	- Rule Action: `Allow Access`
	- Criteria
		- Application -> Segment Groups: `SIPA`
		- Client Types: `ZIA Service Edge`
	- Save the new access policy

> [!NOTE]
> This policy does not seem to work with User and Session Attributes  
> Use Forwarding Control Policy instead to allow users or groups to use SIPA

## Client Forwarding Policy

Then you need to create a new client forwarding policy to instruct the client to send the traffic to ZIA instead of ZPA.  

- Infrastructure -> Private Access -> Client Connector Policies -> Client Forwarding Policy -> Add
	- Name: `SIPA`
	- Rule Action: `Bypass ZPA`
	- Criteria
		- Application -> Segment Groups: `SIPA`
		- Client Types: `Client Connector`

## ZPA Gateway

Configure the ZPA gateway so that ZIA can route traffic via ZPA.  

- Infrastructure -> Internet & SaaS -> Networking Policies -> Forwarding Control ->  Zscaler Private Access -> Add Gateway for ZPA
	- Gateway Name: `SIPA`
	- Server Group: `SIPA`
	- Application Segment: Automatically filled in
	- Save the ZPA Gateway

## Forwarding Control

Configure the forwarding policy that instruct ZIA to route traffic via the ZPA gateway.  

- Infrastructure -> Internet & SaaS -> Networking Policies -> Forwarding Control -> Forwarding Control Policy -> Add Forwarding Rule
	- Name: `SIPA`
	- Forwarding Method: `ZPA`
	- Criteria
		- Destination -> Application Segment: `SIPA`
	- Forward to ZPA Gateway: `SIPA`
	- Save the forwarding policy

> [!IMPORTANT]
> Remember to activate your changes when you are done.
