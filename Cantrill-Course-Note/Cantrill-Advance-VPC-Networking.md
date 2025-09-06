# Cantrill Review – Advanced VPC Networking

## VPC Flow Logs

- Only capture packet **metadata** (NOT CONTENTS)
  - To capture packet contents, use a packet sniffer installed on an EC2 instance.
- Attach virtual monitors to:
  - **VPC**: All ENIs in that VPC.
  - **Subnet**: All ENIs in that subnet.
  - **Single ENI**.
- **Not Real-Time**: VPC Flow Logs are not real-time.
- **Supported Log Destinations**:
  - **S3**:
    - Access log files directly, integrate with third-party monitoring systems.
  - **CloudWatch Logs**:
    - Integrate with other products, stream data to different locations, access programmatically or via console.
- Use **Athena** to query logs in S3 (pay only for data read, schema-on-read architecture).
- **Capture Scope**: Flow logs capture metadata from the attachment point downwards (e.g., VPC → Subnets → ENIs).
- **Capture Types**:
  - ACCEPTED connections.
  - REJECTED connections.
  - ALL connections.
- **Captured Data** (Flow Log Records):
  - Source/destination address.
  - Source/destination port.
  - Protocol:
    - ICMP: 1
    - TCP: 6
    - UDP: 17
  - Action: Accepted or Rejected.
- **Exclusions**:
  - Traffic to/from 169.254.169.254 (metadata service).
  - Traffic to/from 169.254.169.123.
  - DHCP traffic.
  - Amazon DNS Server traffic.
  - Amazon Windows License traffic.

## Egress-Only Internet Gateway

- Allows **outbound-only** (and response) access to Public AWS Services and the Public Internet for **IPv6-enabled instances** or VPC-based services.
- **IPv4 vs. IPv6**:
  - IPv4: Addresses are private or public; NAT allows private IPv4 IPs to access public networks without allowing inbound connections.
  - IPv6: All IPs are public; Internet Gateway allows both inbound and outbound traffic.
- **Egress-Only Gateway**:
  - Provides outbound-only access for IPv6.
  - Attached at the **VPC level** (one per VPC, used by all AZs).
  - Add default IPv6 outbound route (`::/0`) to the route table with the Egress-Only Gateway ID (`eigw-id`) as the target.
  - **Stateful device**: Denies all inbound traffic but allows return traffic from outbound connections.
  - **Highly Available (HA)**: Scales across all AZs in the region by default.

## VPC Endpoints (Gateway)

- Provide private access to **S3** and **DynamoDB** without public addressing.
- **Mechanism**:
  - Add **prefix lists** to the route table, enabling the VPC router to direct traffic to public services via the Gateway Endpoint.
- **Highly Available**: Across all AZs in a region by default.
- **Configuration**:
  - Associated with a VPC; specify which subnets will use it.
  - Automatically configures the route table with prefix lists for S3 and DynamoDB.
- **Endpoint Policy**:
  - Controls access (e.g., restrict to specific S3 buckets).
- **Regional Limitation**: Cannot access cross-region services.
- **Use Cases**:
  - Enable private VPCs to access public resources (S3 or DynamoDB).
  - Prevent "leaky buckets" by setting S3 buckets to private, allowing access only via the Gateway Endpoint.
- **Primary Benefits**:
  - No public addressing required for S3 or DynamoDB.
  - Region-resilient by default.
  - Not accessible outside the VPC.
  - Bucket Policy can DENY traffic not originating from the Gateway Endpoint for maximum security.
- **Deployment**: Deploys instantly after creation.

## VPC Endpoints (Interface)

- Provide private access to **AWS Public Services** (historically, anything except S3 and DynamoDB, but S3 is now supported).
- **Configuration**:
  - Added to specific subnets as an **ENI** (single ENI, not highly available).
  - **Best Practice**: Use one Interface Endpoint per AZ for full HA.
- **Access Control**:
  - Network access controlled via **Security Groups**.
  - **Endpoint Policies** restrict what can be accessed.
- **Supported Protocols**: TCP, IPv4.
- **PrivateLink**:
  - Allows AWS or external services to be injected into the VPC via assigned network interfaces.
  - Useful for heavily regulated industries.
- **DNS**:
  - Provides new service endpoint DNS that resolves to the private address of the service.
  - Types: Endpoint Regional DNS, Endpoint Zonal DNS, Private DNS (default).
- **Problem Solved**:
  - Private resources (e.g., SNS) cannot be accessed without a public IP.
- **Solution**:
  - Traffic flows from a private instance in a private subnet to an Interface Endpoint in a public subnet, then to the service.
  - Private DNS associates a private Route 53 Hosted Zone with the VPC, resolving default service DNS to the Interface Endpoint IP.
- **Limitation**: Only one Interface Endpoint per AZ.
- **Deployment**: Takes a few minutes to deploy after creation.

## VPC Peering

- **Direct Encrypted Network Link**: Connects two VPCs (same/cross-region, same/cross-account).
- **Optional**: Public hostnames resolve to private IPs.
- **Security Groups**:
  - Same-region peers: SGs can reference peer SGs.
  - Cross-region peers: SGs must reference IP or IP ranges.
- **No Transitive Peering**:
  - Creates a logical gateway in each VPC; transitive peering is not supported.
- **Routing Configuration**:
  - Requires route tables on both sides to direct traffic to the remote CIDR via the peer gateway.
  - Configure subnets on both sides for open communication.
  - SGs and NACLs can filter traffic.
- **CIDR Overlap**: Peering connections cannot be created if VPC CIDRs overlap (avoid using the same address ranges in multiple VPCs).
- **Encryption**: Data transferred between VPCs is encrypted.
- **Cross-Region Peering**: Uses the AWS Global Network.