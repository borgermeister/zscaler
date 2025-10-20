# Zscaler Privilege Remote Access

## App Segments

Policies -> Private Applications -> App Segments -> App Segments -> Add Application Segment
- Name: `Borgermeister Home - Windows machines`
- Specify the FQDN of the Windows host and enable **Access Type Privileged Remote Access** for RDP
  - Choose the correct security level
- Add port `3389` under **TCP Port Ranges**
- Create a new **Segment Group** and **Server Group** if needed

# Privileged Consoles

Policies -> Clientless -> Resources -> Privileged Consoles -> Add
- Choose the correct **Portal** and **Application**
