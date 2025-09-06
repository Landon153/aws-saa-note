# Cantrill Review – Hybrid Environments and Migration

## Border Gateway Protocol (BGP) 101
- **Autonomous System (AS)**: Routers controlled by one entity – a network in BGP.
- **Autonomous System Number (ASN)**:
  - Unique and allocated by IANA.
  - Range: 0-65535.
  - 64512–65534 are private (can be used in private peering without official allocation).
- BGP operates over **TCP/179** – reliable protocol.
- Peering is manually configured, not automatic.
- **Path-Vector Protocol**:
  - Exchanges the best path to a destination between peers (ASPATH).
- **iBGP**: Internal BGP, routing within an AS.
- **eBGP**: External BGP, routing between ASs.
- **ASPATH**:
  - “I” indicates the origin (network learned from a locally connected network).
  - BGP exchanges the shortest ASPATH between peers.
  - Example: Brisbane -> Alice Springs defaults to a 5 Mbps satellite link (shorter path) over a 1 Gbps fiber link (longer path).
  - Shortest route chosen, not necessarily the most performant or highest throughput.
- **ASPATH Prepending**:
  - Artificially lengthens a path to influence route preference.
  - Example: Satellite path (ASPATH: 202,202,202,I) vs. fiber path (201,202,I).
- An AS advertises all known shortest paths to peers, prepending its own ASN to the path to create a source-to-destination path.

## IPSEC VPN Fundamentals
- **IPSEC**: Group of protocols for secure tunnels across insecure networks.
  - Between two peers (local and remote).
  - Provides authentication, encryption, and visibility to modifications.
- **IPSEC Tunnels**:
  - Created as needed for “Interesting Traffic” (traffic matching rules, e.g., network prefixes or complex types).
  - Tunnels are torn down and rebuilt when interesting traffic is detected.
- **Two Phases**:
  - **IKE Phase 1 (Slow & Heavy)**:
    - Authenticates using pre-shared key or certificate.
    - Uses asymmetric encryption to create a shared symmetric key.
    - Creates IKE Security Association (SA) – Phase 1 tunnel.
  - **IKE Phase 2 (Fast & Agile)**:
    - Uses keys from Phase 1 to agree on encryption methods and keys for bulk data transfer.
    - Creates IPSEC SA – Phase 2 tunnel (runs over Phase 1).
    - Allows tunnels to be torn down and rebuilt for interesting traffic.
- **IKE Phase 1 – How It Works**:
  - **Diffie-Hellman Key Exchange (DH)**:
    - Each side creates a DH private key and derives a public key.
    - Public keys are exchanged over the public internet.
    - Each side uses its private key and the peer’s public key to generate the same shared DH key.
    - DH key used to exchange key material and agreements, generating a symmetric key for Phase 1 tunnel encryption.
- **IKE Phase 2 – How It Works**:
  - Uses DH key and symmetric Phase 1 tunnel for fast setup.
- **Policy-Based VPN**:
  - Rule sets match traffic to a pair of SAs.
  - Different rules/security settings for different traffic types.
- **Route-Based VPN**:
  - Matches traffic to prefixes with a single pair of SAs.
  - Simpler setup, less functionality.

## AWS Site-to-Site VPN
- Logical connection between a VPC and an on-premises network, encrypted using IPSEC over the public internet.
- **Full HA**: Achievable with correct design and implementation.
- **Quick Provisioning**: Less than an hour.
- **Key Components**:
  - VPC.
  - Virtual Private Gateway (VGW).
  - Customer Gateway (CGW):
    - Logical (AWS side).
    - Physical device (on-premises).
  - VPN Connection between VGW and CGW.
- **Static vs. Dynamic VPN (BGP)**:
  - **Static**:
    - Routes for remote side (AWS) added to route tables as static routes.
    - Networks for remote site statically configured on the VPN connection (CGW).
    - No load balancing or multi-connection failover.
  - **Dynamic**:
    - Routes can be added statically or via route propagation (automatically adds routes to AWS route tables).
    - BGP configured on both customer and AWS sides using ASN.
    - Networks exchanged via BGP.
    - Multiple VPN connections provide HA and traffic distribution.
- **VPN Considerations**:
  - **Speed Limit**: 1.25 Gbps.
  - **Latency**: Inconsistent due to public internet.
  - **Cost**: AWS hourly cost, GB out cost, on-premises data cap.
  - **Setup Speed**: Hours (software configuration).
  - Can be used as a backup for Direct Connect (DX) or while waiting for DX setup (up to a month).

## Direct Connect (DX) Concepts
- **Physical Connection**: 1, 10, or 100 Gbps.
- **Path**: Business Premises -> DX Location -> AWS Region.
- **Port Allocation**: At a DX location with hourly port cost and outbound data transfer (inbound is free).
- **Provisioning Time**: Weeks for physical cable installation, no resilience by default.
- **Benefits**: Low and consistent latency, high speeds.
- **Access**: AWS Private Services (VPCs) and Public Services without internet.
- **DX Location**:
  - Not owned by AWS, located in large regional centers.
  - AWS has space and equipment.
  - COMMS providers can extend DX ports to business premises.
- **Cross Connect**: Links AWS port to customer/partner port.

## Direct Connect (DX) Resilience and HA
- **AWS Regions**: Multiple DX locations, typically in major metro data centers.
- **Connections**: DX locations connected to AWS Region via redundant high-speed connections.
- **Single Points of Failure** (default):
  - DX Location, DX Router, Cross Connect, Customer DX Router, Extension, Customer Premises, Customer Router.
- **HA Configuration**:
  - Use 2 DX Routers, 2 Customer DX Routers, 2 Customer Premises Routers.
  - Resilient against hardware failure in one path.
  - **Remaining SPOFs**: DX Location, Customer Premises, extension link path (if telco uses the same cable path).
- **Enhanced HA**:
  - Provision two separate customer premises with two DX Customer Premises Routers, connecting to two DX Locations using two Customer DX Routers and two extension links to AWS DX Routers.
  - Protects against location or hardware failure, operates at reduced resiliency during outages.
  - Risk remains if an entire location and a router in the other location/customer premises fail.
- **Key Takeaways**:
  - DX is not HA unless architected redundantly.
  - Site-to-Site VPN can serve as a DX backup.

## Direct Connect (DX) + Public VIF + VPN
- Combines to provide end-to-end IPSEC encrypted tunnel access to private VPC resources.
- **Public VIF + VPN**:
  - Encrypted and authenticated tunnel over DX (low and consistent latency).
  - Uses Public VIF + VGW/TGW Public Endpoints.
  - Transit agnostic (DX or public internet).
  - End-to-end (CGW to TGW/VGW); MACsec is single-hop based.
  - Wider vendor support than MACsec.
  - More cryptographic overhead than MACsec, limiting speeds.
  - Can be used during DX provisioning or as a backup.
- IPSEC does not compete with MACsec.

## AWS Transit Gateway
- Simplifies networking between VPCs, VPN, and Direct Connect.
- Supports transitive routing between VPCs (same/different account, same/different region).
- **Network Transit Hub**: Connects VPCs to on-premises networks.
- **Single Network Object**: HA and scalable.
- **Attachments**:
  - VPC, Site-to-Site VPN, Direct Connect Gateway.
- **VPC Attachments**: Configured with a subnet in each AZ where service is required.
- **Customer Gateway**: Enables bi-directional transitive routing between on-premises and multiple AWS VPCs.
- **Peering**:
  - Peer with other TGWs in other accounts/regions.
  - Cross-region data uses AWS Global Network (not public internet).
  - Supports same/cross-account peering.
- **Integration**: Connects with Direct Connect Gateway using a transit VIF.
- **Routing**: Default route table (RT) provided; supports multiple RTs for complex routing.
- **Considerations**:
  - Supports transitive routing.
  - Enables global networks.
  - Share TGW between accounts using AWS RAM.
  - Reduces complexity compared to VPC peering.

## Storage Gateway
- Integrates local infrastructure with AWS storage (S3, EBS Snapshots, S3 Glacier).
- **Form**: Virtual Machine or hardware appliance.
- **Protocols**: iSCSI, NFS, SMB.
- **Use Cases**: Migrations, extensions, storage tiering, disaster recovery (DR), backup system replacement.
- **Modes**:
  - **Volume Stored**:
    - All data stored locally, copied to S3 as EBS snapshots (constantly).
    - Use cases: Full disk backups, DR (create EBS volumes in AWS), low-latency access.
    - Limitation: Does not improve on-premises datacenter capacity (main data on gateway).
    - Limits: 32 volumes per GW, 16 TB per volume, 512 TB total per GW.
  - **Volume Cached**:
    - Data stored in AWS S3 (AWS-managed, visible only via Storage GW console, raw block storage).
    - Can create EBS snapshots.
    - Appears on-premises, cached locally.
    - Benefits: Datacenter capacity extension, data stored in AWS.
    - Limits: 32 volumes per GW, 32 TB per volume, 1 PB total per GW.

## Storage Gateway Tape – VTL Mode
- **VTL (Virtual Tape Library)**:
  - Replaces traditional tape backup with AWS storage.
  - **Traditional Tape Backup**:
    - Large backups stored on tapes (e.g., LTO-9: 24 TB raw, up to 60 TB compressed).
    - Components: Drives, Library, Robots (Media Changer), Tape Shelf (offsite storage).
    - Tapes not in the library cannot be used; transport to offsite storage takes time and costs.
  - **AWS Configuration**:
    - On-premises: Backup Server, Media Changer, Tape Drive(s), Upload Buffer, Local Cache.
    - AWS: Virtual Tape Library (VTL) in S3, Tape Shelf (VTS) in S3 Glacier or Deep Archive.
    - Capacity: Virtual tapes (100 GB to 5 TiB), 1 PB across 1500 virtual tapes in S3, unlimited in VTS.
  - **Capabilities**:
    - Maintain or extend existing backup systems using AWS storage.
    - Migrate backups to AWS storage.

## Storage Gateway (File)
- Bridges local file storage (NFS/SMB) with S3.
- **Features**:
  - Supports multi-site, maintains storage structure.
  - Integrates with AWS services and S3 Object Lifecycle Management.
  - Mount points (shares) via NFS (Linux) or SMB (Windows), mapped to S3 buckets.
  - Files stored in mount points appear as S3 objects.
  - Read/write caching for LAN-like performance.
  - Primary data in S3.
  - 10 bucket shares per Storage GW.
  - Supports AWS services (Athena, S3 Events, Lambda, etc.).
  - Multiple contributors (on-premises locations) supported.
  - Use **NotifyWhenUploaded API** to notify other gateways of object changes.
  - No Object Locking; new files overwrite previous versions (use read-only mode or tight access control).
  - Supports cross-region replication with multiple File GWs and S3 buckets.
  - Integrates with S3 Lifecycle Management for storage class transitions.

## Snowball / Snowball Edge / Snowmobile
- **Purpose**: Move large amounts of data IN and OUT of AWS.
- **Form**: Physical storage (suitcase for Snowball/Snowball Edge, truck for Snowmobile).
- **Process**:
  - Order empty from AWS, load data, return.
  - Order with data from AWS, empty, return.
- **Exam Note**: Know when to use each.

## DataSync
- Orchestrates large-scale data movement (amounts or files) from on-premises NAS/SAN to AWS or vice versa.
- **Use Cases**: Migrations, data processing transfers, archival/cost-effective storage, DR/BC.
- **Features**:
  - Scalable: 10 Gbps per agent (~100 TB/day).
  - Bandwidth limiters to avoid link saturation.
  - Incremental and scheduled transfers.
  - Compression and encryption.
  - Automatic recovery from transit errors.
  - Integrates with S3, EFS, FSx.
  - Pay-per-use (per GB cost).
- **Components**:
  - **Task**: Defines what, how quickly, and from/to where data is synced.
  - **Agent**: Software for reading/writing to on-premises data stores via NFS or SMB.
  - **Location**: Source and destination (NFS, SMB, EFS, FSx, S3).
- **Operation**:
  - DataSync Agent (on-premises, runs on virtualization platforms like VMware) communicates with AWS DataSync Endpoint.
  - Encryption in transit (TLS).
  - Schedules avoid or target specific time periods.
  - Bandwidth limits (MiB/s) minimize customer impact.

## FSx for Windows File Server
- Advanced shared file system accessible over SMB, integrates with Active Directory (managed or self-hosted).
- **Features**:
  - Fully managed native Windows file servers/shares.
  - Designed for Windows environments.
  - Integrates with Directory Service or self-managed AD.
  - Single or Multi-AZ within a VPC.
  - On-demand and scheduled backups.
  - Accessible via VPC, Peering, VPN, Direct Connect.
  - Supports deduplication, Distributed File System (DFS), KMS at-rest encryption, enforced in-transit encryption.
  - Volume shadow copies for file-level versioning.
- **Performance**:
  - 8 MB/s to 2 GB/s.
  - 100K’s IOPS.
  - <1 ms latency.
- **Key Features**:
  - VSS for user-driven restores.
  - Native Windows file system.
  - Windows permission model.
  - Supports DFS for scale-out file share structure.
  - Managed, no file server administration.
  - Integrates with Directory Service and self-managed AD.
- **Example Access Path**: `\\fs-xxx123.animals4life.org\catpics`.

## FSx for Lustre
- Designed for High-Performance Computing (HPC) scenarios (e.g., Big Data, Machine Learning, Financial Modeling).
- **Features**:
  - Managed Lustre for Linux clients (POSIX).
  - 100s GB/s throughput, sub-millisecond latency.
  - **Deployment Types**:
    - **Persistent**: Long-term, HA in one AZ, self-healing.
    - **Scratch**: Short-term, no replication, optimized for speed.
  - Accessible over VPN or Direct Connect.
  - Data resides in the file system during processing, lazy-loaded from S3 (linked repository), exportable to S3 via `hsm_archive`.
  - Metadata on Metadata Targets (MST), objects on Object Storage Targets (OSTs, 1.17 TiB).
  - **Performance**:
    - Scratch: 200 MB/s per TiB.
    - Persistent: 50, 100, or 200 MB/s per TiB.
    - Both support burst up to 1,300 MB/s per TiB (credit system).
  - **Size**: Minimum 1.2 TiB, increments of 2.4 TiB.
  - **Operation**: Runs from 1 AZ, 1 ENI, 1 VPC.
  - **Key Features**:
    - Scratch: Pure performance, short-term, no HA/replication.
    - Persistent: Replication within one AZ, auto-heals on hardware failure.
    - Backup to S3 (manual or automatic, 0-35 day retention).

## AWS Transfer Family
- Managed file transfer service for S3 and EFS.
- **Protocols**:
  - FTP: Unencrypted file transfer.
  - FTPS: File transfer with TLS encryption.
  - SFTP: File transfer over SSH.
  - AS2: Structured B2B data.
- **Identities**: Service-managed, Directory Service, Custom (Lambda/API Gateway).
- **Managed File Transfer Workflows (MFTW)**: Serverless file workflow engine.
- **Endpoint Types**: Public, VPC-Internet, VPC-Internal.
- **Key Features**:
  - Multi-AZ, resilient, and scalable.
  - Cost: Per-hour provisioned server + data transferred.
  - FTP/FTPS: Supports Directory Service or Custom IDP only, VPC only (no public).
  - AS2: VPC-Internet/Internal only.
  - Integrates with existing workflows or creates new ones with MFTW.