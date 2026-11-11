# connect: a routed network with two subnets

A gateway VM joins two private subnets and the internet. Everything is built from code with `vagrant up`.

```
                 Internet
                     |
                   (NAT)
                     |
              [enp0s3] 10.0.2.15
                 GATEWAY
   routing, NAT, DHCP, DNS, firewall
   [enp0s8] 192.168.50.1     [enp0s9] 192.168.60.1
        |                          |
   subnet A (gateway-lan)     subnet B (gateway-lan-b)
        |                          |
     client1                    client2
  192.168.50.50 (DHCP)       192.168.60.10 (static)
```

## How to run

```bash
vagrant destroy -f
time vagrant up
```

Machines: `gateway`, `client1` (subnet A), `client2` (subnet B). All use the box `debian13-local`.

## Files

| File | Job |
|---|---|
| `Vagrantfile` | Creates the three VMs, their network cards and the cables between them |
| `provision/provision.sh` | Runs `linux.sh` then `login.sh` on every VM, before the network scripts |
| `provision/linux.sh`, `provision/login.sh` | Carried forward from add-vm (they call the scripts in `provision/providers/`) |
| `provision/gateway.sh` | Addresses, IP forwarding, NAT and firewall rules |
| `provision/dhcp.sh` | DHCP server and the static reservation by MAC |
| `provision/dns.sh` | Internal DNS resolver (dnsmasq) |
| `provision/client.sh` | Client A: DHCP, default route via the gateway, DNS settings |
| `provision/client-b.sh` | Client B: static address, default route via the gateway, DNS settings |
| `systemd/` | Monitoring service installed by `linux.sh` (from add-vm) |
| `keys/` | Public keys installed by `login.sh` (from add-vm) |
| `diagram.png` | Network diagram with the addressing plan |
| `troubleshooting.md` | Problems hit while building, and how they were diagnosed |

Client A is rebooted once at the end of its provisioning (`reboot: true` in the Vagrantfile). dhcpcd only reads its configuration at start, so the reboot is what makes it ignore the NAT card's DNS servers.

## Addressing plan

| | Subnet A | Subnet B |
|---|---|---|
| CIDR | 192.168.50.0/24 | 192.168.60.0/24 |
| Network address | 192.168.50.0 | 192.168.60.0 |
| Broadcast address | 192.168.50.255 | 192.168.60.255 |
| Usable hosts | 254 | 254 |
| Gateway | 192.168.50.1 | 192.168.60.1 |
| Static range | .2 to .99 | .2 to .99 |
| DHCP pool | .100 to .200 | not served |
| Spare | .201 to .254 | .100 to .254 |
| Reservation | 192.168.50.50 (client1, by MAC) | none |
| Static host | none | 192.168.60.10 (client2) |

### Why /24

I chose /24 for both subnets because this is a small lab gateway built for
learning and practicing networking: handling servers, connections and traffic.
A /24 gives 254 usable addresses per subnet, far more than the one or two
clients I need, so there is room to add machines later without redesigning.
It is also easy to read and debug, because only the last number changes, and
the two ranges (192.168.50.0/24 and 192.168.60.0/24) are clearly separate
networks that cannot overlap.

### Reserved addresses

| Range | Purpose |
|---|---|
| `.1` | Gateway, the same last octet in both subnets so it is easy to remember |
| `.2` to `.99` | Static addresses for servers and special hosts |
| `.100` to `.200` | Dynamic DHCP pool (subnet A only) |
| `.50` (subnet A) | Reservation by MAC, outside the pool so it can never collide with a dynamic client |

Static and dynamic addresses never share a range.

## Routing and NAT

- `net.ipv4.ip_forward=1` (file in `/etc/sysctl.d/`) lets the gateway pass packets between its cards.
- The gateway is directly connected to both subnets, so it needs no extra routes between them.
- Each client has **one default route, pointing at the gateway**. Traffic to the other subnet and to the internet both go there. The Vagrant NAT card on each client keeps only its address, which Vagrant SSH needs, not its default route.
- NAT: one `masquerade` rule on `enp0s3`. Traffic between the subnets is never translated, so each client sees the other's real address.

## DHCP

- Server: `isc-dhcp-server`, listening on `enp0s8` only (subnet A).
- Pool `192.168.50.100` to `192.168.50.200`. Router and DNS options point to `192.168.50.1`.
- **Static reservation:** client1's NIC 2 has a fixed MAC set in the Vagrantfile (`08:00:27:aa:00:01`), and `dhcpd.conf` maps it to `192.168.50.50`. The MAC is fixed because VirtualBox would otherwise generate a new one on every rebuild and the reservation would stop matching.
- Client B uses a static address and does not use DHCP.
- **Only one DHCP server answers.** The network cards use VirtualBox internal networks (`intnet`), configured with `vb.customize`, not Vagrant `private_network`. Vagrant does not configure these interfaces itself, and VirtualBox runs no DHCP server on internal networks, so only `isc-dhcp-server` answers.

## DNS

- Resolver: `dnsmasq`, DNS only (DHCP disabled), listening on `192.168.50.1` and `192.168.60.1`.
- `no-hosts` is set, so `/etc/hosts` is not used. Records live in `/etc/dnsmasq.d/lab.conf`:

| Name | Address |
|---|---|
| gateway | 192.168.50.1 |
| client-a | 192.168.50.50 |
| client-b | 192.168.60.10 |

- Unknown names are forwarded to 8.8.8.8 and 1.1.1.1.
- Client A learns its DNS server from DHCP. Client B has `nameserver 192.168.60.1` written in `/etc/resolv.conf`.
- On client A, `/etc/dhcpcd.conf` has `nohook resolv.conf` for the NAT card, so VirtualBox's DNS does not end up in `resolv.conf` and cannot answer "not known" for internal names.

Test with the resolver directly, not only with ping:

```bash
dig client-b @192.168.50.1 +short     # 192.168.60.10
ping -c 3 client-b
```

## Firewall policy

nftables, in the `forward` chain of `table inet filter`, with `policy drop` (default deny).

| Traffic | Result |
|---|---|
| Replies to connections already allowed | allowed |
| Subnet A or B to the internet (out `enp0s3`) | allowed |
| ICMP echo request/reply between A and B | allowed |
| SSH (tcp 22) from A to B | allowed |
| SSH from B to A | blocked |
| Any other port between A and B (for example 80) | blocked |
| Anything else crossing the gateway | blocked |

Only traffic crossing the gateway is filtered. Traffic to the gateway itself (DHCP, DNS) does not pass through the `forward` chain.

### Demonstration

```bash
# allowed
vagrant ssh client1 -c "ping -c 3 client-b"
vagrant ssh client1 -c "nc -zv -w 3 192.168.60.10 22"
# blocked
vagrant ssh client1 -c "nc -zv -w 3 192.168.60.10 80"
vagrant ssh client2 -c "nc -zv -w 3 192.168.50.50 22"
```

## Verify the whole network

```bash
vagrant ssh gateway -c "ip -br a; sudo nft list ruleset; systemctl is-active isc-dhcp-server dnsmasq nftables"
vagrant ssh client1 -c "ip -br a show enp0s8; ip route"
vagrant ssh client2 -c "ip -br a show enp0s8; ip route"
```

## Rebuild time

`vagrant destroy -f` followed by `time vagrant up` takes about **7 minutes 49 seconds** (`7:49.28` total) for all three VMs, including the client A reboot.