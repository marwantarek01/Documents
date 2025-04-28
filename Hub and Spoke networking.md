# Hub and Spoke networking documentation

The Hub-and-Spoke architecture is a network design model where a central Hub VCN connects to multiple Spoke VCNs through a Dynamic Routing Gateway (DRG). In this document it is used to route traffic from Spoke VCNs through a centralized network firewall deployed in the Hub. By forwarding traffic through the Hub, organizations can enforce consistent security policies, perform deep packet inspection, and centralize traffic monitoring. 
![network architecture](https://github.com/marwantarek01/assets/blob/main/Network%20Architecture%201.png)
## Table of Contents  

  - [Hub](#hub)
    - [Ingress route table](#ingress-route-table)
    - [Hub VCN subnets route tables](#hub-vcn-subnets-route-tables)
    - [Firewall subnet route table](#firewall-subnet-route-table)
  - [Spoke](#spoke)
    - [Spoke subnets route table](#spoke-subnets-route-table)
  - [Dynamic Routing Gateway](#dynamic-routing-gateway)
    - [DRG Attachment Main Components](#drg-attachment-main-components)
    - [DRG Hub attachment Configuration](#drg-hub-attachment-configuration)
    - [DRG Spoke attachment Configuration](#drg-spoke-attachment-configuration)
## Hub  
The hub is a central Virtual Cloud Network (VCN) that serves as a shared point for managing the network firewall.
In the following section, a detailed explanation is provided for the route tables configured to ensure that all traffic flows through the network firewall, including inter-subnet communication within the hub or spoke VCNs.

### Ingress route table
- The ingress route table is created at the VCN level. Its purpose is to forward all incoming traffic from spoke VCNs to the hub VCN’s network firewall subnet, ensuring that all traffic is inspected by the firewall before reaching its destination.

- Dynamic Routing Gateway (DRG) acts as a central gateway between the spoke and hub vcn, and a detailed explanation of this flow will be provided later on in this guide.

- ### Route table

    | Destination  | Target Type | network_entity_id | Description                                |
    |--------------|-------------|-------------------|--------------------------------------------|
    | 0.0.0.0/0    | CIDR_BLOCK  |  firewall_id      | Route all incoming traffic in the Hub VCN to the firewall |
    | var.public-sn| CIDR_BLOCK  | firewall_id       | Route traffic with **public-sn** destination to the firewall      |
    |var.private-sn| CIDR_BLOCK  | firewall_id       | Route traffic with **private-sn** destination to the firewall     |

   `network_entity_id` in Terraform defines the **target** resource  that traffic should be routed to.
In the OCI Console, this corresponds to the target ip address.
### Hub VCN subnets route tables
  To handle intercommunication between subnets within the Hub VCN, we need to override the default routing behavior (where subnets in the same VCN communicate directly with each other). This can be achieved by configuring the route tables to forward all traffic to the firewall subnet. After the firewall inspects the traffic, it will route it to the original destination subnet. 
  
  this configuration is applied to all subnets in the Hub VCN (except the network firewall subnet)
  
- ### Route table

    | Destination      | Target Type | network_entity_id | Description                                                   |
    |------------------|-------------|-------------------|---------------------------------------------------------------|
    | 0.0.0.0/0        | CIDR_BLOCK  | firewall_id       | Route incoming traffic to the firewall                        |
    | var.public-sn    | CIDR_BLOCK  | firewall_id       | Route traffic with **public-sn** destination to the firewall  |
    | var.private-sn   | CIDR_BLOCK  | firewall_id       | Route traffic with **private-sn** destination to the firewall |

- Variables like `var.public-sn` is used instead of hardcoding the CIDR blocks of subnets in the Terraform files.


### Firewall subnet route table
All routes to destination subnets are defined in the firewall subnet's route table.
This is because, as explained, all traffic is forwarded to the firewall.
Therefore, the firewall must have routes to all destination subnets to ensure proper traffic flow.

- ### Route table

    | Destination    | Target Type | network_entity_id   | Description                              |
    |----------------|-------------|---------------------|------------------------------------------|
    | 0.0.0.0/0      | CIDR_BLOCK  | natgw-id            | route internet traffic to nat gateway    |
    |all-ruh-services| CIDR_BLOCK  | sgw-id            | route internet traffic to service gateway  |
    |`prod-CIDR`     | CIDR_BLOCK  | drg-id              | route prod traffic to DRG                |
    |`dev-CIDR`      | CIDR_BLOCK  | drg-id              | route dev traffic to DRG                 |
    | var.public-sn  | CIDR_BLOCK  | var.public-sn       | Route traffic to **public-sn** (in hub vcn) destination  |
    | var.private-sn | CIDR_BLOCK  | var.private-sn      |Route traffic to **private-sn** (in hub vcn) destination   |

   The last 2 rules are used for handling intercommunication trrafic


## Spoke  

### Spoke subnets route table
  All traffic originating from subnets within the spoke VCNs — including inter-subnet traffic — should be routed to the DRG, which then forwards it to the Network Firewall through the hub-attachment.
- ### Route table

    | Destination    | Target Type | network_entity_id   | Description                         |
    |----------------|-------------|---------------------|-------------------------------------|
    | 0.0.0.0/0      | CIDR_BLOCK  | drg-id              | route  traffic to DRG               |
    |var.db_sn       | CIDR_BLOCK  | drg-id              | route db (in dev vcn) traffic to DRG  |


## Dynamic Routing Gateway  
 DRG (Dynamic Routing Gateway) is a virtual router that serves as a central connection point for Virtual Cloud Networks (VCNs) in a hub-and-spoke architecture. It connects to both the hub and spoke VCNs through **DRG attachments**.
  
The main components and configuration details of DRG Attachments will be explained in the following section.
### DRG Attachment Main Components 

- #### DRG Route table
    A DRG Route Table is used to control how traffic flows between DRG attachments (where DRG should forward the traffic from a specific attachment).
    
- #### VCN Route table
    Controls how traffic from the DRG enters the VCN (i.e., how traffic arriving via the DRG is routed inside the VCN). It is used to enforce trrafic to a specfic subent (i.e, network firewall subnet).
    


   **Comparison of DRG Route Table vs. VCN Route Table**

    | Section             | Applies To              | Direction      | Controls Where Traffic Goes     |
    |---------------------|--------------------------|----------------|----------------------------------|
    | **DRG Route Table** | DRG side of attachment   | Outbound       | From VCN to other attachments    |
    | **VCN Route Table** | VCN side of attachment   | Inbound        | From DRG into VCN’s subnets      |

- #### Import Route distribution
  Instead of manually defining static route rules in the DRG Route Table, the table can be configured to dynamically learn routes by selecting a Route Distribution. This enables the DRG to automatically import routes from one or more attachments.  

  Import Route Distribution configuration includes **Route Distribution Statements**, which serve as policy rules that define the specific routes to be learned from designated DRG attachments.

### DRG Hub attachment Configuration 
- Regarding Hub attachment creation, the default `Autogenerated DRG Route Table for VCN Attachments` is used instead of manually creating a route table. It includes an import route distribution, ```Autogenerated Import Route Distribution for ALL Routes```, with 'Match All' route statements. So, it imports all routes from all drg attachments.
   
    This configuration is required for the hub attachment route table in order to **dynamically** learn all propagated routes, enabling successful traffic forwarding after firewall inspection.

    #### DRG Hub attachment Route table
    | Destination    | Target Type | Next hop attachment name   | Description                  |
    |----------------|-------------|----------------------------|------------------------------|
    | 0.0.0.0/0      | CIDR_BLOCK  |  hub-attachment            | Route traffic to the hub vcn |
    |`hub-vcn cidr` | CIDR_BLOCK   |  hub-attachment            | Route traffic to the hub vcn |
    |`dev-vcn cidr`  | CIDR_BLOCK  |  dev-attachment            | Route traffic to the dev vcn |
    |`prod-vcn cidr`| CIDR_BLOCK   |  prod-attachment           | Route traffic to the prod vcn|

    the first rule is learned from vcn ingress route table 


- Select the VCN **Ingress Route** Table as the `VCN Route Table` in the Hub Attachment configuration. As previously explained, the VCN Ingress Route Table handles inbound traffic entering the Hub VCN, ensuring that all incoming traffic is correctly forwarded to the firewall for inspection.
 
 
### DRG Spoke attachment Configuration
 For the Spoke VCNs, DRG route table is created, intended to forward traffic entering the DRG from Spoke VCN toward the firewall subnet in the Hub VCN, this can be acheived by defining `Import Route Distribution` with a `Route Distribution Statement` that explicitly accepts routes from the **Hub-Attachment**. 
   
   This setup ensures that the Spoke VCN's DRG attachment can only learn and import routes propagated from the Hub attachment, enforcing trrafic forwarding to the network firewall through the hub-attachment.
    #### DRG Spoke attachment Route table
    | Destination    | Target Type | Next hop attachment name   | Description                  |
    |----------------|-------------|----------------------------|-------------------------------|
    | 0.0.0.0/0      | CIDR_BLOCK  |  hub-attachment            | Route traffic to the hub vcn    |
    |`hub-vcn cidr` | CIDR_BLOCK  |  hub-attachment            | Route traffic to the hub vcn |



