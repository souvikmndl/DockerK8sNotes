# Docker Networking — Notes & Reference

A working engineer's notes on how Docker networking actually works: the drivers you'll use, the Linux primitives underneath, and the exact path a packet takes from inside a container out to the internet and back.

---

## 1. The Mental Model

Docker networking is not magic — it's a thin orchestration layer over standard Linux kernel features:

- **Network namespaces** — give each container its own isolated network stack (its own interfaces, routing table, iptables rules, `/proc/net`).
- **veth pairs** — virtual "patch cables" with two ends; one end sits in the container's namespace, the other on the host, plugged into a bridge.
- **Linux bridge** — a virtual L2 switch (`docker0` by default) living in the host namespace.
- **iptables / netfilter** — does the NAT, masquerading, port forwarding, and inter-container firewalling.
- **conntrack** — the connection-tracking table that remembers NAT translations so return packets find their way home.

Once you internalize that Docker is *"namespaces + veth + bridge + iptables"*, everything else is detail.

---

## 2. The Network Drivers

| Driver | Scope | Use case |
|---|---|---|
| `bridge` | single host | **Default.** Containers on a private L2 segment, NAT to the outside. |
| `host` | single host | Container shares the host's network namespace directly. No isolation, no NAT, best raw performance. |
| `none` | single host | No networking at all (only loopback). Full isolation. |
| `overlay` | multi-host | Connects containers across multiple Docker hosts (Swarm / multi-node). Uses VXLAN. |
| `macvlan` | single host | Container gets its own MAC and appears as a physical device on the LAN. |
| `ipvlan` | single host | Like macvlan but shares the parent's MAC; useful where the switch limits MACs per port. |

`bridge` and `overlay` are the two you must understand deeply.

---

## 3. Popular & Useful Commands

### Managing networks

```bash
docker network ls                      # list networks
docker network inspect bridge          # full JSON: subnet, gateway, connected containers
docker network create my-net           # create a user-defined bridge network
docker network rm my-net               # delete a network
docker network prune                   # remove all unused networks

# Connect / disconnect a running container
docker network connect my-net web
docker network disconnect my-net web
```

### Creating a network with an explicit subnet

```bash
docker network create \
  --driver bridge \
  --subnet 172.28.0.0/16 \
  --gateway 172.28.0.1 \
  --ip-range 172.28.5.0/24 \
  my-custom-net
```

- `--subnet` — the address space Docker manages for this network.
- `--gateway` — the bridge's own IP (the default route for containers).
- `--ip-range` — restrict auto-assigned IPs to a sub-slice of the subnet (handy when reserving addresses for static assignment).

### Running containers on a network

```bash
docker run -d --name web --network my-net nginx
docker run -d --name web --network my-net --ip 172.28.0.10 nginx   # static IP
docker run -d -p 8080:80 nginx                                     # publish host:container port
docker run -d --network host nginx                                 # host networking
```

### Inspecting what's actually happening

```bash
docker port web                        # show published port mappings
docker exec web ip addr                # interfaces inside the container
docker exec web ip route               # container routing table
docker exec web cat /etc/resolv.conf   # Docker's embedded DNS (127.0.0.11)

# On the host:
ip addr show docker0                   # the default bridge
brctl show                             # (bridge-utils) show bridges + attached veth ports
bridge link                            # modern replacement for brctl show
iptables -t nat -L -n -v               # see the NAT / masquerade rules Docker installed
```

### Handy diagnostic one-liners

```bash
# See the veth pair endpoints
docker exec web ip link            # note the ifindex, e.g. eth0@if42
ip link | grep 42                  # find the matching host-side veth

# Watch conntrack translations live (needs conntrack-tools)
conntrack -L | grep <container-ip>
```

---

## 4. The Default Bridge (`docker0`)

When the Docker daemon starts, it creates a bridge named `docker0`, typically with subnet `172.17.0.0/16` and gateway `172.17.0.1`.

For **each container**:

1. Docker creates a **veth pair** — `vethXXXX` (host side) and what becomes `eth0` (container side).
2. The container end is moved into the container's **network namespace** and renamed `eth0`.
3. The host end is attached as a **port on `docker0`**.
4. The container gets an IP from the bridge subnet (e.g. `172.17.0.2`), with `docker0` (`172.17.0.1`) as its default gateway.

Visually:

```
         Container A ns              Container B ns
        ┌──────────────┐            ┌──────────────┐
        │ eth0         │            │ eth0         │
        │ 172.17.0.2   │            │ 172.17.0.3   │
        └──────┬───────┘            └──────┬───────┘
               │ veth pair                 │ veth pair
     ──────────┼───────────────────────────┼──────────  host namespace
          vethA│                       vethB│
        ┌──────┴───────────────────────────┴───────┐
        │             docker0  (bridge)             │
        │             172.17.0.1/16                 │
        └────────────────────┬──────────────────────┘
                             │  (iptables NAT / MASQUERADE)
                        ┌────┴─────┐
                        │  eth0    │  host's real NIC
                        │ 192.168.1.50
                        └────┬─────┘
                             │
                          [ LAN / Internet ]
```

### Default bridge vs. user-defined bridge — important difference

- **Default `docker0`**: containers reach each other only by **IP**. No automatic name resolution.
- **User-defined bridge** (`docker network create ...`): Docker runs an **embedded DNS server at `127.0.0.11`** inside each container, so containers resolve each other by **container name**. This is why you should almost always create your own network rather than using the default bridge.

```bash
docker network create app-net
docker run -d --name db  --network app-net postgres
docker run -d --name api --network app-net myapi   # can now reach "db:5432" by name
```

User-defined bridges also give you better isolation: only containers explicitly attached to the same network can talk.

---

## 5. Creating Subnets Inside a Docker Host

You can carve up multiple isolated L2 segments on a single host simply by creating multiple bridge networks, each with its own subnet:

```bash
docker network create --subnet 10.10.0.0/24 frontend-net
docker network create --subnet 10.20.0.0/24 backend-net
```

Each network is a separate bridge with its own `/24`. Containers on `frontend-net` cannot reach `backend-net` at L2 — they're on different bridges. A container that needs to span both (e.g. an API gateway) can be attached to **both**:

```bash
docker network connect frontend-net gateway
docker network connect backend-net  gateway
```

Now `gateway` has two interfaces (one per bridge) and can route/proxy between the tiers, while the DB on `backend-net` stays unreachable from the frontend tier. This is the classic three-tier isolation pattern, done entirely with bridges and subnets on one host.

Behind the scenes each `docker network create` gives you a new `br-<id>` bridge (you'll see them in `ip addr`), and Docker's IPAM (IP Address Management) hands out addresses from that subnet.

---

## 6. How the Bridge Connects to the Host Network

The bridge (`docker0` / `br-xxxx`) is L2-isolated — the container subnet (`172.17.0.0/16`) is **private** and not routable on your LAN or the internet. Two mechanisms bridge that gap:

### (a) Outbound: IP masquerading (source NAT)

Docker installs an iptables rule in the `nat` table's `POSTROUTING` chain, roughly:

```
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

Meaning: *any packet from the container subnet leaving via a non-`docker0` interface (i.e. heading out to the real network) gets its source IP rewritten to the host's IP.* `MASQUERADE` is like `SNAT` but auto-detects the outbound interface's IP — convenient for hosts with dynamic addresses.

This also requires `net.ipv4.ip_forward=1` on the host (Docker enables it), because the host is now **routing** between the bridge and the physical NIC.

### (b) Inbound: port publishing (destination NAT)

When you run `-p 8080:80`, Docker installs a `DNAT` rule in `PREROUTING`:

```
-A DOCKER -p tcp --dport 8080 -j DNAT --to-destination 172.17.0.2:80
```

*Traffic arriving at the host on port 8080 is rewritten to go to the container at `172.17.0.2:80`.* Docker also runs a small userland helper, **`docker-proxy`**, as a fallback for certain cases (e.g. connecting from the host itself to a published port, or where the kernel path doesn't cover it) — though the primary data path is the kernel iptables rules.

---

## 7. The Full Packet Journey: Container → Internet → Back

This is the part people find mysterious. Let's trace a container at `172.17.0.2` making an HTTPS request to a server at `93.184.216.34:443`. The host's LAN IP is `192.168.1.50`.

### Outbound

1. **Inside the container**, the app opens a socket. The container's routing table says: *anything not local → default gateway `172.17.0.1` (docker0)*. So the packet is:
   ```
   src 172.17.0.2:49152  →  dst 93.184.216.34:443
   ```
   It leaves `eth0`, travels the veth pair, and pops out on the host side attached to `docker0`.

2. **Host routing kicks in.** Because `ip_forward=1`, the host treats itself as a router. The destination `93.184.216.34` isn't local, so it consults the host routing table → send via the real NIC `eth0` toward the LAN's default gateway.

3. **POSTROUTING / MASQUERADE fires** just before the packet leaves the physical NIC. The kernel rewrites the **source** address:
   ```
   src 192.168.1.50:52000  →  dst 93.184.216.34:443
   ```
   (Note the source *port* may also be rewritten to keep translations unique — this is PAT/NAPT.)

4. **conntrack records the translation.** The kernel's connection-tracking table stores an entry like:
   ```
   tcp  ESTABLISHED
   original:  172.17.0.2:49152  → 93.184.216.34:443
   reply:     93.184.216.34:443 → 192.168.1.50:52000
   ```
   This is the crucial bit: the mapping from the *masqueraded* tuple back to the *original* container tuple is remembered.

5. Packet goes out to the LAN gateway → ISP → internet → reaches the destination server. As far as the remote server is concerned, it's talking to `192.168.1.50` — it has no idea the container's private `172.17.0.2` exists.

### Return

6. The server replies:
   ```
   src 93.184.216.34:443  →  dst 192.168.1.50:52000
   ```
   It sends this back to the host's public/LAN IP — the only address it knows.

7. The packet arrives at the host NIC. **conntrack looks it up.** It matches the "reply" tuple recorded in step 4, so the kernel knows this belongs to a masqueraded connection and performs the **reverse translation (un-NAT)**:
   ```
   src 93.184.216.34:443  →  dst 172.17.0.2:49152
   ```

8. Now the destination is `172.17.0.2`, which the host routes **into `docker0`**, across the veth pair, and into the container's `eth0`. The app receives the reply on the exact socket it opened.

### Why the container's "internal return address" isn't a problem

The container never actually sends its private address out to the world — masquerade strips it at the border. The **host** holds the memory (in conntrack) of which container owns which translated (IP:port) tuple. When the reply comes back to the host's real IP, conntrack reverses the translation and delivers it to the right container. So the "return address" that matters on the wire is always the host's real IP; the private address only lives on the inside, and conntrack is the bookkeeper that stitches the two together.

```
CONTAINER            DOCKER0 / HOST                     INTERNET
172.17.0.2  ──req──▶  [MASQUERADE: src→192.168.1.50]  ──▶ 93.184.216.34
                       [conntrack: remember mapping]
172.17.0.2  ◀─reply─  [un-NAT: dst→172.17.0.2]        ◀── 93.184.216.34
                       [conntrack: reverse lookup]
```

---

## 8. Overlay Networks (Multi-Host)

Bridges only span a single host. To let containers on **different physical/VM hosts** talk as if on one L2 segment (used in Docker Swarm, and conceptually similar to what CNI plugins do in Kubernetes), Docker uses the **overlay** driver.

### How it works

- Overlay networks use **VXLAN** (Virtual eXtensible LAN), which **encapsulates** L2 Ethernet frames inside UDP packets (default port **4789**) and tunnels them between hosts. This is MAC-in-UDP tunneling.
- Each host has a VXLAN tunnel endpoint (**VTEP**). A container frame is wrapped with a VXLAN header (carrying a **VNI**, the VXLAN Network Identifier that isolates one overlay from another), then an outer UDP/IP header addressed to the destination host's VTEP.
- The receiving host's VTEP strips the outer headers and delivers the original frame to the target container's bridge — so the two containers believe they're on the same LAN despite being on different machines.

```
Host A                                             Host B
┌─────────────┐                                    ┌─────────────┐
│ container   │  original L2 frame                 │ container   │
│ 10.0.0.5    │──┐                              ┌──▶│ 10.0.0.6    │
└─────────────┘  │                              │   └─────────────┘
                 ▼                              │
            [ VTEP: wrap in VXLAN/UDP ]         [ VTEP: unwrap ]
                 │  outer UDP:4789               ▲
                 │  src=HostA-IP dst=HostB-IP    │
                 └──────── physical network ─────┘
```

### Control plane

- Swarm distributes network state (which container IP/MAC lives on which host) via a **gossip protocol**, so each VTEP knows where to tunnel frames for a given destination MAC.
- An overlay also gets an associated bridge on each host and its own subnet, plus optional encryption (IPSec) of the VXLAN traffic with `--opt encrypted`.

```bash
docker network create -d overlay --attachable my-overlay
docker service create --network my-overlay --name web nginx
```

### Ports that must be open between hosts for overlay

| Port | Protocol | Purpose |
|---|---|---|
| 4789 | UDP | VXLAN data plane |
| 7946 | TCP/UDP | Gossip / control plane |
| 2377 | TCP | Swarm cluster management |

---

## 9. Quick Comparison: bridge vs host vs overlay vs macvlan

| Aspect | bridge | host | overlay | macvlan |
|---|---|---|---|---|
| Isolation | good | none | good | good |
| NAT involved | yes | no | yes (for external) | no |
| Cross-host | no | no | **yes** | no (L2 only) |
| Performance | slight NAT overhead | native | encapsulation overhead | near-native |
| Container gets LAN IP | no | uses host IP | no (overlay subnet) | **yes** |
| Typical use | most single-host apps | latency-critical / host-tools | Swarm, multi-node | container as first-class LAN device |

---

## 10. Gotchas & Tips

- **Always prefer user-defined bridges** over the default `docker0` — you get DNS-based service discovery and tighter isolation for free.
- **`docker0` subnet clashes**: if `172.17.0.0/16` overlaps with your corporate VPN/LAN, change it via `/etc/docker/daemon.json` → `"bip": "172.26.0.1/16"` and `"default-address-pools"`.
- **`ip_forward` must be on** for containers to reach the outside — Docker sets it, but hardened hosts or security tooling sometimes reset it.
- **conntrack table exhaustion**: high-connection-count hosts can fill `nf_conntrack_max`; you'll see dropped connections. Tune `net.netfilter.nf_conntrack_max`.
- **`host` networking skips all of this** — no veth, no NAT, no port publishing (the container binds host ports directly). Fast, but you lose isolation and can hit port conflicts.
- **Published ports bypass container firewalls you might expect**: Docker's iptables rules sit in front of the host firewall, so a `-p` publish can be reachable even if `ufw` looks like it should block it. Be deliberate — bind to `127.0.0.1:8080:80` if you only want local access.
- **Inspecting the namespace directly**: `nsenter` or `ip netns` let you drop into a container's net namespace from the host for deep debugging.

---

## 11. One-Screen Summary

- Docker networking = **network namespaces + veth pairs + Linux bridge + iptables + conntrack**.
- A container gets an `eth0` (one end of a veth pair); the other end plugs into a bridge (`docker0` or `br-xxxx`) in the host namespace.
- The container subnet is private; **MASQUERADE (source NAT)** rewrites outbound packets to the host IP, and **conntrack** remembers the mapping so replies are un-NATed back to the right container.
- Inbound access uses **`-p` → DNAT** rules.
- **User-defined bridges** add embedded DNS (name resolution) and isolation.
- Multiple **subnets/bridges** on one host give you tiered isolation; attach a container to several networks to bridge tiers.
- **Overlay networks** use **VXLAN (UDP 4789)** to tunnel L2 frames between hosts so containers across machines share a virtual network.