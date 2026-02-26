# Networking — Medium Questions

---

## 1. NAT Gateway vs VPC Peering — what's the difference?

> **Also asked as:** "NAT Gateway vs VPC Peering"
> **Also asked as:** "What are NAT gateway?"

Both are AWS networking constructs used to route traffic from private subnets, but they solve two completely different problems: **Egress vs Internal Connectivity**.

### NAT Gateway (Many-to-One Outbound)
- **Purpose:** Allows resources in a private subnet (like EC2 or EKS nodes) to connect to the **Internet** (or AWS public services like S3/ECR) without giving them public IP addresses.
- **Direction:** One-way outbound. The internet cannot initiate a connection back to your private instances.
- **Traffic Flow:** Private Instance → NAT Gateway (in public subnet) → Internet Gateway → Internet.
- **Cost Warning:** You pay an hourly rate *plus* a data processing fee per GB. Pulling large Docker images through a NAT Gateway can cause massive bill spikes. (Pro tip: Use VPC Endpoints to avoid this for AWS services).

### VPC Peering (Many-to-Many Internal)
- **Purpose:** Connects **two different VPCs** together so they can communicate using private IP addresses. It acts as a backbone connection.
- **Direction:** Two-way internal. Resources in VPC A can talk to VPC B, and vice versa, as if they were on the same network.
- **Traffic Flow:** Private Instance (VPC A) → VPC Peering Connection → Private Instance (VPC B). Traffic never touches the public internet.
- **Limitations:** No transitive peering. If A is peered with B, and B is peered with C, A still **cannot** talk to C.
- **Cost:** You do not pay an hourly fee for a peering connection, but you do pay for cross-AZ data transfer if the traffic crosses Availability Zones.

**Summary:** 
Use a **NAT Gateway** when your private pod needs to download an update from GitHub.
Use **VPC Peering** when your private pod in the "App VPC" needs to query a database living in the "Shared Data VPC".
