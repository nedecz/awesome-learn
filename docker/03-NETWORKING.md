# Docker Networking

## Table of Contents

1. [Overview](#overview)
   - [The Docker Networking Model](#the-docker-networking-model)
   - [Container Network Model (CNM)](#container-network-model-cnm)
2. [Network Drivers](#network-drivers)
   - [Driver Comparison](#driver-comparison)
3. [Bridge Networking](#bridge-networking)
   - [Default Bridge Network](#default-bridge-network)
   - [User-Defined Bridge Networks](#user-defined-bridge-networks)
   - [Default vs User-Defined Bridge](#default-vs-user-defined-bridge)
4. [Host Networking](#host-networking)
5. [Overlay Networking](#overlay-networking)
   - [Overlay for Docker Swarm](#overlay-for-docker-swarm)
   - [Overlay for Standalone Containers](#overlay-for-standalone-containers)
6. [Macvlan Networking](#macvlan-networking)
7. [IPvlan Networking](#ipvlan-networking)
8. [DNS and Service Discovery](#dns-and-service-discovery)
   - [Embedded DNS Server](#embedded-dns-server)
   - [DNS Resolution Flow](#dns-resolution-flow)
   - [Custom DNS Configuration](#custom-dns-configuration)
9. [Port Mapping and Publishing](#port-mapping-and-publishing)
   - [Port Mapping Patterns](#port-mapping-patterns)
   - [Publishing All Ports](#publishing-all-ports)
10. [Container-to-Container Communication](#container-to-container-communication)
    - [Same Network](#same-network)
    - [Cross-Network Communication](#cross-network-communication)
11. [Network Security](#network-security)
    - [Network Isolation](#network-isolation)
    - [Inter-Container Firewall Rules](#inter-container-firewall-rules)
    - [Encrypting Overlay Traffic](#encrypting-overlay-traffic)
12. [Docker Network CLI Commands](#docker-network-cli-commands)
    - [docker network create](#docker-network-create)
    - [docker network ls](#docker-network-ls)
    - [docker network inspect](#docker-network-inspect)
    - [docker network connect](#docker-network-connect)
    - [docker network disconnect](#docker-network-disconnect)
    - [docker network rm](#docker-network-rm)
13. [Troubleshooting Networking Issues](#troubleshooting-networking-issues)
    - [Common Issues](#common-issues)
    - [Debugging Tools](#debugging-tools)
    - [Diagnostic Commands](#diagnostic-commands)
14. [Best Practices](#best-practices)
15. [Next Steps](#next-steps)
16. [Version History](#version-history)

---

## Overview

This document provides a comprehensive guide to **Docker networking** — from the fundamental networking model and built-in drivers, through DNS service discovery and port mapping, to network security and troubleshooting.

### Target Audience

- **Developers** connecting multi-container applications on a single host
- **DevOps Engineers** designing container networking for CI/CD and staging environments
- **SREs** debugging container connectivity issues in production
- **Platform Engineers** architecting multi-host container networking with Swarm or overlay drivers

### Scope

- Docker's Container Network Model (CNM) and its components
- All built-in network drivers: bridge, host, overlay, macvlan, ipvlan, none
- DNS-based service discovery within Docker networks
- Port mapping and container-to-container communication patterns
- Network security, isolation, and encryption
- CLI reference for Docker network management
- Troubleshooting network connectivity issues

### The Docker Networking Model

Docker networking is built on the **Container Network Model (CNM)**, which defines three building blocks:

```
┌────────────────────────────────────────────────────────────────────┐
│                    Container Network Model (CNM)                    │
│                                                                    │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐ │
│  │   Sandbox     │    │   Endpoint   │    │      Network         │ │
│  │              │    │              │    │                      │ │
│  │ Network      │    │ Virtual      │    │ Group of endpoints   │ │
│  │ namespace    │    │ network      │    │ that can             │ │
│  │ for a        │    │ interface    │    │ communicate          │ │
│  │ container    │    │ (veth pair)  │    │ directly             │ │
│  │              │    │              │    │                      │ │
│  │ Contains:    │    │ Connects a   │    │ Implemented by       │ │
│  │ • interfaces │    │ sandbox to   │    │ a network driver     │ │
│  │ • routes     │    │ a network    │    │ (bridge, overlay,    │ │
│  │ • DNS config │    │              │    │  host, macvlan)      │ │
│  └──────────────┘    └──────────────┘    └──────────────────────┘ │
└────────────────────────────────────────────────────────────────────┘
```

### Container Network Model (CNM)

The CNM is implemented by **libnetwork**, which provides the networking subsystem for Docker. When a container starts:

1. A **sandbox** (network namespace) is created for the container
2. An **endpoint** (veth pair) connects the sandbox to the network
3. The **network** (bridge, overlay, etc.) provides connectivity between endpoints

```
┌───────────────────────────────────────────────────────────────┐
│                        Docker Host                             │
│                                                               │
│  ┌──────────────────┐          ┌──────────────────┐           │
│  │  Container A      │          │  Container B      │           │
│  │  ┌──────────────┐│          │┌──────────────┐  │           │
│  │  │  Sandbox     ││          ││  Sandbox     │  │           │
│  │  │  (netns)     ││          ││  (netns)     │  │           │
│  │  │   eth0       ││          ││   eth0       │  │           │
│  │  └──────┬───────┘│          │└──────┬───────┘  │           │
│  └─────────┼────────┘          └───────┼──────────┘           │
│            │  Endpoint (veth)          │  Endpoint (veth)      │
│            │                           │                       │
│       ┌────┴───────────────────────────┴────┐                 │
│       │         Network (docker0 bridge)     │                 │
│       │         172.17.0.0/16               │                 │
│       └─────────────────┬───────────────────┘                 │
│                         │                                     │
│                    ┌────┴────┐                                 │
│                    │  eth0   │   Host network interface        │
│                    └────┬────┘                                 │
└─────────────────────────┼─────────────────────────────────────┘
                          │
                     Physical Network
```

---

## Network Drivers

Docker provides several built-in network drivers, each suited to different use cases.

### Driver Comparison

| Driver | Scope | Use Case | DNS Discovery | Isolation | Multi-Host |
|---|---|---|---|---|---|
| **bridge** | Local | Default; single-host container networking | ✅ (user-defined only) | ✅ Per-network | ❌ |
| **host** | Local | Maximum performance; no network isolation | ❌ | ❌ None | ❌ |
| **overlay** | Swarm / Global | Multi-host container networking | ✅ | ✅ Per-network | ✅ |
| **macvlan** | Local | Containers need real IPs on physical network | ❌ | ✅ Per-network | ❌ |
| **ipvlan** | Local | Like macvlan but shares host MAC address | ❌ | ✅ Per-network | ❌ |
| **none** | Local | Completely disable networking | ❌ | ✅ Total | ❌ |

---

## Bridge Networking

The **bridge** driver is the default network driver for Docker. It creates a software bridge (virtual switch) on the host that containers connect to via virtual Ethernet (veth) pairs.

### Default Bridge Network

When Docker is installed, it automatically creates a default bridge network named `bridge` (backed by the `docker0` Linux bridge device).

```bash
# The default bridge network is created automatically
docker network ls
# NETWORK ID     NAME      DRIVER    SCOPE
# a1b2c3d4e5f6   bridge    bridge    local
# f6e5d4c3b2a1   host      host      local
# 1a2b3c4d5e6f   none      null      local

# Run containers on the default bridge
docker run -d --name web1 nginx:1.25
docker run -d --name web2 nginx:1.25

# Containers get IPs from the 172.17.0.0/16 subnet
docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web1
# 172.17.0.2

docker inspect --format '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web2
# 172.17.0.3
```

```
Default Bridge Network:

  ┌──────────────┐     ┌──────────────┐
  │   web1       │     │   web2       │
  │  172.17.0.2  │     │  172.17.0.3  │
  │   eth0       │     │   eth0       │
  └──────┬───────┘     └──────┬───────┘
         │ veth                │ veth
         │                     │
    ┌────┴─────────────────────┴────┐
    │        docker0 bridge          │
    │        172.17.0.1              │
    └───────────────┬───────────────┘
                    │
                    │ iptables NAT
                    │
               ┌────┴────┐
               │  eth0   │  Host: 192.168.1.100
               └────┬────┘
                    │
               Physical Network
```

> **Limitation:** On the default bridge, containers can communicate by IP address but **not by container name**. DNS-based discovery requires a user-defined bridge network.

### User-Defined Bridge Networks

User-defined bridge networks provide **automatic DNS resolution** between containers, better isolation, and configurable subnets.

```bash
# Create a user-defined bridge network
docker network create \
    --driver bridge \
    --subnet 172.20.0.0/16 \
    --gateway 172.20.0.1 \
    --ip-range 172.20.1.0/24 \
    my-app-network

# Run containers on the custom network
docker run -d --name api --network my-app-network myapp-api:1.0
docker run -d --name db --network my-app-network postgres:16

# Containers can reach each other by NAME
docker exec api ping -c 3 db
# PING db (172.20.1.2): 56 data bytes
# 64 bytes from 172.20.1.2: seq=0 ttl=64 time=0.089 ms

# DNS resolution works automatically
docker exec api nslookup db
# Server:    127.0.0.11
# Address:   127.0.0.11:53
# Name:      db
# Address:   172.20.1.2
```

### Default vs User-Defined Bridge

| Feature | Default Bridge (`bridge`) | User-Defined Bridge |
|---|---|---|
| **DNS resolution** | ❌ By IP only | ✅ By container name |
| **Automatic connection** | ✅ All containers by default | ❌ Must specify `--network` |
| **Isolation** | All containers share one network | Containers only see their network |
| **Custom subnets** | ❌ Fixed 172.17.0.0/16 | ✅ Configurable |
| **Live connect/disconnect** | ❌ | ✅ `docker network connect/disconnect` |
| **Link legacy support** | ✅ `--link` (deprecated) | Not needed (DNS works) |

**Recommendation:** Always use user-defined bridge networks. The default bridge is retained for backward compatibility.

---

## Host Networking

With the **host** network driver, the container shares the host's network namespace directly. There is no network isolation — the container uses the host's IP address and ports.

```bash
# Run a container with host networking
docker run -d --name web --network host nginx:1.25

# nginx listens on port 80 of the HOST directly
# No port mapping needed (-p is ignored with host networking)
curl http://localhost:80
```

```
Host Networking:

  ┌────────────────────────────────────────────┐
  │                Docker Host                  │
  │                                            │
  │  ┌──────────────┐                          │
  │  │  Container    │  Uses host's eth0        │
  │  │  (nginx)     │  directly — no veth,     │
  │  │              │  no bridge, no NAT       │
  │  └──────────────┘                          │
  │                                            │
  │  ┌──────────────┐                          │
  │  │   eth0       │  Host: 192.168.1.100     │
  │  │   :80 nginx  │  Port 80 = nginx         │
  │  └──────┬───────┘                          │
  └─────────┼──────────────────────────────────┘
            │
       Physical Network
```

| Advantage | Disadvantage |
|---|---|
| No NAT overhead — maximum network performance | No port isolation — port conflicts with host services |
| Lowest latency for network-intensive applications | Container can see all host network interfaces |
| Simplest networking model | Cannot run multiple containers on the same port |
| No bridge or veth overhead | Linux only (not supported on Docker Desktop for Mac/Windows) |

**Use cases:** High-performance applications, monitoring agents that need host network visibility, sidecar containers that need to bind to the host's loopback.

---

## Overlay Networking

The **overlay** driver creates a distributed network across multiple Docker hosts, enabling containers on different machines to communicate as if they were on the same local network.

### Overlay for Docker Swarm

Overlay networks are the default networking solution for Docker Swarm services.

```bash
# Initialize a Swarm (on the manager node)
docker swarm init --advertise-addr 192.168.1.100

# Create an overlay network
docker network create \
    --driver overlay \
    --subnet 10.0.1.0/24 \
    --attachable \
    my-overlay-net

# Deploy a service on the overlay network
docker service create \
    --name web \
    --network my-overlay-net \
    --replicas 3 \
    --publish published=80,target=80 \
    nginx:1.25
```

```
Overlay Network Across Hosts:

  Host A (192.168.1.100)               Host B (192.168.1.101)
  ┌─────────────────────┐              ┌─────────────────────┐
  │  ┌─────┐  ┌─────┐  │              │  ┌─────┐  ┌─────┐  │
  │  │web.1│  │web.2│  │              │  │web.3│  │ db  │  │
  │  │10.0.│  │10.0.│  │              │  │10.0.│  │10.0.│  │
  │  │1.2  │  │1.3  │  │              │  │1.4  │  │1.5  │  │
  │  └──┬──┘  └──┬──┘  │              │  └──┬──┘  └──┬──┘  │
  │     │        │      │              │     │        │      │
  │  ┌──┴────────┴───┐  │   VXLAN     │  ┌──┴────────┴───┐  │
  │  │  br0 (overlay) │──┼──tunnel────┼──│  br0 (overlay) │  │
  │  │  10.0.1.0/24  │  │  UDP 4789  │  │  10.0.1.0/24  │  │
  │  └───────────────┘  │              │  └───────────────┘  │
  │         │            │              │         │            │
  │    ┌────┴────┐       │              │    ┌────┴────┐       │
  │    │  eth0   │       │              │    │  eth0   │       │
  └────┼─────────┼───────┘              └────┼─────────┼───────┘
       │         │                           │         │
  ─────┴─────────┴───────────────────────────┴─────────┴─────────
                        Physical Network
```

### Overlay for Standalone Containers

To attach standalone containers (not Swarm services) to an overlay network, use the `--attachable` flag:

```bash
# Create an attachable overlay network
docker network create --driver overlay --attachable my-attachable-overlay

# Run standalone containers on the overlay (from any Swarm node)
docker run -d --name app1 --network my-attachable-overlay myapp:1.0
```

---

## Macvlan Networking

The **macvlan** driver assigns a unique MAC address to each container, making it appear as a physical device on the network. Containers get IP addresses from the physical network's DHCP server or a static range.

```bash
# Create a macvlan network
docker network create \
    --driver macvlan \
    --subnet 192.168.1.0/24 \
    --gateway 192.168.1.1 \
    -o parent=eth0 \
    my-macvlan-net

# Run a container with a specific IP on the physical network
docker run -d --name db \
    --network my-macvlan-net \
    --ip 192.168.1.50 \
    postgres:16
```

**Use cases:** Legacy applications requiring specific IPs, applications that need to be directly accessible on the LAN, migrating from VMs to containers without network changes.

> **Note:** Macvlan requires the host network interface to be in promiscuous mode. Some cloud providers and wireless interfaces do not support this.

---

## IPvlan Networking

**IPvlan** is similar to macvlan but all containers share the host's MAC address. This avoids promiscuous mode requirements and works better in environments that restrict the number of MAC addresses per port.

```bash
# Create an ipvlan L2 network
docker network create \
    --driver ipvlan \
    --subnet 192.168.1.0/24 \
    --gateway 192.168.1.1 \
    -o parent=eth0 \
    -o ipvlan_mode=l2 \
    my-ipvlan-net

# Create an ipvlan L3 network (routing mode)
docker network create \
    --driver ipvlan \
    --subnet 10.10.0.0/24 \
    -o parent=eth0 \
    -o ipvlan_mode=l3 \
    my-ipvlan-l3
```

| Mode | Description | Use Case |
|---|---|---|
| L2 (default) | Containers share the host's link layer | LAN-connected containers |
| L3 | Containers are routed by the host | Cross-subnet container communication |

---

## DNS and Service Discovery

### Embedded DNS Server

Docker provides a built-in DNS server at `127.0.0.11` for containers on **user-defined networks**. This DNS server resolves container names, service names, and network aliases to IP addresses.

```
DNS Resolution in Docker:

  ┌──────────────────┐
  │   Container A     │
  │                  │
  │  ping "db"      │
  │       │          │
  │       ▼          │
  │  /etc/resolv.conf│
  │  nameserver      │
  │  127.0.0.11      │
  └────────┬─────────┘
           │
           ▼
  ┌──────────────────────────────┐
  │  Docker Embedded DNS          │
  │  127.0.0.11                  │
  │                              │
  │  1. Check container names    │
  │  2. Check network aliases    │
  │  3. Check service names      │
  │  4. Forward to host DNS      │
  └──────────────────────────────┘
```

### DNS Resolution Flow

```bash
# Container names are resolvable on user-defined networks
docker network create app-net
docker run -d --name postgres --network app-net postgres:16
docker run -d --name redis --network app-net redis:7

# From another container, resolve by name
docker run --rm --network app-net alpine nslookup postgres
# Name:      postgres
# Address 1: 172.18.0.2 postgres.app-net

# Network aliases provide additional DNS names
docker run -d --name cache \
    --network app-net \
    --network-alias redis-primary \
    --network-alias session-store \
    redis:7

# All three names resolve to the same container
docker run --rm --network app-net alpine nslookup cache
docker run --rm --network app-net alpine nslookup redis-primary
docker run --rm --network app-net alpine nslookup session-store
```

### Custom DNS Configuration

```bash
# Override DNS servers for a container
docker run -d --name app \
    --dns 8.8.8.8 \
    --dns 8.8.4.4 \
    myapp:1.0

# Add DNS search domains
docker run -d --name app \
    --dns-search example.com \
    --dns-search internal.example.com \
    myapp:1.0

# Add custom host entries (like /etc/hosts)
docker run -d --name app \
    --add-host db.local:192.168.1.50 \
    --add-host cache.local:192.168.1.51 \
    myapp:1.0
```

---

## Port Mapping and Publishing

Port mapping allows external traffic to reach containers by mapping host ports to container ports. Docker uses **iptables** rules to perform the NAT translation.

### Port Mapping Patterns

```bash
# Map host port 8080 to container port 80
docker run -d -p 8080:80 nginx:1.25

# Map to a specific host interface
docker run -d -p 127.0.0.1:8080:80 nginx:1.25

# Map with a random host port
docker run -d -p 80 nginx:1.25
docker port <container-id>
# 80/tcp -> 0.0.0.0:32768

# Map UDP port
docker run -d -p 5514:514/udp syslog-server

# Map multiple ports
docker run -d \
    -p 80:80 \
    -p 443:443 \
    -p 8080:8080 \
    my-web-server:1.0

# Map a range of ports
docker run -d -p 8000-8010:8000-8010 my-service:1.0
```

| Pattern | Command | Description |
|---|---|---|
| `hostPort:containerPort` | `-p 8080:80` | Map specific host port to container port |
| `ip:hostPort:containerPort` | `-p 127.0.0.1:8080:80` | Bind to specific host interface |
| `containerPort` | `-p 80` | Map random host port to container port |
| `hostPort:containerPort/protocol` | `-p 5514:514/udp` | Map with specific protocol (tcp/udp) |
| `hostPortRange:containerPortRange` | `-p 8000-8010:8000-8010` | Map a range of ports |

### Publishing All Ports

```bash
# Publish all ports declared with EXPOSE in the Dockerfile
docker run -d -P nginx:1.25

# Check the mapped ports
docker port <container-id>
# 80/tcp -> 0.0.0.0:32769
# 443/tcp -> 0.0.0.0:32770
```

---

## Container-to-Container Communication

### Same Network

Containers on the same user-defined network can communicate freely using container names.

```bash
# Create a network and run containers
docker network create backend
docker run -d --name api --network backend myapp-api:1.0
docker run -d --name db --network backend postgres:16
docker run -d --name cache --network backend redis:7

# API can reach the database by name
docker exec api ping -c 1 db
# PING db (172.20.0.3): 56 data bytes
# 64 bytes from 172.20.0.3: seq=0 ttl=64 time=0.085 ms

# Application code uses container names as hostnames
# DATABASE_URL=postgresql://user:pass@db:5432/myapp
# REDIS_URL=redis://cache:6379
```

### Cross-Network Communication

Containers on different networks are **isolated by default**. To enable communication, connect a container to multiple networks.

```bash
# Create two networks
docker network create frontend
docker network create backend

# API server needs access to both networks
docker run -d --name api --network backend myapp-api:1.0
docker network connect frontend api

# Web server only on frontend
docker run -d --name web --network frontend nginx:1.25

# Database only on backend
docker run -d --name db --network backend postgres:16

# web → api (via frontend) ✅
# api → db  (via backend)  ✅
# web → db  (different networks, no connection) ❌
```

```
Cross-Network Isolation:

  ┌──────────────────────────────────────────────────────────────┐
  │                                                              │
  │   Frontend Network                Backend Network            │
  │   ┌──────────────┐               ┌──────────────┐           │
  │   │              │               │              │           │
  │   │  ┌────┐      │               │    ┌─────┐   │           │
  │   │  │web │      │               │    │ db  │   │           │
  │   │  └──┬─┘      │               │    └──┬──┘   │           │
  │   │     │        │               │       │      │           │
  │   │     │   ┌────┴───────────────┴────┐  │      │           │
  │   │     │   │         api             │  │      │           │
  │   │     │   │  (connected to both)    │  │      │           │
  │   │     │   └─────────────────────────┘  │      │           │
  │   │     │        │               │       │      │           │
  │   └─────┼────────┘               └───────┼──────┘           │
  │                                                              │
  │   web ──► api ✅                  api ──► db ✅              │
  │   web ──► db  ❌ (isolated)                                  │
  └──────────────────────────────────────────────────────────────┘
```

---

## Network Security

### Network Isolation

Docker networks provide isolation at the network layer. Containers on different networks cannot communicate unless explicitly connected.

```bash
# Create isolated networks for different tiers
docker network create --internal dmz-net       # --internal: no outbound internet
docker network create app-net
docker network create data-net

# Containers on --internal networks cannot reach the internet
docker run --rm --network dmz-net alpine ping -c 1 8.8.8.8
# ping: bad address '8.8.8.8' — no external access
```

### Inter-Container Firewall Rules

Docker manipulates **iptables** to control container traffic. You can inspect and customize these rules.

```bash
# View Docker-related iptables rules
iptables -L -n -v --line-numbers
iptables -t nat -L -n -v

# Docker creates these chains:
# DOCKER          — container port mappings
# DOCKER-ISOLATION-STAGE-1  — inter-network isolation
# DOCKER-ISOLATION-STAGE-2  — inter-network isolation (continued)
# DOCKER-USER     — user-defined rules (applied BEFORE Docker rules)
```

```bash
# Add custom rules in the DOCKER-USER chain
# Block all traffic from containers to a specific external IP
iptables -I DOCKER-USER -d 10.0.0.50 -j DROP

# Allow only specific container subnet to reach external service
iptables -I DOCKER-USER -s 172.20.0.0/16 -d 10.0.0.50 -p tcp --dport 5432 -j ACCEPT
iptables -A DOCKER-USER -d 10.0.0.50 -j DROP
```

### Encrypting Overlay Traffic

Overlay networks support IPsec encryption for data in transit between hosts.

```bash
# Create an encrypted overlay network
docker network create \
    --driver overlay \
    --opt encrypted \
    secure-overlay

# All traffic between containers on this network is encrypted
# Uses IPsec ESP in tunnel mode
docker service create \
    --name secure-api \
    --network secure-overlay \
    myapp:1.0
```

> **Note:** Encrypted overlays add CPU overhead and latency. Benchmark your workload before enabling in performance-sensitive environments.

---

## Docker Network CLI Commands

### docker network create

```bash
# Basic bridge network
docker network create my-network

# Bridge with custom subnet and gateway
docker network create \
    --driver bridge \
    --subnet 172.25.0.0/16 \
    --gateway 172.25.0.1 \
    --ip-range 172.25.1.0/24 \
    custom-bridge

# Internal network (no external access)
docker network create --internal isolated-net

# Overlay network with encryption
docker network create \
    --driver overlay \
    --opt encrypted \
    --attachable \
    secure-overlay

# Network with custom MTU and bridge name
docker network create \
    -o com.docker.network.bridge.name=br-custom \
    -o com.docker.network.driver.mtu=9000 \
    jumbo-net
```

### docker network ls

```bash
# List all networks
docker network ls

# Filter by driver
docker network ls --filter driver=bridge

# Filter by name pattern
docker network ls --filter name=app

# Custom format
docker network ls --format "table {{.Name}}\t{{.Driver}}\t{{.Scope}}"
```

### docker network inspect

```bash
# Inspect a network (shows containers, config, IPAM)
docker network inspect my-network

# Get specific fields with Go template
docker network inspect --format '{{range .Containers}}{{.Name}}: {{.IPv4Address}}{{"\n"}}{{end}}' my-network

# Get the subnet
docker network inspect --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}' my-network
```

### docker network connect

```bash
# Connect a running container to an additional network
docker network connect my-network my-container

# Connect with a specific IP address
docker network connect --ip 172.25.1.100 my-network my-container

# Connect with network aliases
docker network connect --alias db-primary my-network my-container
```

### docker network disconnect

```bash
# Disconnect a container from a network
docker network disconnect my-network my-container

# Force disconnect
docker network disconnect -f my-network my-container
```

### docker network rm

```bash
# Remove a network (must have no connected containers)
docker network rm my-network

# Remove multiple networks
docker network rm net1 net2 net3

# Remove all unused networks
docker network prune -f

# Remove unused networks older than 24 hours
docker network prune --filter "until=24h" -f
```

---

## Troubleshooting Networking Issues

### Common Issues

| Issue | Possible Cause | Solution |
|---|---|---|
| Container cannot resolve other container names | Using default bridge network | Switch to a user-defined bridge network |
| Port already in use | Host port conflict | Use a different host port or stop the conflicting service |
| Container cannot reach the internet | Missing DNS configuration or `--internal` network | Check `/etc/resolv.conf` in container; verify network is not `--internal` |
| Containers on same network cannot communicate | Firewall rules or network misconfiguration | Check `iptables` rules; verify containers are on the same network |
| Overlay network not working | Firewall blocking VXLAN (UDP 4789) | Open UDP 4789, TCP/UDP 7946 between Docker hosts |
| Slow network performance | MTU mismatch between container and host | Check and align MTU: `docker network inspect` vs host `ip link` |
| Port mapping not accessible externally | Bound to `127.0.0.1` | Use `-p 0.0.0.0:8080:80` or `-p 8080:80` |

### Debugging Tools

```bash
# Install debugging tools in a container
docker run --rm --network my-network nicolaka/netshoot \
    nslookup my-service

# Common debugging commands in netshoot
docker run --rm -it --network my-network nicolaka/netshoot bash
# Inside the container:
# nslookup <service-name>
# dig <service-name>
# ping <service-name>
# traceroute <service-name>
# curl http://<service-name>:<port>/health
# tcpdump -i eth0 port 80
# ss -tlnp
# ip addr show
# ip route
```

### Diagnostic Commands

```bash
# Check container's network settings
docker inspect --format '{{json .NetworkSettings}}' my-container | jq .

# List ports mapped for a container
docker port my-container

# Check container's DNS configuration
docker exec my-container cat /etc/resolv.conf

# Check container's network interfaces
docker exec my-container ip addr show

# Check container's routing table
docker exec my-container ip route

# Test connectivity from container
docker exec my-container ping -c 3 google.com
docker exec my-container wget -qO- --timeout=5 http://api:8080/health

# View Docker's iptables rules
iptables -t nat -L DOCKER -n -v
iptables -L DOCKER-USER -n -v

# Check bridge network details on the host
brctl show
# Or with modern tools
bridge link show
ip link show type bridge
```

---

## Best Practices

1. **Always use user-defined bridge networks** — Never rely on the default bridge for production workloads. User-defined networks provide DNS discovery, better isolation, and live connect/disconnect.

2. **Use network aliases for service discovery** — Assign meaningful aliases that describe the service role rather than relying on container names.

3. **Bind to specific interfaces** — Use `-p 127.0.0.1:8080:80` instead of `-p 8080:80` for services that should only be accessible locally.

4. **Use `--internal` networks for databases** — Create internal networks for data-tier containers that should not have outbound internet access.

5. **One network per concern** — Separate frontend, backend, and data networks. Connect containers to multiple networks as needed.

6. **Monitor network performance** — Use `docker stats` and tools like `iperf3` to benchmark container network throughput.

7. **Set appropriate MTU values** — Match container MTU to the underlying network, especially in cloud and overlay environments where encapsulation reduces effective MTU.

8. **Document port mappings** — Use Docker Compose or explicit `-p` flags to document which ports are exposed and why.

9. **Prefer overlay encryption for sensitive traffic** — Enable `--opt encrypted` on overlay networks carrying sensitive data between hosts.

10. **Clean up unused networks** — Run `docker network prune` regularly to remove stale networks.

---

## Next Steps

Continue your Docker learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | Docker Overview | Core Docker concepts, architecture, and CLI essentials |
| [01-IMAGES-AND-DOCKERFILES.md](01-IMAGES-AND-DOCKERFILES.md) | Images & Dockerfiles | Building images, Dockerfile instructions, multi-stage builds |
| [02-CONTAINER-RUNTIME.md](02-CONTAINER-RUNTIME.md) | Container Runtime | Docker Engine, containerd, runc, and container lifecycle internals |

---

## Version History

| Version | Date | Changes |
|---|---|---|
| 1.0 | 2025 | Initial Docker Networking documentation |
