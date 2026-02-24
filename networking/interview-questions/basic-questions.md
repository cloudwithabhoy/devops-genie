# Networking — Basic Questions

---

## 1. Explain what happens when you type www.example.com in a browser.

This is a classic full-stack networking question. The answer covers DNS, TCP, TLS, and HTTP — all in one flow.

```
Browser → DNS resolution → TCP handshake → TLS handshake → HTTP request → Response → Render
```

---

**Step 1: DNS resolution — translate the domain name to an IP address.**

```
Browser checks its own DNS cache (TTL-based)
  ↓ (cache miss)
OS checks /etc/hosts file
  ↓ (not found)
OS sends query to configured DNS resolver (e.g., 8.8.8.8 or your ISP's resolver)
  ↓
Resolver checks its own cache
  ↓ (cache miss)
Resolver queries root DNS server → "Who handles .com?"
  ↓
Root server responds: "Ask the .com TLD nameserver"
  ↓
Resolver queries TLD nameserver → "Who handles example.com?"
  ↓
TLD responds: "Ask ns1.example.com (the authoritative nameserver)"
  ↓
Resolver queries authoritative nameserver → "What is the IP for www.example.com?"
  ↓
Authoritative nameserver responds: 93.184.216.34
  ↓
Resolver caches the result (TTL = 300s) and returns it to the browser
```

**Step 2: TCP connection — 3-way handshake.**

```
Browser → SYN         → Server
Browser ← SYN-ACK    ← Server
Browser → ACK         → Server
# Connection established
```

**Step 3: TLS handshake (HTTPS) — negotiate encryption.**

```
Browser → ClientHello (supported TLS versions, cipher suites)
Server  → ServerHello + Certificate (public key, issued by CA)
Browser → validates certificate against trusted CA store
Browser → generates session key, encrypts with server's public key, sends it
Server  → decrypts session key with its private key
Both sides now have the same session key → encrypted channel established
```

**Step 4: HTTP request sent.**

```
GET / HTTP/1.1
Host: www.example.com
Accept: text/html
```

**Step 5: Server processes and responds.**

The server (nginx, Apache, or app server) receives the request, runs application logic (if dynamic), and returns:

```
HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1256

<!DOCTYPE html>...
```

**Step 6: Browser renders the page.**

Browser parses HTML → finds `<link>`, `<script>`, `<img>` tags → makes additional HTTP requests for each asset → renders the page progressively.

---

**Key things that can go wrong at each step:**

| Step | Common failure |
|---|---|
| DNS | Wrong IP in DNS, TTL too high after migration |
| TCP | Port 80/443 not open in security group / firewall |
| TLS | Expired certificate, mismatched hostname, untrusted CA |
| HTTP | 404 (wrong path), 502 (app crashed), 503 (overloaded) |
| Render | CORS errors blocking assets, JS errors in console |

---

## 2. What is the OSI model and the TCP/IP model?

Both models describe how data travels across a network. The OSI model is a conceptual framework; TCP/IP is the practical model the internet actually uses.

**OSI Model — 7 layers:**

```
Layer 7 — Application   → HTTP, HTTPS, DNS, FTP, SSH       (what the app sends)
Layer 6 — Presentation  → TLS/SSL encryption, encoding     (format & encrypt)
Layer 5 — Session       → Managing connections/sessions     (session control)
Layer 4 — Transport     → TCP / UDP, ports, segmentation    (end-to-end delivery)
Layer 3 — Network       → IP addressing, routing            (logical addressing)
Layer 2 — Data Link     → MAC addresses, Ethernet frames    (node-to-node)
Layer 1 — Physical      → Cables, signals, bits             (raw transmission)
```

**TCP/IP Model — 4 layers (maps to OSI):**

```
Application  → OSI layers 5, 6, 7  → HTTP, DNS, SSH, TLS
Transport    → OSI layer 4         → TCP, UDP
Internet     → OSI layer 3         → IP, ICMP, routing
Network      → OSI layers 1, 2     → Ethernet, Wi-Fi, MAC
```

**Why DevOps engineers care:**

| Layer | Real-world relevance |
|---|---|
| L7 (Application) | HTTP status codes, ALB routing, WAF rules |
| L4 (Transport) | NLB, Security Group port rules, TCP handshake |
| L3 (Network) | IP addresses, CIDR, routing tables, VPC |
| L2 (Data Link) | MAC addresses, VLANs (mostly cloud-abstracted) |

When debugging: "Is this a network layer issue (L3 — can I ping the IP?) or an application layer issue (L7 — can I curl the endpoint)?"

---

## 3. What is the difference between TCP and UDP?

Both are transport layer (L4) protocols. The core difference: TCP guarantees delivery; UDP doesn't.

**TCP — Transmission Control Protocol.**

Connection-oriented. Before any data is sent, a 3-way handshake establishes the connection:
```
Client → SYN        → Server
Client ← SYN-ACK   ← Server
Client → ACK        → Server
# Connection established — data flows
```

Features: ordered delivery, acknowledgements, retransmission on loss, flow control.

**UDP — User Datagram Protocol.**

Connectionless. Fire and forget — no handshake, no acknowledgement, no retransmission.

```
Client → Data → Server   (no confirmation, no guarantee)
```

**When to use each:**

| Use case | Protocol | Why |
|---|---|---|
| Web (HTTP/HTTPS) | TCP | Every byte must arrive correctly |
| SSH | TCP | Commands must be received in order |
| Database queries | TCP | Data integrity is critical |
| DNS lookups | UDP | Fast, small packet, retry on failure is fine |
| Video streaming | UDP | Better to drop frames than pause to retransmit |
| VoIP / gaming | UDP | Low latency > perfect delivery |
| QUIC (HTTP/3) | UDP | Modern HTTP with its own reliability built on top |

**DevOps relevance:** Security groups and NACLs require you to specify TCP or UDP. NLB supports both. DNS uses UDP port 53. SSH uses TCP port 22.

---

## 4. What is DNS and how does name resolution work?

DNS (Domain Name System) translates human-readable domain names (`api.example.com`) into IP addresses (`93.184.216.34`) that computers use to route traffic.

**Resolution process (recursive):**

```
1. Browser checks its local DNS cache (TTL-based)
2. OS checks /etc/hosts file
3. OS sends query to configured DNS resolver (e.g. 8.8.8.8)
4. Resolver checks its cache → miss
5. Resolver queries root DNS servers → "Who handles .com?"
6. Root server → "Ask the .com TLD nameserver"
7. Resolver queries .com TLD → "Who handles example.com?"
8. TLD → "Ask ns1.example.com (authoritative nameserver)"
9. Resolver queries ns1.example.com → "What is api.example.com?"
10. Authoritative nameserver → 93.184.216.34 (with TTL)
11. Resolver caches + returns IP to the browser
```

**Key DNS record types:**

| Record | Purpose | Example |
|---|---|---|
| `A` | Domain → IPv4 address | `api.example.com → 93.184.216.34` |
| `AAAA` | Domain → IPv6 address | `api.example.com → 2606:2800::1` |
| `CNAME` | Domain → another domain | `www.example.com → example.com` |
| `MX` | Mail exchange server | `example.com → mail.example.com` |
| `TXT` | Arbitrary text (SPF, DMARC, verification) | SPF record |
| `NS` | Authoritative nameserver for the domain | `example.com → ns1.example.com` |
| `PTR` | Reverse DNS — IP → domain | `93.184.216.34 → api.example.com` |

**TTL (Time to Live):** How long resolvers cache the record. Low TTL (60s) = faster propagation of changes but more DNS queries. High TTL (86400s) = cached for 24 hours.

> **Also asked as:** "What is DNS?" — covered above.

---

## 5. What is the difference between HTTP and HTTPS?

HTTP (HyperText Transfer Protocol) is the protocol browsers and servers use to exchange data. HTTPS is HTTP with TLS encryption on top.

**HTTP — plain text, no encryption.**

```
Client → GET /login HTTP/1.1 → Server
Server → 200 OK + response   → Client
# All data is readable by anyone between client and server
```

**HTTPS — encrypted with TLS.**

```
Client → TLS handshake → Server   (negotiate encryption)
Client → GET /login (encrypted) → Server
Server → 200 OK (encrypted)     → Client
# Data is encrypted — interceptors see only ciphertext
```

**TLS handshake (simplified):**
```
1. Client sends supported TLS versions + cipher suites
2. Server responds with its certificate (public key, issued by CA)
3. Client verifies certificate against trusted CA store
4. Both sides derive session keys
5. Encrypted channel established
```

**Practical differences:**

| | HTTP | HTTPS |
|---|---|---|
| Port | 80 | 443 |
| Encryption | None | TLS (AES, ChaCha20) |
| Authentication | None | Server identity verified by CA |
| Data integrity | None | HMAC — tampering detected |
| SEO | Penalised by Google | Preferred |
| Required for | Legacy internal tools | Any public-facing service |

**In AWS:** ACM (AWS Certificate Manager) issues free TLS certificates. ALB terminates TLS — the backend EC2/container communicates with the ALB over plain HTTP internally (within the VPC). This is called TLS termination at the load balancer.

---

## 6. What is the difference between a public IP and a private IP?

**Private IP — used within a private network (LAN / VPC).**

Private IPs are not routable on the internet. Three reserved ranges (RFC 1918):

```
10.0.0.0    – 10.255.255.255   (10.0.0.0/8)    — large networks, VPCs
172.16.0.0  – 172.31.255.255   (172.16.0.0/12) — medium networks
192.168.0.0 – 192.168.255.255  (192.168.0.0/16) — home/office routers
```

Devices inside a VPC or office network use private IPs. They communicate with each other directly. To reach the internet, traffic must go through NAT (Network Address Translation).

**Public IP — globally routable on the internet.**

Every device that communicates directly on the internet needs a public IP. Public IPs are assigned by ISPs and cloud providers. In AWS, Elastic IPs are static public IPs you own.

**How they work together in AWS:**

```
EC2 instance (private IP: 10.0.1.45)
    ↓ outbound internet request
NAT Gateway (private IP: 10.0.0.10, public IP: 52.66.123.45)
    ↓ translates private → public
Internet → response comes back to 52.66.123.45 → NAT → 10.0.1.45
```

**In DevOps:**
- EC2 in a private subnet: has only a private IP, no direct internet access
- EC2 in a public subnet with EIP: has both private and public IP, directly reachable
- RDS instances: always private IP only — never expose a database to the internet

> **Also asked as:** "Difference between public & private IP?" — covered above.

---

## 7. What is subnetting and how does it work?

Subnetting divides a large IP network into smaller sub-networks (subnets). It controls traffic flow, isolates resources, and manages IP address allocation.

**Why subnet?**

Without subnetting, all devices in a network are on the same broadcast domain — every broadcast packet reaches every device. Subnetting isolates broadcast domains and allows routing between them.

**How it works — the subnet mask.**

A subnet mask tells you which part of an IP address is the network portion and which is the host portion:

```
IP address:   192.168.1.45
Subnet mask:  255.255.255.0   (or /24 in CIDR)

Network part: 192.168.1.0     (fixed — identifies the subnet)
Host part:    .45             (variable — identifies the device)
```

**Calculating usable hosts:**

```
/24 subnet → 256 addresses → 254 usable (first = network, last = broadcast)
/25 subnet → 128 addresses → 126 usable
/26 subnet → 64 addresses  → 62 usable
/28 subnet → 16 addresses  → 14 usable
```

**In AWS VPC:**

```
VPC CIDR: 10.0.0.0/16  (65,536 addresses)
  ├── Public subnet:  10.0.1.0/24  (254 hosts) — ALB, NAT Gateway
  ├── Private subnet: 10.0.2.0/24  (254 hosts) — EC2, EKS nodes
  └── DB subnet:      10.0.3.0/24  (254 hosts) — RDS (no internet route)
```

AWS reserves 5 IPs in each subnet (first 4 + last 1), so a /24 gives you 251 usable IPs.

---

## 8. What is CIDR notation and how do you read it?

CIDR (Classless Inter-Domain Routing) is a compact way to express an IP address range using a prefix length (the number after the slash).

**Reading CIDR:**

```
10.0.0.0/16
│       │
│       └── Prefix length: 16 bits are the network, 16 bits are hosts
└────────── Network address

/16 → 2^(32-16) = 2^16 = 65,536 IP addresses
/24 → 2^(32-24) = 2^8  = 256 IP addresses
/28 → 2^(32-28) = 2^4  = 16 IP addresses
/32 → single IP address (one host)
/0  → all IP addresses (0.0.0.0/0 = the entire internet)
```

**Common CIDR blocks and their sizes:**

| CIDR | Addresses | Common use |
|---|---|---|
| /8 | 16,777,216 | Very large orgs (10.0.0.0/8) |
| /16 | 65,536 | VPC CIDR |
| /24 | 256 | Subnet |
| /28 | 16 | Small subnet (database tier) |
| /32 | 1 | Single IP (Security Group rule for one IP) |
| /0 | All | Default route (0.0.0.0/0 = anywhere) |

**In Security Groups:**

```
0.0.0.0/0      → Allow from anywhere (dangerous for port 22)
10.0.0.0/8     → Allow from any private IP in 10.x.x.x range
52.66.123.45/32 → Allow from exactly one IP address
```

**Quick CIDR check:**

```bash
# Check what IPs are in a CIDR range:
ipcalc 10.0.1.0/24
# Network:   10.0.1.0
# Broadcast: 10.0.1.255
# HostMin:   10.0.1.1
# HostMax:   10.0.1.254
# Hosts/Net: 254
```

---

## 9. What is a routing table and how does traffic get routed?

A routing table is a set of rules that determines where network packets go based on their destination IP address. Every router (and every VPC subnet) has one.

**How routing works:**

```
Packet arrives with destination IP: 8.8.8.8
Router checks routing table (longest prefix match):

Destination     Gateway          Interface
10.0.0.0/16     local            eth0       ← local VPC traffic
0.0.0.0/0       igw-0abc123      eth0       ← everything else → internet

8.8.8.8 doesn't match 10.0.0.0/16 → matches 0.0.0.0/0 → send to IGW
```

Routing uses **longest prefix match** — the most specific route wins. `/32` beats `/24` beats `/16` beats `/0`.

**In AWS VPC:**

```
Public subnet route table:
  10.0.0.0/16   → local          (traffic within VPC stays local)
  0.0.0.0/0     → igw-0abc123   (internet-bound traffic → Internet Gateway)

Private subnet route table:
  10.0.0.0/16   → local          (VPC traffic stays local)
  0.0.0.0/0     → nat-0xyz789   (internet-bound traffic → NAT Gateway)
  10.10.0.0/16  → tgw-0def456   (traffic to on-premises → Transit Gateway)
```

A subnet is "public" because its route table has a route to an Internet Gateway. A subnet is "private" because its route table routes internet traffic through a NAT Gateway instead.

> **Also asked as:** "How does traffic reach your EC2 instance?" — traffic enters the VPC via IGW → route table directs it to the subnet → Security Group allows/denies → reaches the EC2 private IP.

---

## 10. What is the difference between an Internet Gateway and a NAT Gateway?

Both enable communication between a VPC and the internet, but in opposite directions.

**Internet Gateway (IGW) — bi-directional internet access for public resources.**

- Attached to the VPC (one per VPC)
- Allows inbound traffic from the internet TO your EC2 instance (if it has a public IP)
- Allows outbound traffic FROM your EC2 instance TO the internet
- Used by: public subnets (ALBs, bastion hosts, EC2 with public IPs)

```
Internet ←→ IGW ←→ Public subnet (EC2 with Elastic IP)
            ↑ bi-directional
```

**NAT Gateway — outbound-only internet access for private resources.**

- Sits in a public subnet (has a public IP itself)
- Allows EC2 instances in private subnets to initiate outbound connections (to download packages, call APIs)
- Blocks all inbound connections initiated from the internet
- Used by: private subnets (app servers, EKS nodes, Lambda)

```
Internet ← NAT Gateway ← Private subnet (EC2, no public IP)
         outbound only
         (internet cannot initiate connection TO the private EC2)
```

**Side-by-side comparison:**

| | Internet Gateway | NAT Gateway |
|---|---|---|
| Direction | Bi-directional | Outbound only |
| Used by | Public subnets | Private subnets |
| Inbound from internet | Yes (if EC2 has public IP) | No |
| Outbound to internet | Yes | Yes |
| Cost | Free | Charged per GB + hourly |
| Availability | Single, VPC-level | Deploy one per AZ for HA |

**Typical VPC architecture:**

```
Internet
    ↓
Internet Gateway (attached to VPC)
    ↓
Public subnet:   ALB (receives traffic), NAT Gateway (for private subnet egress)
    ↓
Private subnet:  EC2 / EKS nodes (outbound via NAT, no inbound from internet)
    ↓
DB subnet:       RDS (no internet route at all)
```

> **Also asked as:** "What is the difference between an Internet Gateway and a NAT Gateway?" — covered above.

---

## 11. What is the difference between an L4 and L7 load balancer?

Load balancers distribute traffic across multiple servers. The layer they operate at determines how much they understand about the traffic.

**L4 Load Balancer — Transport Layer (TCP/UDP).**

Operates on IP address and port only. It doesn't read the packet content. Routing decisions: "This TCP packet is for port 443 → forward to this backend IP."

```
Client: 1.2.3.4:51234 → L4 LB: 52.66.0.1:443 → Backend: 10.0.1.45:443
        (just TCP, no HTTP inspection)
```

- Ultra-low latency (no packet inspection)
- Preserves source IP
- Supports any TCP/UDP protocol (HTTP, HTTPS, gRPC, databases, custom binary)
- AWS equivalent: **NLB (Network Load Balancer)**

**L7 Load Balancer — Application Layer (HTTP/HTTPS).**

Reads and understands the HTTP request. Routing decisions based on: URL path, HTTP headers, cookies, query strings, hostname.

```
GET /api/users  → route to API servers
GET /static/    → route to CDN or S3
Host: admin.example.com → route to admin service
Cookie: session=abc → route to same backend (sticky sessions)
```

- Path-based routing (one LB, multiple services)
- SSL termination (handles TLS, backend gets plain HTTP)
- WAF integration (inspect and filter HTTP content)
- AWS equivalent: **ALB (Application Load Balancer)**

**Which to use:**

| Need | Use |
|---|---|
| HTTP/HTTPS routing, WAF, path-based rules | L7 (ALB) |
| Non-HTTP protocol, raw TCP, custom binary | L4 (NLB) |
| Static IP, ultra-low latency | L4 (NLB) |
| gRPC, WebSocket, HTTP/2 | L7 (ALB supports all) |

> **Also asked as:** "When do you use ALB vs NLB?" — see `aws/medium-questions.md Q3 and Q8` for AWS-specific implementation.

---

## 12. What happens at common ports — 80, 443, 22, 3306?

Ports are logical endpoints that identify specific services on a server. When a packet arrives at an IP, the port number tells the OS which application to hand it to.

**Well-known ports (0–1023):**

| Port | Protocol | Service | DevOps relevance |
|---|---|---|---|
| 22 | TCP | SSH | Remote server access; always restrict to known IPs in Security Groups |
| 25 | TCP | SMTP | Email — often blocked by cloud providers by default |
| 53 | TCP/UDP | DNS | Name resolution; UDP for queries, TCP for zone transfers |
| 80 | TCP | HTTP | Web traffic (unencrypted); redirect to 443 in production |
| 443 | TCP | HTTPS | Encrypted web traffic; ALB listener |
| 3306 | TCP | MySQL/MariaDB | Database; Security Group should allow only app servers, never 0.0.0.0/0 |
| 5432 | TCP | PostgreSQL | Database; same rule — private access only |
| 6379 | TCP | Redis | Cache; private subnet only |
| 8080 | TCP | HTTP alt | Common for app servers behind a load balancer |
| 9090 | TCP | Prometheus | Metrics scraping |
| 9100 | TCP | Node Exporter | Host metrics for Prometheus |
| 27017 | TCP | MongoDB | Database; private access only |

**Security Group rules based on ports:**

```
# ALB Security Group:
Inbound 80  from 0.0.0.0/0  → public HTTP
Inbound 443 from 0.0.0.0/0  → public HTTPS

# EC2 App Server Security Group:
Inbound 8080 from ALB Security Group only  → app traffic from ALB
Inbound 22   from bastion/VPN IP only      → SSH restricted

# RDS Security Group:
Inbound 3306 from App Server Security Group only  → never from internet
```

---

## 13. What happens when you ping a server?

`ping` uses the ICMP (Internet Control Message Protocol) — it sends an Echo Request to the target IP and expects an Echo Reply.

**What actually happens:**

```
1. ping 8.8.8.8
2. OS creates ICMP Echo Request packet (type 8, code 0)
3. Kernel routes packet: check routing table → default gateway → NIC
4. Packet travels across the internet (multiple hops via routers)
5. 8.8.8.8 receives Echo Request, sends back Echo Reply (type 0, code 0)
6. ping reports: round-trip time (RTT) in milliseconds

64 bytes from 8.8.8.8: icmp_seq=1 ttl=118 time=8.23 ms
```

**What ping tells you:**

| Result | Meaning |
|---|---|
| Reply received | Host is reachable, ICMP is allowed |
| Request timeout | Host unreachable OR ICMP blocked (firewall/Security Group) |
| High RTT (>100ms) | Network latency — routing issue or geographic distance |
| Packet loss | Network congestion or unstable connection |

**TTL (Time to Live):** Each hop decrements TTL by 1. If TTL reaches 0, the packet is dropped and an ICMP "TTL exceeded" message is sent back. `ping` shows the remaining TTL in the response.

**Important:** In AWS, Security Groups don't allow ICMP by default. You must explicitly allow `ICMP - Echo Request` inbound if you want ping to work on an EC2 instance.

```bash
# Basic ping
ping -c 4 8.8.8.8        # Send 4 packets

# Ping with specific packet size
ping -s 1400 8.8.8.8     # Test with larger packets (MTU issues)
```

---

## 14. What are traceroute and netstat, and how do you use them?

**traceroute — trace the path packets take to a destination.**

`traceroute` sends packets with incrementally increasing TTL values. When TTL expires at each hop, the router sends back an ICMP "TTL exceeded" — revealing each router's IP address.

```bash
traceroute google.com

# Output:
 1  10.0.0.1 (10.0.0.1)       0.8 ms    ← your default gateway
 2  172.16.0.1 (172.16.0.1)   3.2 ms    ← ISP router
 3  203.0.113.1                8.1 ms    ← transit router
 4  * * *                               ← router blocking ICMP (firewall)
 5  142.250.195.46             12.4 ms   ← Google
```

`* * *` means the router at that hop is blocking ICMP — it doesn't mean the connection is broken (traffic still passes through).

**What traceroute tells you:**

| Observation | Meaning |
|---|---|
| High RTT at one hop | Congestion or long physical distance at that point |
| `* * *` at every hop after N | Firewall blocking ICMP beyond hop N |
| Sudden RTT jump between two hops | Problem between those two routers |

```bash
# Linux:
traceroute google.com

# macOS:
traceroute google.com

# Windows:
tracert google.com

# TCP-based (bypasses ICMP firewalls):
traceroute -T -p 80 google.com
```

---

**netstat — show active network connections and listening ports.**

```bash
netstat -tlnp
# -t TCP
# -l Listening
# -n Numeric (no hostname resolution)
# -p Show process name/PID

# Output:
Proto  Recv-Q  Send-Q  Local Address    Foreign Address  State    PID/Program
tcp    0       0       0.0.0.0:80       0.0.0.0:*        LISTEN   1234/nginx
tcp    0       0       0.0.0.0:443      0.0.0.0:*        LISTEN   1234/nginx
tcp    0       0       127.0.0.1:5432   0.0.0.0:*        LISTEN   5678/postgres
```

**`ss` is the modern replacement for `netstat`:**

```bash
ss -tlnp     # Same flags, faster output
ss -s        # Summary statistics
ss -tn       # Show established TCP connections
```

**Common use cases:**

```bash
# Is nginx actually listening on port 80?
ss -tlnp | grep :80

# What process is using port 8080?
ss -tlnp | grep :8080

# How many established connections are there?
ss -tn | grep ESTABLISHED | wc -l

# Show connections to a specific remote IP:
ss -tn dst 8.8.8.8
```

> **Also asked as:** "Traceroute & Netstat basics" — covered above.

