# Port scanning & service enumeration

Scanning tells you what's listening; enumeration tells you what it *is* and
where it's weak. The goal isn't a wall of open ports — it's a short list of
services with versions you can act on. Work from fast-and-wide to slow-and-deep.

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. Find the hosts first

Before ports, know what's alive on the subnet:

```
nmap -sn 10.10.10.0/24          # ping sweep, no port scan
```

On a LAN where ICMP is filtered, ARP is more reliable:

```
arp-scan --localnet
```

## 2. Sweep all ports fast, then go deep

Two-stage is faster than one slow full scan. First a full TCP sweep, then a
detailed scan of only the ports that answered:

```
nmap -p- --min-rate 2000 -T4 10.10.10.5           # all 65535 TCP ports
nmap -p 22,80,445 -sC -sV 10.10.10.5              # scripts + versions
```

- `-sS` SYN scan (default as root, half-open, quiet-ish)
- `-sV` service/version detection
- `-sC` default NSE scripts
- `-O` OS fingerprint
- `-sU` UDP (slow — scan a short top-ports list, not `-p-`)

Don't forget UDP: DNS, SNMP, TFTP, NTP all hide there.

```
nmap -sU --top-ports 50 10.10.10.5
```

## 3. Read versions, map to weaknesses

Every version string is a lead. `-sV` output like `OpenSSH 7.2`, `vsftpd 2.3.4`,
`Apache 2.4.49` is a prompt to check for known issues (`searchsploit <product
version>`). Banner-grab manually when NSE is vague:

```
nc -nv 10.10.10.5 22
curl -sI http://10.10.10.5
```

## 4. Enumerate per service

The scan is the start. Each open service has its own enumeration:

- **80/443 HTTP** — directories (`feroxbuster`/`ffuf`), vhosts, tech stack,
  default pages, robots.txt.
- **445 SMB** — shares and null sessions (`smbclient -L //host -N`,
  `enum4linux-ng`).
- **21 FTP** — anonymous login, writable dirs.
- **161 UDP SNMP** — community strings (`snmpwalk -v2c -c public`).
- **53 DNS** — zone transfer, subdomain brute (see the DNS writeup).
- **3306/5432 DB** — default creds, version.

## 5. Stay efficient

- Scan the full TCP range once, early — a service on port 54321 is easy to miss.
- Save machine-readable output (`-oA name`) so you can grep ports later.
- Rate-limit on fragile targets (`--max-rate`) — hammering can crash old
  services or trip detection.

## 6. Defending

- Expose only the ports a service genuinely needs; firewall the rest
  (default-deny inbound).
- Don't leak versions — tune banners and server tokens where you can.
- Patch the versions that *do* show; an accurate banner on a current build is
  fine, an outdated one is an invitation.
- Monitor for sweeps: a single host touching many ports fast is a signature.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
