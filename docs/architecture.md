# Architecture Diagram Description

## System Overview

This document describes the architecture of the Nextcloud RDS deployment, where web traffic flows from end users through an EC2-hosted web server into a managed MySQL database on Amazon RDS.

```
┌──────────┐          ┌─────────────────────────────────┐          ┌────────────────────┐
│          │  HTTPS   │            EC2 Instance           │  MySQL   │    Amazon RDS       │
│   User   │ ──────► │   Nginx (reverse proxy / TLS)     │ ──────► │  MySQL Database    │
│ (Browser)│  :443    │   PHP-FPM (Nextcloud app layer)   │  :3306   │  (Multi-AZ ready)  │
└──────────┘          └─────────────────────────────────┘          └────────────────────┘
                              │                                              │
                       Public Subnet                               Private Subnet
                       (VPC)                                       (VPC)
```

---

## Component Roles

| Component | Role |
|-----------|------|
| **User (Browser)** | Initiates HTTPS requests to the Nextcloud web application |
| **EC2 – Nginx** | Terminates TLS, serves static assets, and proxies dynamic requests to PHP-FPM |
| **EC2 – PHP-FPM** | Executes Nextcloud PHP application logic, reads/writes data via MySQL |
| **Amazon RDS (MySQL)** | Managed relational database; stores Nextcloud metadata, users, and file references |

---

## Network Layout

```
┌────────────────────────────────────── VPC (e.g. 10.0.0.0/16) ──────────────────────────────────────┐
│                                                                                                      │
│  ┌──────────────────────────────────┐        ┌─────────────────────────────────────────────────┐   │
│  │      Public Subnet               │        │            Private Subnet                        │   │
│  │      (e.g. 10.0.1.0/24)          │        │            (e.g. 10.0.2.0/24)                   │   │
│  │                                  │        │                                                   │   │
│  │  ┌───────────────────────────┐   │        │  ┌─────────────────────────────────────────┐    │   │
│  │  │  EC2 Instance             │   │        │  │  RDS MySQL Instance                     │    │   │
│  │  │  Security Group: web-sg   │───┼────────┼─►│  Security Group: rds-sg                 │    │   │
│  │  │  Inbound:  443 (0.0.0.0) │   │        │  │  Inbound: 3306 from web-sg only         │    │   │
│  │  │  Outbound: All            │   │        │  │  Outbound: None required                │    │   │
│  │  └───────────────────────────┘   │        │  └─────────────────────────────────────────┘    │   │
│  └──────────────────────────────────┘        └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Security Groups

Security groups act as virtual firewalls that control inbound and outbound traffic at the instance level.

### `web-sg` — EC2 Web Server Security Group

| Direction | Protocol | Port | Source | Purpose |
|-----------|----------|------|--------|---------|
| Inbound | TCP | 443 | `0.0.0.0/0` | Accept HTTPS from all users |
| Inbound | TCP | 80 | `0.0.0.0/0` | Accept HTTP (redirect to HTTPS) |
| Inbound | TCP | 22 | Trusted IP range | SSH administration |
| Outbound | All | All | `0.0.0.0/0` | Allow responses and outbound updates |

### `rds-sg` — RDS MySQL Security Group

| Direction | Protocol | Port | Source | Purpose |
|-----------|----------|------|--------|---------|
| Inbound | TCP | 3306 | `web-sg` (security group ID) | Allow MySQL traffic **only** from the EC2 web tier |
| Outbound | — | — | — | No outbound rules required for RDS |

> **Key principle:** The RDS instance's security group references `web-sg` by ID rather than a CIDR range. This means only instances that belong to `web-sg` can connect to MySQL — even if the subnet CIDR changes or new instances are added.

---

## Network Isolation

Network isolation is achieved through a combination of subnet placement and security group rules:

1. **Public vs. Private Subnets**
   - The EC2 instance lives in a **public subnet** with a route to an Internet Gateway, allowing it to receive inbound HTTPS traffic from users.
   - The RDS instance lives in a **private subnet** with **no route to the Internet Gateway**, making it completely unreachable from the public internet.

2. **No Direct Database Access from the Internet**
   - Because the RDS subnet has no internet route, even if an attacker obtained the MySQL credentials, they could not connect to port 3306 from outside the VPC.

3. **Least-Privilege Security Group Chaining**
   - `rds-sg` only permits traffic originating from `web-sg`, so the database accepts connections **exclusively** from the Nextcloud application layer.
   - This prevents lateral movement: other resources within the same VPC cannot reach the database unless they are also assigned `web-sg`.

4. **Encrypted Connections**
   - HTTPS (TLS 1.2+) encrypts all traffic between users and Nginx.
   - MySQL connections between EC2 and RDS should use SSL/TLS to encrypt data in transit within the VPC.

---

## Scalability Benefits

This architecture is designed to scale each tier independently:

### Horizontal Scaling of the Web Tier

- Multiple EC2 instances running Nginx + PHP-FPM can be placed behind an **Application Load Balancer (ALB)**.
- Each instance joins `web-sg`, so they all automatically gain access to RDS without any security group changes.
- An **Auto Scaling Group (ASG)** can add or remove EC2 instances based on CPU or request-count metrics.

```
                        ┌──────────────────────┐
          Users ──────► │  Application Load    │
                        │  Balancer (ALB)      │
                        └──────────┬───────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
               EC2 (AZ-a)    EC2 (AZ-b)    EC2 (AZ-c)
               web-sg        web-sg        web-sg
                    └──────────────┼──────────────┘
                                   │
                                   ▼
                            RDS MySQL (rds-sg)
```

### Vertical and Read Scaling of the Database Tier

- RDS instance type can be resized with a single API call or console action (**vertical scaling**).
- **Read Replicas** can be added to offload read-heavy Nextcloud operations (e.g., file listing, search) from the primary instance.
- **Multi-AZ deployment** provides automatic failover to a standby replica in a second Availability Zone, ensuring high availability.

### Storage Scaling

- RDS supports **storage auto-scaling**: disk capacity increases automatically when utilization exceeds a configured threshold, with zero downtime.
- EC2 EBS volumes for Nextcloud data can similarly be resized or replaced with higher-throughput storage classes (gp3, io2) without reprovisioning the instance.

### Summary of Scalability Levers

| Concern | Solution |
|---------|----------|
| Web traffic spikes | ALB + Auto Scaling Group across multiple AZs |
| Database read load | RDS Read Replicas |
| Database availability | RDS Multi-AZ standby |
| Database storage | RDS storage auto-scaling |
| Nextcloud file storage | S3 primary storage backend (offloads EC2 disk entirely) |
