# Cantrill Review – Regional and Global Architecture

## Global
- Global Service Location & Discovery
- Content Delivery (CDN) and optimization
- Global health checks & Failover

## Regional
- Regional entry point
- Scaling & Resilience
- Application services & components

## ELB Evolution
- **Three Types of Load Balancers (ELB)** available within AWS:
  - Split between v1 (avoid/migrate) and v2 (prefer).
- **Classic Load Balancer (CLB)** – v1, introduced in 2009:
  - Not truly Layer 7, lacks features, supports only 1 SSL certificate per CLB.
- **Application Load Balancer (ALB)** – v2, supports HTTP/HTTPS/WebSocket.
- **Network Load Balancer (NLB)** – v2, supports TCP, TLS, UDP.
- **V2 Benefits**: Faster, cheaper, supports target groups and rules.

## Elastic Load Balancer (ELB) Architecture
- **Two Types of ELB**:
  - Internet-facing
  - Internal Only
- **DNS Configuration**: Each ELB is configured with an (A) record DNS name that resolves to ELB Nodes.
- **Multi-AZ**: Configured to run in 2+ Availability Zones (AZs). 1+ Nodes are placed in a subnet in each AZ and scale with load.
- **Nodes and Listeners**: Nodes are configured with listeners that accept traffic on a specific port and protocol, then communicate with targets on a specified port and protocol.
- **IP and Subnet Requirements**: ELB uses 8 IPs per subnet and requires a /27 (or /28*) subnet to allow for scaling.
- **Internet-facing Nodes**: Have public IPv4 IPs.
- **Internal Only Nodes**: Have private IPs.
- **Internal Load Balancers**: Enable scaling between application tiers.
  - Example: Users connect to the Load Balancer via (A) record DNS name to nodes in public (or private) subnets. Traffic is distributed to public (or private) EC2 web servers, then routed to an internal-only ELB, which distributes traffic to private EC2 instances or other resources in private subnets.

## ELB Key Facts
- ELB is a DNS A Record pointing at 1+ Nodes per AZ.
- Nodes in one subnet per AZ can scale.
- Internet-facing means nodes have public IPv4 IPs.
- Internal means private IPs only.
- EC2 doesn’t need to be public to work with a Load Balancer.
  - EC2 doesn’t require a public IP to receive traffic from an internet-facing LB.
- Listener Configuration controls what the LB does.
- Requires 8+ free IPs per subnet and a /27 (or /28*) subnet to allow scaling.

## Application and Network Load Balancer (ALB vs NLB)

### Load Balancer Consolidation
- **CLBs (v1)**: Don’t scale well; each unique HTTPS name requires a separate CLB due to lack of SNI support.
- **V2 Load Balancers**: Support rules and target groups. Host-based rules using SNI with an ALB allow consolidation.

### Application Load Balancer (ALB)
- **Layer 7 Load Balancer**: Listens on HTTP and/or HTTPS.
- **Unsupported Protocols**: No other Layer 7 protocols (e.g., SMTP, SSH, Gaming).
- **No TCP, UDP, or TLS Listeners**.
- **Layer 7 Features**: Supports content type, cookies, custom headers, user location, and app behavior for advanced traffic inspection (rule-based configuration).
- **HTTPS Handling**: Always terminates SSL/TLS on the ALB (no broken SSL).
  - A new connection is made to the application.
  - **Security Risk**: No end-to-end in-flight encryption.
- **SSL Requirement**: ALB must have SSL certificates if HTTPS is used.
- **Performance**: Slower than NLB due to processing more network stack layers.
- **Health Checks**: Evaluates application health at Layer 7.

### ALB Rules
- Rules direct connections arriving at a listener.
- Processed in priority order.
- **Default Rule**: Acts as a catch-all.
- **Rule Conditions**: host-header, http-header, http-request-method, path-pattern, query-string, source-ip.
- **Actions**: forward, redirect, fixed-response, authenticate-oidc, authenticate-cognito.

### Network Load Balancer (NLB)
- **Layer 4 Load Balancer**: Supports TCP, TLS, UDP, TCP_UDP.
- **No HTTP/HTTPS Awareness**: No visibility into headers, cookies, or session stickiness.
- **Performance**: Extremely fast, supports millions of requests per second (mps), ~25% of ALB latency.
- **Supported Protocols**: SMTP, SSH, Game Servers, Financial apps (non-HTTP/S).
- **Health Checks**: Only checks ICMP/TCP handshake, not application-aware (Layer 7).
- **Static IPs**: Supports static IPs, useful for whitelisting.
- **End-to-End Encryption**: Forwards TCP to instances with unbroken encryption.
- **PrivateLink**: Used to provide services to other VPCs.

### ALB vs NLB
- **Unbroken Encryption**: NLB.
- **Static IP for Whitelisting**: NLB.
- **Fastest Performance**: NLB (millions rps).
- **Non-HTTP/HTTPS Protocols**: NLB.
- **PrivateLink**: NLB.
- **Otherwise**: Use ALB.

## Launch Configurations and Launch Templates – EC2 Feature
- **LC and LT Key Concepts**:
  - Define EC2 instance configuration in advance: AMI, Instance Type, Storage, Key Pair, Networking, Security Groups, User Data, IAM Role.
  - **Not Editable**: Both are defined once; Launch Templates (LT) support versioning.
  - **LT Features**: Supports T2/T3 Unlimited, Placement Groups, Capacity Reservations, Elastic Graphics.
  - **Launch Configurations (LC)**: Only for configuring Auto Scaling Groups (ASGs).
  - **Launch Templates (LT)**: Can configure ASGs and EC2 instances, saving time when provisioning via console UI or CLI.

## Gateway Load Balancer (GWLB)
- Helps run and scale 3rd-party appliances (e.g., firewalls, intrusion detection/prevention systems).
- Supports **inbound and outbound traffic** for transparent inspection and protection.
- **GWLB Endpoints**: Traffic enters/leaves via these endpoints.
- **Balancing**: GWLB balances traffic across multiple backend appliances.
- **GENEVE Protocol**: Tunnels traffic and metadata, keeping original packets unaltered.
- **Flow Stickiness**: Ensures one flow always uses the same appliance.
- **Traffic Flow**:
  - **Inbound**: Client traffic enters via the ingress gateway route table, directing traffic to the GWLB Endpoint in the same AZ. Traffic flows to the GWLB (Security VPC), then to backend security appliances. Traffic returns via GENEVE encapsulation to the GWLB, through the GWLB Endpoint, to the ALB, and to the application.
  - **Outbound**: Return traffic from the application goes to the ALB, then to the GWLB Endpoint, GWLB (Security VPC), backend appliances, back to the GWLB, GWLB Endpoint, and out to the Internet Gateway (IGW) to the client.
  - Ensures original source and destination persistence.
- **Benefits**: Transparent, inline network security, scalable, resilient, and abstract.