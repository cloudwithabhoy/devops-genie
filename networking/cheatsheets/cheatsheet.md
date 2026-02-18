# Networking Cheatsheet

> Quick-reference commands and concepts for networking.

## Core Commands / Concepts

### Diagnostic Tools
```bash
# Test connectivity
ping <host>

# Trace network path
traceroute <host>       # Linux
tracert <host>          # Windows

# DNS lookup
nslookup <domain>
dig <domain>

# Show open ports and connections
netstat -tulnp
ss -tulnp

# Capture packets
tcpdump -i eth0 port 80

# Show routing table
ip route show

# Show network interfaces
ip addr show
ifconfig
```

### Common Ports
| Port | Protocol | Service |
|------|----------|---------|
| 22 | TCP | SSH |
| 25 | TCP | SMTP |
| 53 | TCP/UDP | DNS |
| 80 | TCP | HTTP |
| 443 | TCP | HTTPS |
| 3306 | TCP | MySQL |
| 5432 | TCP | PostgreSQL |
| 6379 | TCP | Redis |

<!-- Add more ports, protocols, and commands here -->

## Subnetting Quick Reference

<!-- Add CIDR notation, subnet masks, and examples here -->

## Common Patterns

<!-- Add common networking patterns, e.g., load balancer configs, firewall rules -->
