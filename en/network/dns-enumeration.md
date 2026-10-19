# DNS enumeration & zone transfer

DNS is a map of a target's attack surface: hosts, mail servers, services, and
often internal naming hints. Before touching anything, I enumerate it. And once
in a while a misconfigured server just hands you the entire map with a zone
transfer.

All examples are generic. Nothing here is the solution to a specific challenge.

## 1. The records worth asking for

```
dig target.com A           # IPv4
dig target.com AAAA        # IPv6
dig target.com MX          # mail servers
dig target.com NS          # name servers
dig target.com TXT         # SPF/DKIM, verification strings, sometimes leaks
dig target.com SOA         # zone metadata
dig target.com ANY         # everything the server will give (often filtered now)
```

`NS` and `MX` point you at more infrastructure; `TXT` records routinely leak the
SaaS a company uses (and sometimes more).

## 2. Zone transfer (AXFR)

A zone transfer is how a secondary name server pulls the full zone from the
primary. If the server allows it from anyone, you get **every record** — all
subdomains, internal hosts, the lot:

```
dig axfr target.com @ns1.target.com
```

Try it against *each* name server from the `NS` query — often only one is
misconfigured. A successful AXFR is one of the best recon wins there is; a
refused one just means "do it the slow way".

## 3. Subdomain enumeration (the slow way)

When AXFR is closed, build the subdomain list yourself:

- **Certificate transparency** — every TLS cert is logged publicly. `crt.sh`
  (`https://crt.sh/?q=%25.target.com`) lists names that ever got a cert. Fast,
  passive, no packets to the target.
- **Brute force** — `dnsx`, `gobuster dns`, `dnsrecon` against a wordlist.
- **Passive sources** — amass, subfinder aggregate public data.

(This is exactly the kind of recon a tool like
[Scryn](https://github.com/white-exee/Scryn) automates with crt.sh.)

## 4. Reverse DNS

Going from IP back to name can reveal naming schemes across a range:

```
dig -x 203.0.113.10
```

Sweeping a netblock's PTR records often maps out a whole environment.

## 5. Defending

- **Restrict AXFR** to known secondary servers only — never allow it from the
  world.
- Keep internal hosts in a **split-horizon** zone so they aren't in the public
  one.
- Treat `TXT` records as public: don't put anything sensitive there.

---

Built by **White** · [github.com/white-exee](https://github.com/white-exee)
