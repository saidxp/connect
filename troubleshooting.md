# Troubleshooting log: connect

Problems hit while building the two-subnet network (gateway, client A, client B).
Each entry follows the same shape: **symptom, what I thought, what it really was, how I proved it, fix.**

Method used every time: work up the stack in order: **link, address, route, DNS, firewall**.

## Topology reminder

| Machine | Card | Network | Address |
|---|---|---|---|
| gateway | enp0s3 | NAT (internet) | 10.0.2.15 |
| gateway | enp0s8 | subnet A (`gateway-lan`) | 192.168.50.1 |
| gateway | enp0s9 | subnet B (`gateway-lan-b`) | 192.168.60.1 |
| client1 | enp0s8 | subnet A | 192.168.50.50 (DHCP reservation) |
| client2 | enp0s8 | subnet B | 192.168.60.10 (static) |

---

## 1. `isc-dhcp-server` failed during `apt-get install`

**Symptom**
`vagrant up` stopped with `Job for isc-dhcp-server.service failed` and `dhcpd ... exiting.` The log was cut off and the real error line was not visible.

**What I thought**
The installation was broken, or my `dhcpd.conf` had an error.

**What it really was**
Not an installation problem. The package starts the service right after install, using the default config (empty `INTERFACESv4`, no matching subnet), so dhcpd exits. `apt-get` then returns a non-zero code and `set -e` kills the script before my own config is written.

**How I proved it**
- `sudo dhcpd -t -cf /etc/dhcp/dhcpd.conf` printed only the banner, no errors: the config file was valid.
- After provisioning, `systemctl status` showed `dhcpd -4 ... enp0s8` running and `ip -br a` showed `enp0s8` with `192.168.50.1/24`.

**Fix**
Block auto-start during install, write the config, test it, then start:

```bash
echo -e '#!/bin/sh\nexit 101' > /usr/sbin/policy-rc.d
chmod +x /usr/sbin/policy-rc.d
apt-get install -y isc-dhcp-server
rm -f /usr/sbin/policy-rc.d
# ... write /etc/default/isc-dhcp-server and /etc/dhcp/dhcpd.conf ...
dhcpd -t -cf /etc/dhcp/dhcpd.conf
systemctl enable isc-dhcp-server
systemctl restart isc-dhcp-server
```

**Lesson**
A `-t` syntax check passing does not prove the interface is ready. dhcpd also needs an interface with an IP inside the declared subnet.

---

## 2. Client got `169.254.x.x` instead of an address from the pool

**Symptom**
Client `enp0s8` showed `169.254.39.165/16`.

**What I thought**
The DHCP server was misconfigured or not answering.

**What it really was**
The server was fine. `169.254.x.x` is the fallback address a client gives itself when nobody answers. In the Vagrantfile, the client used `--intnet1` instead of `--intnet2`, so its NIC 2 had no network name and landed on VirtualBox's default `intnet`. Gateway and client were on different virtual cables.

**How I proved it**
Compared the two blocks side by side: gateway `--intnet2 gateway-lan` vs client `--intnet1 gateway-lan`. `--intnetN` must use the same number as `--nicN`.

**Fix**
```ruby
vb.customize ["modifyvm", :id, "--nic2", "intnet"]
vb.customize ["modifyvm", :id, "--intnet2", "gateway-lan"]
```
Then `vagrant halt client1 && vagrant up client1`. The client received an address from the pool.

**Lesson**
Machines on the same subnet must share the same internal network name, and the NIC number must match in both lines.

---

## 3. Gateway `enp0s9` was `DOWN` with no IP

**Symptom**
`ip -br a` on the gateway showed `enp0s9  DOWN`.

**What I thought**
The third card was not created, or the address was wrong.

**What it really was**
The card existed (the Vagrantfile worked). `gateway.sh` wrote the `enp0s9` block into `/etc/network/interfaces`, but only ran `ifup` for `enp0s8`. Writing the file does not change the running network.

**How I proved it**
`grep -A3 enp0s9 /etc/network/interfaces` showed the block, while `ip -br a` showed no address.

**Fix**
Apply both LAN cards inside the script, not by hand:

```bash
for nic in enp0s8 enp0s9; do
    ifdown "$nic" 2>/dev/null || true
    ip addr flush dev "$nic"
    ifup "$nic"
done
```

**Lesson**
A config file describes the intended state. Something must apply it.

---

## 4. Ping from client A to client B: 100% packet loss

**Symptom**
`ping 192.168.60.10` from client A: `3 packets transmitted, 0 received`.

**What I thought**
IP forwarding was off on the gateway, or client B could not answer.

**What it really was**
Client A had no route to subnet B. Its only exit for unknown destinations was the NAT card's default route (`via 10.0.2.2`), so packets never reached the gateway.

**How I proved it**
```bash
ip route get 192.168.60.10
# 192.168.60.10 via 10.0.2.2 dev enp0s3
```
Also checked: `cat /proc/sys/net/ipv4/ip_forward` returned `1`, and client B's table already had a route back to subnet A. That ruled out the gateway and client B.

**Fix (first, by hand, to prove the idea)**
```bash
sudo ip route add 192.168.60.0/24 via 192.168.50.1 dev enp0s8
```
Result: 3 replies with `ttl=63`, proof that the packet crossed one router. The permanent fix is problem 5.

**Lesson**
`ip route get <destination>` shows exactly which way a packet will leave. Narrowing down beats guessing.

---

## 5. Per-destination routes were the wrong design

**Symptom**
`ping -I enp0s8 8.8.8.8` from client A: `Destination Host Unreachable`. Fixing it meant adding one route per destination.

**What I thought**
I needed a new hand-written route for every place I wanted to reach.

**What it really was**
Each client had two exits: the Vagrant NAT card (`enp0s3`) and the LAN card. The NAT card's default route won, so traffic bypassed the gateway. The correct design is one default route per client pointing at the gateway; the gateway knows all networks.

**How I proved it**
After switching the default route, a plain `ping 8.8.8.8` and `ping 192.168.60.10` both worked with no specific routes.

**Fix**
Remove the NAT default route and use the gateway:

```bash
# client A
ip route del default via 10.0.2.2 dev enp0s3 2>/dev/null || true
ip route replace default via 192.168.50.1 dev enp0s8
# client B
ip route del default via 10.0.2.2 dev enp0s3 2>/dev/null || true
ip route replace default via 192.168.60.1 dev enp0s8
```
Also set in the interfaces files (`post-up ip route del default via 10.0.2.2 ...` on `enp0s3`, `gateway` on client B). Vagrant SSH kept working because it uses a forwarded port on the NAT card, not a default route.

**Lesson**
Clients should know one rule: "not my subnet, send to my gateway".

---

## 6. Client scripts wrote the config but never applied it

**Symptom**
`ip route` on client A did not show the route I had put in `client.sh`.

**What I thought**
The `up ip route add ...` line was wrong.

**What it really was**
The `up` hook only runs when the card starts. `vagrant provision` never restarted the card, so the line never ran.

**How I proved it**
Restarting the card made the route appear.

**Fix**
End each client script with:

```bash
ifdown enp0s8 2>/dev/null || true
ifup enp0s8 || true
```

**Lesson**
Same as problem 3: write the config, then apply it.

---

## 7. `nftables.service` failed to start after adding the firewall

**Symptom**
`vagrant up` stopped with `Job for nftables.service failed because the control process exited with error code`. DHCP and DNS were never installed on that run.

**What I thought**
A bug in the nftables service or in the `icmp type` syntax.

**What it really was**
A missing closing brace. `table inet filter` was written inside `table ip nat`, and nftables does not allow nested tables.

**How I proved it**
Reading the braces: the `chain postrouting` was closed, but `table ip nat` was not, and an extra `}` at the end hid the mistake.

**Fix**
Close each table completely before opening the next, and check the file before starting the service:

```bash
nft -c -f /etc/nftables.conf
systemctl enable nftables
systemctl restart nftables
```

**Lesson**
`nft -c -f` prints the exact line of the error. Run it before the service so the failure is clear.

---

## 8. First `ping client-b` failed: "Name or service not known"

**Symptom**
`dig client-b @192.168.50.1 +short` returned `192.168.60.10`, but the first `ping client-b` failed. The second try worked.

**What I thought**
The DNS server was down or slow.

**What it really was**
`dig @server` talks to the server directly, but `ping` uses `/etc/resolv.conf`. Client A's file listed three servers: my gateway (`192.168.50.1`), plus VirtualBox's DNS (`10.0.2.3` and an IPv6 one) added by the NAT card. VirtualBox's DNS does not know `client-b`, so a fallback to it returns "not known". I believe this explains the one-time failure; I did not reproduce it on demand.

**How I proved the fix**
After the change, `resolv.conf` listed only `nameserver 192.168.50.1`.

**Fix**
Tell dhcpcd not to let the NAT card write DNS servers:

```bash
grep -q "nohook resolv.conf" /etc/dhcpcd.conf || cat >> /etc/dhcpcd.conf <<'EOF'

interface enp0s3
nohook resolv.conf
EOF
```
`dhcpcd -n` (HUP) did not apply it, because a separate dhcpcd process serves the NAT card and only reads the file at start. `vagrant reload client1` did. Client B writes its own `resolv.conf` with only `192.168.60.1`, so it needed no change.

**Lesson**
Test the resolver with `dig @server`, but also with `ping` by name: they follow different paths.

---

## Minor notes

| Note | Detail |
|---|---|
| `sysctl: command not found` via `vagrant ssh -c` | Not an error. The non-login shell PATH lacks `/usr/sbin`. Use `cat /proc/sys/net/ipv4/ip_forward` instead. |
| `vagrant ssh client1`: "VM must be running" | The VM was powered off. `vagrant up client1`. |
| Same IPv6 link-local on all VMs | All three share one base box (same machine ID). Harmless for this project. |
| DHCP reservation needs a fixed MAC | Without `--macaddress2 080027AA0001` in the Vagrantfile, a rebuild would give a new MAC and break the reservation. |

## Final verification

| Check | Command | Result |
|---|---|---|
| Subnets routed both ways | `ping` A to B and B to A | works, `ttl=63` |
| NAT | `ping 8.8.8.8` from both clients | works |
| DHCP reservation | `ip -br a show enp0s8` on client A | `192.168.50.50` |
| DNS | `dig client-b @192.168.50.1 +short` | `192.168.60.10` |
| Allowed path | `nc -zv -w 3 192.168.60.10 22` from A | open |
| Blocked path | `nc -zv -w 3 192.168.60.10 80` from A | timed out |
| Blocked path (wrong direction) | `nc -zv -w 3 192.168.50.50 22` from B | timed out |
