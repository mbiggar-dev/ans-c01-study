# AWS Advanced Networking Specialty (ANS-C01) Cheatsheet
## The "Memorise This Bullshit" Edition

*Everything you need to know but will never recall from memory in the real world.*

---

## Exam Quick Facts
- **Questions**: 65 (50 scored + 15 unscored)
- **Passing Score**: 750/1000
- **Time**: 170 minutes
- **Cost**: $300 USD

### Domain Weights
| Domain | Weight |
|--------|--------|
| Network Design | 30% |
| Network Implementation | 26% |
| Network Security, Compliance & Governance | 24% |
| Network Management & Operations | 20% |

---

## VPC Fundamentals

### CIDR Blocks
| Type | Allowed Range | Notes |
|------|--------------|-------|
| IPv4 VPC | /16 to /28 | 65,536 to 16 IPs |
| IPv4 Secondary CIDRs | Up to 5 total | 1 primary + 4 secondary |
| IPv6 VPC | /44 to /60 | In increments of /4 |
| IPv6 Subnet | /44 to /64 | In increments of /4 |
| IPv4 Subnet | /16 to /28 | Same as VPC |

### Reserved IP Addresses (per subnet)
| Address | Purpose |
|---------|---------|
| .0 | Network address |
| .1 | VPC Router |
| .2 | DNS Server (VPC CIDR +2) |
| .3 | Reserved for future use |
| .255 | Broadcast (not supported) |

**5 IPs reserved per subnet** = First 4 + Last 1

### Default VPC
- CIDR: **172.31.0.0/16**
- Default subnets: **/20** per AZ

---

## Direct Connect

### Physical Connections
| Speed | Type | Notes |
|-------|------|-------|
| 1 Gbps | Dedicated | Single-mode fibre |
| 10 Gbps | Dedicated | Single-mode fibre, MACsec available |
| 100 Gbps | Dedicated | Single-mode fibre, MACsec available |
| 50 Mbps - 10 Gbps | Hosted | Via AWS Partner |

**Auto-negotiation must be DISABLED. Port speed/duplex set manually.**

### Virtual Interface Types
| VIF Type | Use Case | MTU |
|----------|----------|-----|
| Private VIF | Connect to VPC via VGW | 1500 or **9001** (jumbo) |
| Transit VIF | Connect to TGW via DX Gateway | 1500 or **8500** (jumbo) |
| Public VIF | Access AWS public services | 1500 |

### BGP ASN Ranges
| Type | Range |
|------|-------|
| Private ASN (16-bit) | **64512 - 65534** |
| Private ASN (32-bit) | **4200000000 - 4294967294** |
| AWS Public ASN | **7224** (used on public VIF AWS side) |
| Default VGW ASN (post June 2018) | **64512** |

### DX Gateway
- **Global resource** - can connect from any region
- Associate with up to **20 VPCs** OR **6 Transit Gateways**
- ASN **cannot be modified after creation** - must delete and recreate
- Only routes between AWS and customer gateway (non-transitive)

### BGP Communities
| Community | Scope |
|-----------|-------|
| 7224:9100 | Local AWS Region only |
| 7224:9200 | Continent |
| 7224:9300 | Global |
| NO_EXPORT | Default on incoming customer prefixes |

### SiteLink
- Direct connectivity between DX locations
- MTU: **8500 or 9001** depending on VIF type
- Overrides normal regional preference for shorter AS path

---

## Transit Gateway

### Key Limits
| Resource | Default Limit |
|----------|---------------|
| TGWs per account per region | 5 |
| Attachments per TGW | 5,000 |
| Route tables per TGW | 20 |
| Routes per route table | 10,000 |
| VPC attachments per VPC | 1 (to same TGW) |

### Bandwidth
| Attachment Type | Bandwidth |
|-----------------|-----------|
| VPC attachment | **50 Gbps** (burst capable) |
| VPN attachment | **1.25 Gbps** per tunnel (or 5 Gbps with LBT) |
| Connect peer | Up to **5 Gbps** each, **4 peers max** = 20 Gbps |

### MTU
- VPC ↔ TGW: **8500 bytes**
- VPN attachments: **1500 bytes**
- DX/Peering: **8500 bytes**

### Routing Behaviour
- VPCs propagate routes via **internal APIs** (not BGP)
- Peering attachments: **static routes only**
- VPN attachments: dynamic (BGP) or static
- ECMP supported for VPN (must be dynamic routing)

---

## Site-to-Site VPN

### Tunnel Specs
| Parameter | Value |
|-----------|-------|
| Tunnels per VPN connection | **2** |
| Standard bandwidth per tunnel | **1.25 Gbps** |
| Large Bandwidth Tunnel | **5 Gbps** (TGW/Cloud WAN only) |
| MTU | **1446 bytes** |
| MSS | **1406 bytes** |
| Inside tunnel CIDR | /30 from **169.254.0.0/16** |

### Max Aggregate Bandwidth
- Single VPN (2 tunnels with ECMP): **~2.5 Gbps**
- Multiple VPNs with ECMP: Up to **50 Gbps** total

### IKE & IPsec
- **IKEv2 recommended** over IKEv1 (faster, more secure)
- Pre-shared key: **8-64 characters**
- Phase 1 encryption: AES128, AES256, AES128-GCM-16, AES256-GCM-16
- DH Groups: 2, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24

### ASN Limits
| Gateway | ASN Range |
|---------|-----------|
| VGW | 1 - 2,147,483,647 (4-byte) |
| CGW | 1 - 65,535 (2-byte) |

### DPD (Dead Peer Detection)
- Timeout: 30 seconds default
- Actions: Clear, Restart, None

---

## Route 53 Resolver

### DNS Server Location
- **VPC CIDR + 2** (e.g., 10.0.0.2 for 10.0.0.0/16)
- IPv6: **fd00:ec2::253**

### Query Limits
| Type | Limit |
|------|-------|
| Default DNS (per ENI) | **1,024 QPS** |
| Resolver endpoint (per ENI) | **10,000 QPS** |
| ENIs per endpoint | Up to **6** |

### Resolver Endpoints
- **Inbound**: On-prem → AWS (allows queries TO your VPC)
- **Outbound**: AWS → On-prem (allows queries FROM your VPC)
- Minimum **2 IPs** per endpoint (HA across AZs)
- Rules can be shared via **RAM**

---

## Load Balancers

### Comparison
| Feature | ALB | NLB | GWLB |
|---------|-----|-----|------|
| Layer | 7 | 4 | 3 |
| Protocol | HTTP/HTTPS/gRPC | TCP/UDP/TLS | IP |
| Static IP | ❌ | ✅ | ✅ |
| Preserve Source IP | ❌ (X-Forwarded-For) | ✅ | ✅ |
| Cross-zone default | ✅ Enabled | ❌ Disabled | ❌ Disabled |
| PrivateLink support | ❌ | ✅ | ✅ |

### Key Limits
- Targets per ALB: **1,000**
- Targets per NLB: **500** per AZ (3,000 total for 6 AZs)
- Rules per ALB: **100** (default)

---

## Global Accelerator

### Endpoints
- **Standard accelerator**: NLB, ALB, EC2, Elastic IP
- **Custom routing accelerator**: VPC subnets with EC2

### Key Features
- **2 static anycast IPs** per accelerator
- Traffic dial: 0-100% per endpoint group
- Endpoint weights for traffic distribution
- Health checks every **10 or 30 seconds**

### vs CloudFront
| Global Accelerator | CloudFront |
|--------------------|------------|
| TCP/UDP | HTTP/HTTPS |
| Static IPs | No static IPs |
| No caching | Caching |
| Gaming, VoIP, non-HTTP | Web content delivery |

---

## CloudFront

### Origin Types
- S3 bucket (REST API endpoint)
- S3 website endpoint (treated as custom origin)
- ALB, NLB, EC2
- Custom HTTP origin
- Lambda function URL
- MediaStore, MediaPackage

### Origin SSL Protocols
- SSLv3 (deprecated, avoid)
- TLSv1.0
- TLSv1.1
- **TLSv1.2** (recommended minimum)
- TLSv1.3 (viewer ↔ CloudFront only, NOT origin)

### Origin Protocol Policy
- **HTTP only**: Default for S3 website endpoints
- **HTTPS only**: Always use HTTPS to origin
- **Match viewer**: Use same protocol as viewer request

### Custom Origin Ports
- HTTP: 80, 443, or **1024-65535**
- HTTPS: 80, 443, or **1024-65535**

### ACM Certificates
- Must be in **us-east-1** for CloudFront
- Can be ACM-issued or imported

### Origin Access
- **OAC** (Origin Access Control): Newer, recommended for S3
- **OAI** (Origin Access Identity): Legacy, still supported

---

## VPC Endpoints

### Types
| Type | Services | DNS | Routing |
|------|----------|-----|---------|
| Interface | Most AWS services | Private DNS optional | ENI in subnet |
| Gateway | **S3, DynamoDB only** | Uses prefix lists | Route table entry |
| Gateway Load Balancer | Third-party appliances | N/A | Route table entry |

### Interface Endpoint Limits
- **50** interface endpoints per VPC (adjustable)
- **10** subnets per interface endpoint

---

## Network ACLs vs Security Groups

| Feature | NACL | Security Group |
|---------|------|----------------|
| Level | Subnet | Instance/ENI |
| State | **Stateless** | **Stateful** |
| Rules | Allow AND Deny | Allow only |
| Evaluation | Numbered order | All rules evaluated |
| Default | Allow all | Deny all inbound |

### NACL Rule Numbers
- Evaluated **lowest to highest**
- Range: 1 - 32766
- Best practice: Use increments of 100

### Ephemeral Ports
- Linux: **32768 - 60999**
- Windows: **49152 - 65535**
- ELB: **1024 - 65535**

---

## Flow Logs

### Capture Locations
- VPC level
- Subnet level
- ENI level

### Destinations
- CloudWatch Logs
- S3
- Kinesis Data Firehose

### NOT Captured
- DNS traffic to Amazon-provided DNS
- DHCP traffic
- Metadata (169.254.169.254)
- Time sync (169.254.169.123)
- Windows license activation
- Traffic to reserved IPs

---

## IPv6

### Key Points
- All IPv6 addresses are **globally unique** (no NAT)
- Egress-only internet gateway for **outbound only** IPv6
- IPv6 CIDR from Amazon: **/56** per VPC (default)
- VPCs now support **/44 to /60** in /4 increments
- Subnets: **/44 to /64** in /4 increments (default /64)

### Dual-Stack
- Instances get both IPv4 and IPv6
- Route tables need entries for both
- Security groups/NACLs need rules for both

---

## Hybrid DNS Architecture

### Forwarding Rules Priority (most to least specific)
1. Private Hosted Zone
2. Resolver Rules
3. VPC DNS (AmazonProvidedDNS)

### Inbound Endpoint Use Case
On-prem → AWS DNS resolution

### Outbound Endpoint Use Case
AWS → On-prem DNS resolution

---

## Traffic Mirroring

### Limits
| Resource | Limit |
|----------|-------|
| Sessions per ENI (source) | 3 |
| Mirror targets per account | 100,000 |
| Filters per account | 100,000 |

### Supported Instance Types
- Nitro-based instances
- NOT supported on: t2, t3 (burstable)

---

## Quick Number Reference

### Common Port Numbers
| Service | Port |
|---------|------|
| SSH | 22 |
| DNS | 53 |
| HTTP | 80 |
| HTTPS | 443 |
| NTP | 123 |
| BGP | 179 |
| ISAKMP/IKE | 500 |
| IPsec NAT-T | 4500 |

### Common CIDR Math
| CIDR | IPs | Usable (AWS) |
|------|-----|--------------|
| /28 | 16 | 11 |
| /27 | 32 | 27 |
| /26 | 64 | 59 |
| /25 | 128 | 123 |
| /24 | 256 | 251 |
| /23 | 512 | 507 |
| /22 | 1,024 | 1,019 |
| /20 | 4,096 | 4,091 |
| /16 | 65,536 | 65,531 |

---

## Troubleshooting Quick Reference

### VPN Tunnel Down
1. Check IKE phase 1 (ISAKMP SA)
2. Check IKE phase 2 (IPsec SA)
3. Verify BGP peering (if dynamic)
4. Check security group / NACL rules

### Asymmetric Routing
- Enable **Appliance Mode** on TGW VPC attachment
- Ensures traffic returns through same AZ

### MTU Issues
- Use Path MTU Discovery
- Set MSS clamping
- Consider jumbo frame support end-to-end

### DNS Resolution Failing
- Check VPC DNS settings (enableDnsHostnames, enableDnsSupport)
- Verify resolver endpoint security groups
- Check route tables for endpoint subnets

---

## Exam Tips

1. **Read questions carefully** - one word can change everything
2. **Eliminate obviously wrong answers** first
3. **Cost-effective** usually means simpler solution
4. **HA/fault-tolerant** usually means multi-AZ/region
5. **When in doubt**: TGW > VPC peering for scale
6. **DX + VPN** = backup pattern, not same-time active-active
7. **Route propagation** is automatic for DX/VPN, not for VPC routes

---

*Good luck, you've got this. It's basically formalising what you already know. 🚀*
