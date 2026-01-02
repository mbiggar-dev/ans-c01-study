# AWS Advanced Networking Specialty (ANS-C01)
## Comprehensive Study Guide

*A deep-dive reference for quiet reading over the next few days. Everything you need to truly understand the material, not just memorise it.*

---

# Part 1: VPC Fundamentals & Architecture

## 1.1 VPC Core Concepts

A VPC is a logically isolated virtual network within AWS. Think of it as your own private data centre in the cloud, but without the pain of managing physical infrastructure.

### IP Addressing

**IPv4 Addressing**
- VPC CIDR range: /16 (65,536 IPs) to /28 (16 IPs)
- RFC 1918 private ranges recommended:
  - 10.0.0.0/8
  - 172.16.0.0/12
  - 192.168.0.0/16
- You can use public IP ranges, but instances won't be publicly routable without additional configuration

**Secondary CIDRs**
- Up to 5 CIDR blocks total (1 primary + 4 secondary)
- Cannot overlap with existing CIDRs
- Cannot be larger than existing route table destinations
- Useful for expansion without re-architecting

**IPv6 Addressing**
- VPC: /44 to /60 (in /4 increments)
- Subnets: /44 to /64 (in /4 increments)
- All IPv6 addresses are globally unique - no NAT needed
- Amazon assigns from their pool, or use BYOIP

### Subnet Design

**Reserved Addresses (5 per subnet)**
```
.0   - Network address
.1   - VPC Router
.2   - DNS Server (always VPC CIDR + 2)
.3   - Reserved for future use
.255 - Broadcast (not supported, but reserved)
```

**Subnet Placement Strategy**
- Minimum: 1 subnet per AZ you want to use
- Best practice: Separate subnets for different tiers (web, app, db)
- Consider future growth - easier to start larger than expand

**Public vs Private Subnets**
- Public: Has route to Internet Gateway, instances get public IPs
- Private: No direct internet route, uses NAT for outbound
- The subnet itself isn't inherently public/private - it's the route table that determines this

---

## 1.2 VPC Routing

### Route Table Fundamentals

Every subnet must be associated with exactly one route table. If you don't explicitly associate, it uses the main route table.

**Route Priority**
1. Most specific route wins (longest prefix match)
2. Static routes take precedence over propagated routes (when prefix length is equal)
3. For equal specificity and type, propagated routes are ordered by:
   - Local VPC route
   - Direct Connect
   - Static VPN routes
   - BGP VPN routes

**Local Route**
- Automatically created, covers VPC CIDR
- Cannot be modified or deleted
- Always takes precedence for intra-VPC traffic

### Gateway Types

**Internet Gateway (IGW)**
- Horizontally scaled, redundant, highly available
- No bandwidth constraints
- Performs 1:1 NAT for instances with public IPs
- One per VPC

**NAT Gateway**
- Managed NAT service for private subnet outbound access
- 5-45 Gbps bandwidth (scales automatically)
- Per-AZ deployment for HA
- Cannot be used for inbound connections
- Supports TCP, UDP, ICMP

**NAT Gateway vs NAT Instance**
| Feature | NAT Gateway | NAT Instance |
|---------|-------------|--------------|
| Availability | HA within AZ | You manage HA |
| Bandwidth | Up to 45 Gbps | Instance dependent |
| Maintenance | Managed by AWS | You patch/update |
| Security Groups | No | Yes |
| Bastion | No | Yes (can be used as) |
| Port Forwarding | No | Yes |

**Egress-Only Internet Gateway**
- IPv6 equivalent of NAT Gateway
- Allows outbound IPv6, blocks inbound
- Stateful (return traffic is allowed)

---

## 1.3 VPC Security

### Security Groups

**Characteristics**
- Stateful - return traffic automatically allowed
- Allow rules only (implicit deny)
- Applied at ENI level
- Can reference other security groups
- Evaluated as a set (all rules considered)

**Best Practices**
- Least privilege principle
- Use descriptive names/tags
- Group by function, not by instance
- Reference security groups instead of IPs where possible

### Network ACLs

**Characteristics**
- Stateless - must explicitly allow return traffic
- Allow AND deny rules
- Applied at subnet level
- Evaluated in rule number order (lowest first)
- Default NACL allows all traffic

**Rule Numbering**
- Range: 1 - 32766
- Best practice: Use increments of 100 (leaves room for insertions)
- * (asterisk) rule = default deny, cannot be changed

**Ephemeral Port Considerations**
You must allow ephemeral ports for return traffic:
- Linux: 32768-60999
- Windows: 49152-65535
- ELB: 1024-65535 (NAT instances too)

### Comparison Table

| Aspect | Security Group | NACL |
|--------|---------------|------|
| Scope | ENI | Subnet |
| State | Stateful | Stateless |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules | Numbered order |
| Default | Deny all inbound | Allow all |

---

## 1.4 VPC Connectivity Options

### VPC Peering

**What It Is**
- Private connectivity between two VPCs
- Uses AWS backbone (no IGW, VPN, or public internet)
- Can span regions (inter-region peering)
- Can span accounts

**Limitations**
- Not transitive (VPC A ↔ B ↔ C doesn't mean A ↔ C)
- CIDR blocks cannot overlap
- One peering connection per VPC pair
- Must update route tables on both sides

**Use Cases**
- Simple hub-spoke with few VPCs
- Lowest latency inter-VPC connectivity
- When Transit Gateway is overkill

### Transit Gateway

**What It Is**
- Regional virtual router
- Hub-and-spoke connectivity for VPCs and on-prem
- Supports transitive routing
- Scales to thousands of connections

**Key Features**
- Route tables for traffic segmentation
- Attachments for VPCs, VPNs, Direct Connect
- Peering between TGWs (inter-region)
- Multicast support
- Equal Cost Multi-Path (ECMP) for VPN

**Attachments and Limits**
| Resource | Default Limit |
|----------|---------------|
| TGWs per region | 5 |
| Attachments per TGW | 5,000 |
| Route tables | 20 |
| Routes per table | 10,000 |
| Peering attachments | 50 |

**Bandwidth**
- VPC attachment: 50 Gbps (can burst higher)
- VPN: 1.25 Gbps per tunnel (or 5 Gbps with Large Bandwidth Tunnel)
- Per-flow limit: ~5 Gbps

**Route Propagation**
- VPCs: Internal APIs (not BGP)
- VPN/Direct Connect: BGP
- Peering: Static routes only

---

# Part 2: Hybrid Connectivity

## 2.1 Direct Connect

### Physical Architecture

**Connection Types**
| Type | Speeds | Description |
|------|--------|-------------|
| Dedicated | 1, 10, 100 Gbps | Your own port at DX location |
| Hosted | 50 Mbps - 10 Gbps | Shared port via APN partner |

**Physical Requirements**
- Single-mode fibre only
- Auto-negotiation must be disabled
- Port speed and duplex set manually
- 802.1Q VLAN tagging required

**Letter of Authorization (LOA-CFA)**
- Required document for cross-connect
- Specifies rack location, port, and technical requirements
- Generated by AWS, given to your colocation provider

### Virtual Interfaces (VIFs)

**Private VIF**
- Access VPC resources via VGW or DX Gateway
- MTU: 1500 or 9001 (jumbo frames)
- Uses private ASN for BGP
- One IPv4 and one IPv6 BGP session supported

**Transit VIF**
- Access TGW via DX Gateway
- MTU: 1500 or 8500 (jumbo frames)
- Supports up to 6 Transit Gateways
- Only one Transit VIF per DX connection

**Public VIF**
- Access all AWS public services (S3, DynamoDB, etc.)
- Uses public ASN (yours) or private ASN (replaced with 7224)
- MTU: 1500 only
- Must advertise public prefixes you own

### Direct Connect Gateway

**Purpose**
- Global resource for connecting DX to multiple VPCs/TGWs
- Eliminates need for VIF per VPC
- Can associate with VPCs in any region

**Associations**
- Up to 20 VGWs (VPCs) OR
- Up to 6 Transit Gateways
- Cannot mix VGWs and TGWs

**Key Points**
- ASN set at creation, cannot change later
- Non-transitive (only routes between DX and AWS, not between attached resources)

### BGP Configuration

**ASN Ranges**
| Type | Range |
|------|-------|
| Private 16-bit | 64512 - 65534 |
| Private 32-bit | 4200000000 - 4294967294 |
| AWS Default VGW | 64512 |
| AWS Public VIF | 7224 |

**BGP Communities**
| Community | Scope |
|-----------|-------|
| 7224:9100 | Local region only |
| 7224:9200 | Continent |
| 7224:9300 | Global |
| NO_EXPORT | Applied to incoming customer prefixes |

**Route Preferences**
- Longest prefix match first
- Then AS_PATH length
- Then Local Preference (communities)
- Finally MED

### Resiliency Patterns

**Development/Non-Critical**
- Single connection, single location

**High Resiliency**
- Two connections at two different locations

**Maximum Resiliency**
- Two connections at each of two locations (4 total)
- Separate devices at each location

**LAG (Link Aggregation Group)**
- Bundle up to 4 connections
- Same speed, same location, same device
- Active-active with LACP
- Provides bandwidth aggregation + redundancy

### MACsec

**What It Is**
- Layer 2 encryption between your router and AWS
- Available on 10 Gbps and 100 Gbps dedicated connections
- Uses IEEE 802.1AE standard

**Key Points**
- Connection Key Name (CKN) + Connectivity Association Key (CAK)
- Secret must be created in Secrets Manager
- Does not encrypt beyond the DX location

---

## 2.2 Site-to-Site VPN

### VPN Fundamentals

**Components**
- Customer Gateway (CGW): Represents your on-prem device
- Virtual Private Gateway (VGW): AWS-side VPN endpoint for VPC
- VPN Connection: Two encrypted tunnels

**Why Two Tunnels?**
- AWS maintains endpoints in separate facilities
- Automatic failover if one tunnel fails
- Both can be active (active-active with ECMP on TGW)

### Tunnel Specifications

| Parameter | Value |
|-----------|-------|
| Tunnels per connection | 2 |
| Standard bandwidth | 1.25 Gbps per tunnel |
| Large Bandwidth Tunnel | 5 Gbps (TGW only) |
| MTU | 1446 bytes |
| MSS | 1406 bytes |
| Inside tunnel CIDR | /30 from 169.254.0.0/16 |

### IKE and IPsec

**IKEv2 Recommended Over IKEv1**
- Fewer messages (4 vs 6)
- Built-in NAT traversal
- Asymmetric authentication
- Better performance

**Encryption Options**
- AES-128, AES-256
- AES-128-GCM-16, AES-256-GCM-16

**Diffie-Hellman Groups**
- Supported: 2, 5, 14-24
- Higher = more secure but slower

**Pre-Shared Key**
- 8-64 characters
- Alphanumeric, periods, underscores
- Cannot start with zero

### Accelerated VPN

**What It Is**
- VPN over AWS Global Accelerator
- Uses AWS edge locations for ingress
- Improves performance for distributed users

**When to Use**
- Users spread globally
- Latency-sensitive applications
- Unreliable internet paths

**Considerations**
- Additional cost
- Both tunnels on same accelerator
- Cannot convert existing VPN to accelerated

### VPN + Transit Gateway

**Benefits**
- ECMP across multiple VPN connections
- Centralised VPN management
- Transitive routing to all attached VPCs

**ECMP Requirements**
- Must use dynamic (BGP) routing
- Both tunnels active
- Same route advertised from on-prem

**Maximum Bandwidth**
- Single VPN: ~2.5 Gbps (2 tunnels × 1.25 Gbps)
- With ECMP: Up to 50 Gbps across multiple VPNs

---

## 2.3 Hybrid DNS

### Route 53 Resolver

**Default Behaviour**
- VPC DNS at CIDR + 2 (e.g., 10.0.0.2)
- Resolves: Public DNS, Private Hosted Zones, EC2 internal names
- Cannot be queried from outside VPC

**Inbound Endpoints**
- Allow on-prem to query AWS DNS
- Create ENIs in your VPC subnets
- On-prem forwards to these IPs
- Use Case: Resolve Private Hosted Zones from on-prem

**Outbound Endpoints**
- Allow AWS to query on-prem DNS
- Conditional forwarding rules
- Use Case: Resolve on-prem domain from VPC

### Resolver Rules

**Forwarding Rules**
- Domain name + target DNS servers
- Associated with VPCs
- Can be shared via RAM

**System Rules**
- Auto-created for VPC internal names
- Cannot be modified

**Rule Priority**
1. Resolver Rules (most specific domain wins)
2. Private Hosted Zones
3. Public DNS

### Query Limits

| Type | QPS Limit |
|------|-----------|
| Default DNS (per ENI) | 1,024 |
| Resolver Endpoint (per ENI) | 10,000 |
| ENIs per Endpoint | Up to 6 |

### DNS Firewall

**Purpose**
- Filter outbound DNS queries
- Block known bad domains
- Prevent DNS exfiltration

**Components**
- Domain Lists (allow/block lists)
- Rule Groups (ordered rules)
- Firewall associations to VPCs

---

# Part 3: Content Delivery & Edge

## 3.1 CloudFront

### Core Concepts

**Edge Locations**
- 400+ points of presence globally
- Cache content close to users
- Also used for Lambda@Edge

**Regional Edge Caches**
- Larger cache, fewer locations
- Sits between edge locations and origin
- Keeps content cached longer

### Origins

**S3 Bucket Origin**
- REST API endpoint (not website endpoint)
- Origin Access Control (OAC) recommended
- Legacy: Origin Access Identity (OAI)

**Custom Origin**
- Any HTTP/HTTPS endpoint
- EC2, ALB, on-prem server
- Must have public DNS name

**S3 Website Endpoint**
- Treated as custom origin
- No OAC support
- HTTP only (no HTTPS to origin)

**Origin Groups**
- Primary + secondary origin
- Automatic failover on 5xx or timeout

### Origin Protocols

**Origin Protocol Policy**
| Setting | Behaviour |
|---------|-----------|
| HTTP Only | Always HTTP to origin (S3 website default) |
| HTTPS Only | Always HTTPS to origin |
| Match Viewer | Same protocol as viewer request |

**Origin SSL Protocols**
- SSLv3, TLSv1.0, TLSv1.1, TLSv1.2
- TLSv1.3 NOT supported to origin
- Recommendation: TLSv1.2 minimum

**Custom Origin Ports**
- HTTP: 80, 443, or 1024-65535
- HTTPS: 80, 443, or 1024-65535

### Caching Behaviour

**Cache Key**
- URL path
- Query strings (configurable)
- Headers (configurable)
- Cookies (configurable)

**TTL Settings**
- Minimum TTL: Floor for caching
- Maximum TTL: Ceiling for caching
- Default TTL: Used when origin doesn't specify

**Cache Invalidation**
- Removes objects from edge caches
- Billed per path
- Use versioned file names instead where possible

### Security

**Viewer Protocol Policy**
| Setting | Behaviour |
|---------|-----------|
| HTTP and HTTPS | Accept both |
| Redirect HTTP to HTTPS | 301 redirect |
| HTTPS Only | HTTP returns 403 |

**SSL Certificates**
- ACM certificates: Must be in us-east-1
- Custom certificates: Must include SNI
- Supports RSA and ECDSA

**Field-Level Encryption**
- Encrypt sensitive fields at edge
- Only decryptable by your application
- Uses asymmetric encryption

**Signed URLs/Cookies**
- Restrict access to content
- Time-limited access
- IP restrictions possible

---

## 3.2 Global Accelerator

### What It Is

AWS Global Accelerator improves availability and performance by using the AWS global network instead of the public internet.

### Components

**Accelerator**
- Entry point with 2 static anycast IPs
- Points to listener(s)

**Listener**
- Port (or port range)
- Protocol (TCP or UDP)
- Routes to endpoint groups

**Endpoint Group**
- One per region
- Traffic dial (0-100%)
- Health check settings

**Endpoints**
- ALB, NLB, EC2, Elastic IP
- Weight for distribution

### Types of Accelerators

**Standard Accelerator**
- For ALB, NLB, EC2, Elastic IP
- Health checking
- Traffic distribution
- Client IP preservation (optional)

**Custom Routing Accelerator**
- For gaming, VoIP, IoT
- Maps ports to specific instances
- Deterministic routing
- Endpoints are VPC subnets

### Global Accelerator vs CloudFront

| Feature | Global Accelerator | CloudFront |
|---------|-------------------|------------|
| Protocols | TCP/UDP | HTTP/HTTPS |
| Caching | No | Yes |
| Static IPs | Yes | No |
| Use Case | Gaming, VoIP, non-HTTP | Web content |

---

# Part 4: Load Balancing

## 4.1 Application Load Balancer (ALB)

### Characteristics
- Layer 7 (HTTP/HTTPS/gRPC)
- Content-based routing
- Host/path-based rules
- Native HTTP/2 and WebSocket
- Cross-zone load balancing: Always on

### Target Types
- Instance
- IP address (including on-prem)
- Lambda function

### Routing Rules

**Conditions**
- Host header
- Path pattern
- HTTP headers
- HTTP methods
- Query strings
- Source IP

**Actions**
- Forward
- Redirect (301/302)
- Fixed response
- Authenticate (Cognito/OIDC)

### Key Features

**Sticky Sessions**
| Type | Cookie | Notes |
|------|--------|-------|
| Duration-based | AWSALB | ELB-generated |
| Application-based | Custom | Your app generates |

**Connection Draining (Deregistration Delay)**
- Default: 300 seconds
- Range: 0-3600 seconds
- Allows in-flight requests to complete

**Slow Start**
- Gradually increase traffic to new targets
- 30-900 seconds
- Prevents overwhelming new instances

---

## 4.2 Network Load Balancer (NLB)

### Characteristics
- Layer 4 (TCP/UDP/TLS)
- Ultra-low latency
- Millions of requests per second
- Static IP per AZ
- Cross-zone: Off by default

### Target Types
- Instance
- IP address
- ALB (for layered architecture)

### Key Features

**Preserve Source IP**
- Enabled by default for instance targets
- Configurable for IP targets
- Client sees real client IP

**Proxy Protocol v2**
- Alternative for preserving client IP
- Binary header with connection info
- Used when source IP preservation not possible

**Connection Idle Timeout**
- Default: 350 seconds
- Range: 60-6000 seconds
- Applies to TCP connections

**Sticky Sessions**
- Based on source IP (source_ip)
- Not supported on TLS listeners

---

## 4.3 Gateway Load Balancer (GWLB)

### Purpose
Deploy, scale, and manage third-party virtual appliances (firewalls, IDS/IPS, etc.)

### Architecture
- Operates at Layer 3 (IP packets)
- Uses GENEVE encapsulation (port 6081)
- Transparent to applications

### Components
- Gateway Load Balancer (in appliance VPC)
- Gateway Load Balancer Endpoint (in consumer VPC)
- Target group with appliances

### Traffic Flow
1. Traffic hits GWLB endpoint
2. Encapsulated and sent to appliances
3. Inspected/modified by appliances
4. Returned to GWLB
5. Forwarded to destination

---

# Part 5: VPC Endpoints & PrivateLink

## 5.1 Endpoint Types

### Gateway Endpoints

**Supported Services**
- Amazon S3
- DynamoDB

**Characteristics**
- FREE (no hourly or data charges)
- Route table entry points to endpoint
- Uses prefix lists for routing
- One endpoint per VPC per service

**S3 Gateway Endpoint**
- Policies can restrict bucket access
- Cannot be accessed from on-prem
- Region-specific

### Interface Endpoints

**How It Works**
- Creates ENI in your subnet
- Private IP from your VPC CIDR
- DNS resolution routes traffic to endpoint

**Private DNS**
- Optional (recommended)
- Resolves public service name to private IP
- Requires VPC DNS settings enabled

**Pricing**
- Hourly: ~$0.01/hour per AZ
- Data: ~$0.01/GB

**Limits**
- Default: 50 per VPC (adjustable)
- 10 subnets per endpoint
- Bandwidth: 10 Gbps per AZ, scales to 100 Gbps

### Gateway Load Balancer Endpoints

**Purpose**
- Route traffic through security appliances
- Insert inspection into traffic flow

**Use Cases**
- Centralised firewall
- IDS/IPS
- Traffic monitoring

---

## 5.2 Endpoint Services (PrivateLink)

### Creating an Endpoint Service

**Requirements**
- Network Load Balancer (or Gateway LB for GWLB endpoints)
- Service in your VPC

**Acceptance Settings**
- Manual: You approve each connection
- Automatic: Accept all requests

**Sharing**
- Within account
- Across accounts (with principal list)

### Consumer Side

**Discovering Services**
- AWS services: Use service name
- Your services: Need service name from provider
- AWS Marketplace: Available in console

**Connection States**
- Pending: Awaiting acceptance
- Available: Connected
- Rejected: Provider denied
- Deleted: Removed

---

# Part 6: Network Security

## 6.1 AWS Network Firewall

### Architecture

**Components**
- Firewall: Deployed in dedicated subnets
- Firewall Policy: Rules and settings
- Rule Groups: Collections of rules

**Deployment Models**
- Distributed: Firewall per VPC
- Centralised: Shared via Transit Gateway

### Rule Types

**Stateless Rules**
- Like NACLs - inspect each packet independently
- Priority-ordered evaluation
- Actions: Pass, Drop, Forward to Stateful

**Stateful Rules**
- Like Security Groups - track connections
- Suricata-compatible format
- Deep packet inspection
- Actions: Pass, Drop, Reject, Alert

### Rule Group Types

**Stateless**
- 5-tuple matching (src/dst IP, ports, protocol)
- TCP flags inspection

**Stateful Options**
- Standard rules: IP, port, direction
- Domain list: Filter by domain name
- Suricata: Full IPS capabilities

### Suricata Basics

**Rule Format**
```
action protocol src_ip src_port -> dst_ip dst_port (options)
```

**Example**
```
drop tcp $HOME_NET any -> $EXTERNAL_NET 443 (msg:"Block HTTPS"; sid:1;)
```

**Key Options**
- `flow:to_server` - Only match client-to-server
- `content:"string"` - Match payload content
- `sid:` - Unique rule ID

---

## 6.2 AWS WAF

### Components

**Web ACL**
- Container for rules
- Associated with CloudFront, ALB, API Gateway, AppSync, Cognito
- Capacity: 1,500 WCU default, 5,000 max

**Rules**
- Match conditions + action
- Allow, Block, Count, CAPTCHA, Challenge

**Rule Groups**
- Reusable sets of rules
- AWS Managed (free and paid)
- Your own custom groups

### Rule Types

**Rate-Based**
- Limit requests per IP or other aggregation
- Evaluation windows: 1, 2, 5, 10 minutes
- Threshold: 100-2,000,000,000
- WCU: 2 base + 30 per custom key

**Regular Rules**
- IP match
- Geographic match
- String match
- Regex match
- Size constraints
- SQL injection
- XSS

### Body Inspection Limits

| Resource | Default | Maximum |
|----------|---------|---------|
| CloudFront | 16 KB | 64 KB |
| API Gateway | 16 KB | 64 KB |
| ALB | 8 KB | 8 KB |

### Common AWS Managed Rule Groups

**IP Reputation**
- Amazon IP Reputation List
- Anonymous IP List

**Baseline**
- Core Rule Set (CRS)
- Admin Protection
- Known Bad Inputs

**Use Case Specific**
- SQL Database
- Linux Operating System
- PHP Application
- WordPress Application

---

## 6.3 AWS Shield

### Shield Standard
- Free, automatic
- All AWS customers
- Layer 3/4 DDoS protection
- SYN floods, UDP reflection

### Shield Advanced
- $3,000/month + data fees
- Layer 7 protection
- DDoS cost protection
- 24/7 DDoS Response Team (DRT)
- Advanced metrics and reporting
- Automatic application layer mitigation

### Protected Resources (Shield Advanced)
- CloudFront distributions
- Route 53 hosted zones
- Global Accelerator
- ALB, NLB, Classic LB
- Elastic IPs

---

# Part 7: Monitoring & Troubleshooting

## 7.1 VPC Flow Logs

### What's Captured
- Source/destination IP
- Source/destination port
- Protocol
- Packets/bytes
- Start/end time
- Action (ACCEPT/REJECT)

### What's NOT Captured
- DNS to Amazon DNS
- DHCP traffic
- Instance metadata (169.254.169.254)
- Time sync (169.254.169.123)
- Windows license activation
- Traffic to reserved IPs

### Destinations
- CloudWatch Logs
- S3
- Kinesis Data Firehose

### Log Levels
- VPC
- Subnet
- Network interface

### Custom Fields (Optional)
- TCP flags
- Traffic type
- Flow direction
- Packet metadata

---

## 7.2 Traffic Mirroring

### Purpose
Copy network traffic for analysis

### Components
- Mirror Source: ENI to capture from
- Mirror Target: ENI, NLB, or GWLB endpoint
- Mirror Filter: What to capture
- Mirror Session: Links source, target, filter

### Limits
- 3 sessions per source ENI
- 100,000 targets per account
- 100,000 filters per account

### Supported Instances
- Nitro-based only
- NOT supported: T2, T3

---

## 7.3 Reachability Analyzer

### Purpose
Analyse network paths without sending traffic

### What It Checks
- Security groups
- NACLs
- Route tables
- Load balancer rules
- VPC peering/TGW attachments

### Use Cases
- Verify connectivity before deployment
- Diagnose connectivity issues
- Validate security configurations

---

## 7.4 Common Troubleshooting Scenarios

### VPN Tunnel Down

**Check Order**
1. IKE Phase 1 (ISAKMP SA)
2. IKE Phase 2 (IPsec SA)
3. BGP peering (if dynamic routing)
4. Route propagation
5. Security group/NACL rules

### Asymmetric Routing

**Symptoms**
- Intermittent connectivity
- Stateful inspection failures

**Solutions**
- Enable Appliance Mode on TGW VPC attachment
- Use same AZ for ingress and egress
- Configure symmetric routing in on-prem

### MTU Issues

**Symptoms**
- Large packets dropped
- Slow transfers
- Connection hangs

**Solutions**
- Enable Path MTU Discovery
- Set MSS clamping on VPN
- Consider jumbo frame support end-to-end

### DNS Resolution Failures

**Check Order**
1. VPC DNS settings (enableDnsHostnames, enableDnsSupport)
2. DHCP option set
3. Resolver endpoint security groups
4. Route tables for endpoint subnets
5. Private hosted zone associations

---

# Part 8: Network Automation

## 8.1 CloudFormation

### VPC Resources
```yaml
AWS::EC2::VPC
AWS::EC2::Subnet
AWS::EC2::RouteTable
AWS::EC2::Route
AWS::EC2::InternetGateway
AWS::EC2::NatGateway
AWS::EC2::VPCEndpoint
AWS::EC2::SecurityGroup
AWS::EC2::NetworkAcl
```

### Cross-Stack References
- Export values from one stack
- Import in another with `!ImportValue`
- Useful for shared networking

---

## 8.2 AWS Transit Gateway Network Manager

### Purpose
- Centralised view of global network
- Visualise topology
- Monitor connections
- Event notifications

### Components
- Global Networks
- Devices (on-prem equipment)
- Sites (physical locations)
- Links (connections between sites)

---

## 8.3 VPC IPAM

### Purpose
- Plan, track, and monitor IP addresses
- Automate IP allocation
- Prevent overlapping CIDRs

### Features
- IP pools
- Allocation rules
- Audit trail
- Integration with AWS Organizations

---

# Exam Strategy

## Time Management
- 170 minutes / 65 questions ≈ 2.6 minutes per question
- Flag difficult questions, return later
- Don't overthink straightforward questions

## Question Types

**Scenario-Based**
- Most common
- Read requirements carefully
- Identify constraints (cost, performance, security)

**Best Practice**
- Usually one clearly correct answer
- Others might work but aren't optimal

**Troubleshooting**
- Follow logical diagnostic order
- Consider all components in path

## Key Themes

1. **Hybrid Connectivity**: DX + VPN patterns
2. **Transit Gateway**: Central routing hub
3. **Security**: Defense in depth
4. **DNS**: Hybrid resolution patterns
5. **High Availability**: Multi-AZ, multi-region

## Final Tips

- **When in doubt about HA**: Multi-AZ first, then multi-region
- **Cost questions**: Gateway endpoints are free, NAT Gateways are not
- **Performance questions**: DX > VPN > Internet
- **Security questions**: Layer multiple controls (SG + NACL + Firewall)
- **Troubleshooting**: Always check security groups and route tables first

---
