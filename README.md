# Lab 2 — Securely Deploying Resources in a VPC Using CloudFormation

## Architecture Overview

```
Region
└── VPC (10.0.0.0/16)
    ├── AZ-A
    │   ├── Public Subnet A (10.0.1.0/24)  → IGW → Internet
    │   │   ├── Web Server A (Apache)
    │   │   └── NAT Gateway A
    │   └── Private Subnet A (10.0.3.0/24) → NAT-A → Internet
    │       └── App Server A
    └── AZ-B
        ├── Public Subnet B (10.0.2.0/24)  → IGW → Internet
        │   ├── Web Server B (Apache)
        │   └── NAT Gateway B
        └── Private Subnet B (10.0.4.0/24) → NAT-B → Internet
            └── App Server B
```

- Internet Gateway: single IGW shared by both public subnets
- NAT Gateways: one per AZ — private subnet A routes through NAT-A, private subnet B routes through NAT-B (no cross-AZ dependency)
- All EC2 access: AWS Systems Manager Session Manager only (no SSH, no bastion)

## Deploy

```bash
aws cloudformation deploy \
  --template-file vpc-lab.yaml \
  --stack-name lab2-vpc \
  --capabilities CAPABILITY_NAMED_IAM
```

Stack outputs include the public IPs and URLs for both web servers.

## Validate

### Browser access
Open the URLs from the stack outputs — both should display the Apache page.

### Session Manager — ping public → private
```bash
# From Web Server A session
ping <AppServerA-PrivateIP>

# From Web Server B session
ping <AppServerB-PrivateIP>
```

### Session Manager — outbound internet from private instances
```bash
# From App Server A or B session (routes through AZ-aligned NAT Gateway)
ping 8.8.8.8
traceroute 8.8.8.8
# or install a package to confirm egress
sudo dnf install -y curl
```

## Design Decisions

- **AZ-aligned NAT Gateways**: each private subnet routes outbound traffic through the NAT Gateway in its own AZ. This eliminates cross-AZ data transfer costs and removes a single point of failure — if AZ-A goes down, AZ-B private resources still have egress through NAT-B.
- **Least-privilege security groups**: public SG allows only TCP/80 (HTTP) and ICMP from within the VPC. Private SG allows only ICMP from within the VPC. No SSH (port 22) anywhere.
- **SSM Session Manager**: IAM role with `AmazonSSMManagedInstanceCore` attached to all instances via an instance profile. No key pairs needed.
- **Amazon Linux 2023**: uses `dnf` instead of `yum`; AMI resolved dynamically via SSM Parameter Store so the template stays region-agnostic.

## AWS Regional NAT Gateway vs. AZ-scoped NAT Gateways

A standard NAT Gateway is **AZ-scoped** — it lives in one subnet inside one AZ. Traffic from a private subnet in a different AZ must cross AZ boundaries to reach it, which adds latency and incurs inter-AZ data transfer charges. If that AZ fails, all private subnets routing through it lose egress.

AWS announced a **Regional NAT Gateway** (public preview) that operates at the VPC level rather than the AZ level. It automatically routes outbound traffic through the nearest healthy AZ, eliminating the need to deploy one NAT Gateway per AZ. A single Regional NAT Gateway could replace the two AZ-scoped NAT Gateways in this architecture, simplifying the template and reducing cost while maintaining high availability natively.
