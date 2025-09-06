# Cantrill Review – Security, Deployment, & Operations

## AWS Secrets Manager vs Parameter Store
- **Primary Difference**: Secrets Manager periodically triggers a Lambda function to automatically rotate secrets.
- **AWS Secrets Manager**:
  - Shares functionality with Parameter Store.
  - Designed for secrets (passwords, API keys).
  - Usable via Console, CLI, API, or SDKs (integration).
  - Supports **automatic rotation** using Lambda.
  - Directly integrates with AWS products (e.g., RDS).
  - Encrypted at rest.
  - IAM access control.
  - Supports auto-rotation.

## Application Layer (L7) Firewall
- A **Web Application Firewall (WAF)** that protects web applications or APIs against common web exploits and bots that may:
  - Affect availability.
  - Compromise security.
  - Consume excessive resources.

## AWS Shield
- **AWS Shield Standard & Advanced**: DDoS protection.
  - **Shield Standard**: Free for all AWS customers.
  - **Shield Advanced**: Costs $3,000/month (per organization, 1-year lock-in) + data transfer out costs.
- **Attack Types**:
  - **Network Volumetric Attacks (L3)**: Saturate capacity.
  - **Network Protocol Attacks (L4)**: TCP SYN Flood, leaving connections open to prevent new ones.
  - **Application Layer Attacks (L7)**: e.g., web request floods (`query.php?search=all_the_cat_images_ever`).
- **Shield Standard**:
  - Free, protects at the perimeter (Region/VPC or AWS Edge).
  - Defends against common L3/L4 attacks.
  - Best protection with Route 53, CloudFront, and AWS Global Accelerator.
- **Shield Advanced**:
  - Protects:
    - CloudFront.
    - Route 53.
    - Global Accelerator.
    - Resources with Elastic IPs (EIPs) – EC2.
    - ALBs, CLBs, NLBs.
  - Not automatic; must be explicitly enabled in Shield Advanced or AWS Firewall Manager Advanced Policy.
  - **Cost Protection**: Covers EC2 scaling for unmitigated attacks.
  - **Proactive Engagement**: AWS Shield Response Team (SRT).
  - **Key Features**:
    - WAF integration (includes basic AWS WAF fees for web ACLs, rules, and web requests).
    - L7 DDoS protection (uses WAF).
    - Real-time visibility of DDoS events and attacks.
    - Health-based detection (application-specific checks for proactive engagement).
    - Protection groups.

## CloudHSM vs. KMS
- **KMS**: AWS-managed, shared but separated.
- **CloudHSM**: True single-tenant Hardware Security Module (HSM).
  - AWS-provisioned, fully customer-managed.
  - Fully FIPS 140-2 Level 3 (KMS is Level 2 overall, some Level 3).
- **CloudHSM APIs**: Industry-standard (PKCS#11, Java Cryptography Extensions (JCE), Microsoft CryptoNG (CNG)).
- **KMS Integration**: KMS can use CloudHSM as a custom key store.
- **Operation**:
  - HSMs run in an AWS-managed HSM VPC with ENIs added to customer VPC.
  - Keys and policies stay in sync when nodes are added/removed.
  - EC2 instances require AWS CloudHSM Client installed.
  - AWS provisions HSM but has no access to the secure area holding key material.
- **CloudHSM Use Cases**:
  - No native AWS integration (e.g., no S3 SSE).
  - Offload SSL/TLS processing for web servers.
  - Enable Transparent Data Encryption (TDE) for Oracle Databases.
  - Protect private keys for an Issuing Certificate Authority (CA).

## AWS Config
- **Two Primary Functions**:
  - Record configuration changes over time on resources.
  - Audit changes and compliance with standards.
- Does **NOT** prevent changes (no protection).
- **Regional Service**: Supports cross-region and cross-account aggregation.
- **Notifications**: Changes generate SNS notifications and near-real-time events via EventBridge & Lambda.
- **Storage**: All information stored regionally in an S3 config bucket.
- **Evaluation**: Resources evaluated against config rules (AWS-managed or custom via Lambda).
- **Remediation**: EventBridge can invoke Lambda for automatic resource remediation based on AWS Config events.

## Amazon Macie
- **Data Security and Privacy Service**:
  - Discovers, monitors, and protects data stored in S3 buckets.
  - Automated discovery of sensitive data (PII, PHI, financial data).
- **Data Identifiers**:
  - **Managed Data Identifiers**: AWS-maintained, covering common sensitive data types (credentials, financial, health, personal identifiers).
  - **Custom Data Identifiers**: User-created, regex-based.
    - Regex: Defines a pattern (e.g., `[A-Z]-\d{8}`).
    - Keywords: Optional sequences near regex matches.
    - Maximum Match Distance: Defines proximity of keywords to regex.
    - Ignore Words: Excludes matches containing specific words.
- **Integration**: With Security Hub and “finding events” to EventBridge.
- **Management**: Centrally managed via AWS Organizations or one Macie account inviting others.
- **Findings**: Policy Findings or Sensitive Data Findings.

## Notes
- **Shield Standard**:
  - Automatically provided with:
    - CloudFront.
    - Route 53.
  - Protects against **DDoS attacks**.
- **WAF Protections**:
  - Layer 7 attacks.
  - SQL Injection.
  - Cross-Site Scripting.
- **WAF Integration**:
  - CloudFront.
  - API Gateway.
  - ALB.
- **Secrets Manager vs. Parameter Store**:
  - Main feature: **Password Rotation** (Secrets Manager).