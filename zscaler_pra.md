# Zscaler Privilege Remote Access

## App Segments

Policies -> Private Applications -> App Segments -> App Segments -> Add Application Segment
- Name: `Borgermeister Home - Windows machines`
- Specify the FQDN of the Windows host and enable **Access Type Privileged Remote Access** for RDP
  - Choose the correct **Connection Security** level (I had to choose `TLS` to get it working in my home lab)
- Add port `3389` under **TCP Port Ranges**
- Create a new **Segment Group** and **Server Group** if needed

## Privileged Consoles

Policies -> Clientless -> Resources -> Privileged Consoles -> Add
- Choose the correct **Portal** and **Application**

## Access Policy

Policies -> Private Applications -> Policies -> Access Policy -> Add
- Name: `Borgermeister PRA - Windows machines`
- Rule Action: **Allow Access**
- Traffic Steering
  - App Connector Selection Method: **All App Connector groups for the application**
- Criteria
  - Choose the correct **Applicatin Segments** or **Segment Groups**
  - Choose the correct **User and Session Attributes**
  - Client Types: **Web Browser**
