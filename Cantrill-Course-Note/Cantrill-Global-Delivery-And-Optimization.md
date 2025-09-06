# Cantrill Review – Global Delivery and Optimization

## CloudFront Architecture

### Terms
- **Origin**: The source location of your content.
  - **S3 Origin** or **Custom Origin** (Custom requires public IPv4).
- **Distribution**: The configuration unit of CloudFront.
- **Edge Location**: Local cache of your data.
  - Infrastructure cannot be deployed in Edge Locations.
- **Regional Edge Cache**: Larger version of an edge location.
  - Caches less frequently accessed data.

- Integrates with **ACM** for SSL and supports HTTPS.
- Upload directly to origins (no caching).
  - Client upload to cache location is NOT supported (download-only capabilities).
- **Behavior**: A configuration within a distribution.
  - Uses pattern matching.
- Origins are linked to behaviors as content sources.
- Distributions have one or more behaviors.
- A distribution can have multiple behaviors configured with a path pattern. If requests match the pattern, that behavior is used; otherwise, the default is used.
- Origins, Origin Groups, TTL, Protocol Policies, and Restricted Access are configured via Behaviors.

## CloudFront Behaviors
- Configured at the behavior level within a distribution:
  - Lambda Functions.
  - TTL.
  - SSL.
  - Restrict Viewer Access (force signed cookie or signed URL usage).

## CloudFront – TTL and Invalidations
- More frequent cache **HITS** reduce origin load.
- **Default TTL** (behavior): 24 hours (validity period).
- **Minimum TTL** and **Maximum TTL** can be set:
  - Applied at the behavior level.
  - Influence object-level configurations.
- **Object-Level Configurations**:
  - **Origin Header**: Cache-Control max-age (seconds).
  - **Origin Header**: Cache-Control s-maxage (seconds).
  - **Origin Header**: Expires (Date & Time).
  - Note: All header configurations (bucket-level) are restricted by the Min/Max TTL applied to the behavior. If no Min/Max TTL exists, the default (24 hours) is used.
- **Custom Origin or S3 Origin**:
  - Custom Origin: TTL applied in the application or web server.
  - S3 Origin: Applied via object metadata.
- **Cache Invalidations**:
  - Performed on a distribution, applies to all edge locations (not immediate).
  - Immediately expires TTL based on /path.
    - Examples:
      - `/images/whiskers1.jpg`
      - `/images/whiskers*`
      - `/images`
      - `/*` (invalidates all objects cached by a distribution).
  - Costs apply regardless; use only to correct errors.
- **Version File Names**:
  - Example: `Whiskers1_v1.jpg`, `_v2.jpg`, `_v3.jpg`.
  - Logging is more effective to track object usage.
  - Less expensive (no continued invalidation).
  - Different from S3 Versioning (uses different file names for different data, cached independently in edge locations).

## AWS Certificate Manager (ACM)
- **HTTP**: Simple and secure.
- **HTTPS**: Adds SSL/TLS encryption to HTTP (data encrypted in transit).
- **Certificates**: Prove identity, signed by a trusted authority (chain of trust).
- **ACM**:
  - Runs a public or private Certificate Authority (CA).
  - **Private CA**: Applications must trust your private CA.
  - **Public CA**: Browsers trust a list of providers, forming a chain of trust.
- Certificates can be generated or imported:
  - Generated: Automatically renews.
  - Imported: User is responsible for renewal (renew with external CA and reupload).
- Certificates can only be deployed to supported AWS services:
  - **Supported**: CloudFront and ALBs.
  - **Not Supported**: EC2 certificates, S3 certificates (S3 Origins handle certificates natively).
- **Regional Service**: Certificates cannot leave the region they are generated or imported into.
  - Example: For an ALB in `ap-southeast-2`, use a certificate in ACM in `ap-southeast-2`.
- **Global Services** (e.g., CloudFront):
  - Operate as if in `us-east-1`.
  - Certificates are deployed by ACM to the distribution, then sent to Edge Locations for identity verification, trust establishment, and secure connections.

## CloudFront and SSL/TLS
- **CloudFront Default Domain Name (CNAME)**:
  - Example: `https://d111111111abcdef8.cloudfront.net/`.
- SSL supported by default (`*.cloudfront.net` certificate).
- **Alternate Domain Names** (CNAME) supported:
  - Example: `cdn.catagram.com`.
- Verify ownership (optionally HTTPS) using a matching certificate.
- Generate or import certificates in ACM (`us-east-1`).
- **Supported Viewer Protocols**:
  - HTTP or HTTPS.
  - HTTP redirected to HTTPS.
  - HTTPS only.
- **Two SSL Connections**:
  - Viewer to CloudFront.
  - CloudFront to Origin.
  - Both require valid PUBLIC certificates (self-signed certificates NOT supported).
- **CloudFront and SNI**:
  - **SNI (Server Name Indication)**: Identifies the domain name for the client (TLS extension).
    - Allows one IP to host multiple HTTPS websites.
    - Older browsers may not support SNI (CloudFront charges extra for dedicated IP).
  - **SNI Mode**: Free.
  - **Dedicated IP**: $600/month.
- Origins require certificates issued by a trusted CA:
  - ALB: Use ACM.
  - Others: Use externally generated certificates (no self-signed certificates).
  - S3 Origins: Handle certificates natively.
- Certificates applied to CloudFront must match the DNS name of the Origin and vice versa.

## CloudFront Origin Types and Architecture
- **Origin Group**: Used by behaviors to group multiple origins for added resiliency.
- **S3 Bucket**:
  - If used as a static website, CloudFront treats it as a custom (webserver) origin.
  - **Origin Access**: Restrict access to allow only CloudFront Distribution.
    - Legacy: Legacy Access Identities.

## Securing CloudFront and S3 Using OAI
- **Origin Access Identity (OAI)**:
  - Applicable only for S3 origins (not using the static website feature).
  - OAI is an identity associated with CloudFront Distributions.
  - CloudFront assumes the OAI identity.
  - OAI can be used in S3 Bucket Policies.
    - Best practice: Default deny except for OAI associated with CloudFront (explicitly allow CloudFront, implicitly deny all other traffic).
  - Once associated, accesses are from the OAI.
  - OAI can be associated with multiple distributions (best practice: 1:1 OAI:CF Distribution).
- **Security for Custom Origins**:
  - **Custom Headers**:
    - Configure HTTPS viewer policy.
    - Require all requests to include the header (injected at CloudFront Edge Location).
  - Use the same location as the origin (Origin Protocol Policy).
  - Specify CloudFront IP ranges allowed in WAF.
  - Custom Origins can use either approach or a combination.

## CloudFront Private Distributions & Behaviors
- **CloudFront Private Behaviors, Signed URLs & Cookies**:
  - **Public Distributions**: Open access to objects.
  - **Private Distributions**: Require Signed Cookie or Signed URL.
  - Distributions are created with a single behavior.
  - Single Behavior: Whole distribution is public or private.
  - Multiple Behaviors: Each can be public or private.
  - **Old Method**: CloudFront Key created by Account Root User; account added as Trusted Signer.
  - **New Method**: Trusted Key Group(s).
    - More secure (avoids using trusted signer by account root user).
    - Supports associating multiple keys with a distribution/behavior.

## CloudFront Signed URLs vs. Signed Cookies
- **Signed URLs**: Provide access to one object.
  - Historically, RTMP distributions couldn’t use cookies (no longer a limitation).
  - Use URLs if the client doesn’t support cookies.
- **Signed Cookies**: Provide access to groups of objects.
  - Use for groups of files or all files of a type (e.g., all cat GIFs).
  - Use if maintaining application URLs is important.

## Lambda@Edge
- Run lightweight Lambda functions at Edge Locations to adjust data between Viewer and Origin.
- **Supported Runtimes**: Node.js and Python.
- **Environment**: Runs in AWS Public Space (not VPC).
- **Limitations**: Layers not supported; different limits compared to standard Lambda functions.
- **Event Triggers**:
  - **Viewer Request**: After CloudFront receives a request from a viewer.
  - **Origin Request**: Before CloudFront forwards the request to an origin.
  - **Origin Response**: After CloudFront receives a response from an origin.
  - **Viewer Response**: Before a response is forwarded to the viewer.
- **Limits**:
  - Viewer: 128MB, 5 seconds.
  - Origin: Standard Lambda limits, 30 seconds.
- **Use Cases**:
  - A/B testing (Viewer Request).
  - Migration between S3 Origins (Origin Request).
  - Different objects based on device type (Origin Request).
  - Vary content display by country (Origin Request).

## Global Accelerator
- Improves global network performance by providing entry points to the AWS global transit network using **ANYCAST IP Addresses**.
- **ANYCAST IPs**:
  - Example: `1.2.3.4` & `4.3.2.1`.
  - Single IP available in multiple locations; routing directs traffic to the closest location.
- **Traffic Flow**:
  - Initially uses the public internet to reach a Global Accelerator Edge Location.
  - From the edge, data transits the **AWS Global Backbone Network** (fewer hops, under AWS control).
  - Significantly better performance than CloudFront for non-HTTP/S traffic.
- **Key Concepts**:
  - Moves the AWS network closer to customers (unlike CloudFront, which caches content closer to customers).
  - Connections enter at the edge using ANYCAST IPs and transit over the AWS backbone to one or more locations.
  - Supports **non-