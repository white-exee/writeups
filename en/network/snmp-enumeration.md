# SNMP enumeration

SNMP is used to manage network gear and servers, and it loves to talk. On v1/v2c
the only "authentication" is a **community string** — and it's very often left on
the default. Find it and the device will dump interfaces, processes, software,
routes, sometimes even users.

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. The basics

- UDP **161** (queries), 162 (traps).
- **v1 / v2c**: community string in cleartext. `public` = read-only by default,
  `private` = read/write.
- **v3**: real auth + encryption (so these tricks don't apply).

Because it's UDP, a quick TCP scan misses it — scan UDP 161 specifically.

## 2. Finding the community string

Default strings are everywhere. Brute a short list fast with `onesixtyone`:

```
onesixtyone -c community.txt 192.168.1.10
# or nmap
nmap -sU -p161 --script snmp-brute 192.168.1.10
```

Common ones to try first: `public`, `private`, `manager`, `community`, the
vendor name.

## 3. Walking the tree

With a valid community string, `snmpwalk` dumps the whole MIB:

```
snmpwalk -v2c -c public 192.168.1.10
snmpbulkwalk -v2c -c public 192.168.1.10          # faster
snmp-check 192.168.1.10 -c public                 # nicely formatted summary
```

Useful OIDs when you don't want everything:

```
1.3.6.1.2.1.1.5     # hostname
1.3.6.1.2.1.25.4.2.1.2   # running processes
1.3.6.1.2.1.25.6.3.1.2   # installed software
1.3.6.1.2.1.2.2.1.2      # network interfaces
1.3.6.1.4.1.77.1.2.25    # Windows users (LanMan)
```

What comes out: hostnames, full interface and routing tables, ARP cache, process
list, installed packages, and on Windows the user accounts — a complete picture
of the host without touching anything else.

## 4. Write access

If a `private`-style string allows writes (`snmpset`), you can change device
config — reroute, disable interfaces, etc. Rare but very powerful on network
gear.

## 5. Defending

- **Disable SNMP** if you don't use it. If you do, **use v3** with auth +
  encryption.
- Change default community strings to something non-guessable, and make them
  **read-only** unless write is truly needed.
- **Firewall UDP 161** to the monitoring hosts only.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
