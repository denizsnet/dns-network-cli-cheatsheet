# DNS & Network CLI Cheatsheet

Copy-paste commands for everyday **DNS, email and network troubleshooting** on Linux, macOS and Windows: `dig`, `nslookup`, `Resolve-DnsName`, `curl`, `ping`, `traceroute`, `nc` and `whois`.

Each section links to a free browser tool on **[IP DNS Hub](https://ipdnshub.com/)** that runs the same check without a terminal.

## Contents

- [DNS records](#dns-records)
- [Query a specific resolver / propagation](#query-a-specific-resolver--propagation)
- [Reverse DNS (PTR)](#reverse-dns-ptr)
- [Email: MX, SPF, DKIM, DMARC](#email-mx-spf-dkim-dmarc)
- [DNSSEC](#dnssec)
- [Local DNS cache](#local-dns-cache)
- [Reachability: ping, traceroute, ports](#reachability-ping-traceroute-ports)
- [HTTP checks with curl](#http-checks-with-curl)
- [WHOIS, RDAP and ASN](#whois-rdap-and-asn)
- [Blacklists (DNSBL)](#blacklists-dnsbl)
- [Censorship checks (DNS level)](#censorship-checks-dns-level)

---

## DNS records

```bash
dig example.com A +short          # IPv4
dig example.com AAAA +short       # IPv6
dig example.com CNAME +short
dig example.com NS +short         # nameservers
dig example.com SOA +short
dig example.com TXT +short
dig example.com CAA +short        # which CAs may issue certificates
dig example.com A                 # full answer incl. TTL
```

Windows:

```powershell
nslookup -type=a example.com
nslookup -type=txt example.com
Resolve-DnsName example.com -Type AAAA
Resolve-DnsName example.com -Type NS
```

`+short` prints only the answer. Without it, `dig` also shows the **TTL** — how long resolvers may cache the record.

Browser tools: [DNS Record Lookup](https://ipdnshub.com/dnsrecord) · [DNS Report](https://ipdnshub.com/dnsreport)

## Query a specific resolver / propagation

```bash
dig @1.1.1.1 example.com A +short     # Cloudflare
dig @8.8.8.8 example.com A +short     # Google
dig @9.9.9.9 example.com A +short     # Quad9

# Compare several resolvers
for r in 1.1.1.1 8.8.8.8 9.9.9.9 208.67.222.222; do
  echo "$r -> $(dig @$r example.com A +short | tr '\n' ' ')"
done

# Ask the domain's authoritative nameserver directly
dig @$(dig example.com NS +short | head -1) example.com A +short
```

Windows: `nslookup example.com 1.1.1.1`

If the authoritative nameserver already returns the new value, the change is live and resolvers will pick it up when the old TTL expires.

Browser tool: [DNS Propagation Checker](https://ipdnshub.com/propagation) · Guides: [How to check DNS propagation](https://ipdnshub.com/resources/how-to-check-dns-propagation), [Why DNS changes are not showing](https://ipdnshub.com/resources/why-dns-changes-are-not-showing) · Resolver list: [public-dns-servers](https://github.com/denizsnet/public-dns-servers)

## Reverse DNS (PTR)

```bash
dig -x 8.8.8.8 +short
nslookup 8.8.8.8                       # Windows
Resolve-DnsName 8.8.8.8                # PowerShell
```

Forward-confirmed reverse DNS (important for mail servers): the PTR name must resolve back to the same IP.

```bash
ip=203.0.113.25; name=$(dig -x $ip +short); dig $name A +short
```

Browser tool: [Reverse DNS Lookup](https://ipdnshub.com/reversedns) · Guide: [Reverse DNS and PTR records explained](https://ipdnshub.com/resources/reverse-dns-ptr-records-explained)

## Email: MX, SPF, DKIM, DMARC

```bash
dig example.com MX +short                          # mail servers (lower number = preferred)
dig example.com TXT +short | grep spf1             # SPF
dig _dmarc.example.com TXT +short                  # DMARC
dig selector1._domainkey.example.com TXT +short    # DKIM (selector name depends on the provider)
```

Windows:

```powershell
nslookup -type=mx example.com
Resolve-DnsName _dmarc.example.com -Type TXT
```

Rules of thumb: only **one** SPF record per domain, at most **10 DNS lookups** in SPF, and MX records must point to hostnames (not IPs or CNAMEs).

Guide: [SPF, DKIM and DMARC explained](https://ipdnshub.com/resources/spf-dkim-dmarc-explained) · Browser tools: [Free Email Test](https://ipdnshub.com/freeemail), [Reverse MX Lookup](https://ipdnshub.com/reversemx)

## DNSSEC

```bash
dig example.com DNSKEY +dnssec +multi     # keys and signatures
dig example.com DS +short                 # DS record at the parent zone
dig @1.1.1.1 example.com A +dnssec        # look for the "ad" flag = validated
delv example.com                          # validation result (BIND tools)
```

A signed zone without a DS record at the parent is treated as unsigned. A DS record that does not match the zone's keys makes the domain fail on validating resolvers (`SERVFAIL`).

Browser tool: [DNSSEC Checker](https://ipdnshub.com/dnssec) · Guide: [DNSSEC explained](https://ipdnshub.com/resources/dnssec-explained-how-to-check-if-a-domain-is-signed)

## Local DNS cache

```bash
# Windows
ipconfig /flushdns
ipconfig /displaydns

# macOS
sudo dscacheutil -flushcache; sudo killall -HUP mDNSResponder

# Linux (systemd-resolved)
resolvectl flush-caches
resolvectl query example.com
```

Chrome: open `chrome://net-internals/#dns` and clear the host cache.

Hosts file: `C:\Windows\System32\drivers\etc\hosts` (Windows), `/etc/hosts` (macOS, Linux). An entry there overrides DNS.

## Reachability: ping, traceroute, ports

```bash
ping -c 4 example.com              # Linux / macOS
ping example.com                   # Windows (4 packets)
traceroute example.com             # Linux / macOS
tracert -d example.com             # Windows, no reverse lookups (faster)
mtr -rw example.com                # combined ping + traceroute report

nc -vz example.com 443             # is a TCP port open?
```

PowerShell:

```powershell
Test-Connection example.com -Count 4
Test-NetConnection example.com -Port 443
Test-NetConnection example.com -TraceRoute
```

Many servers block ICMP: a failing `ping` does not mean a website is down. `* * *` lines in the middle of a traceroute are usually routers that do not answer probes.

Browser tools: [Ping](https://ipdnshub.com/ping), [Traceroute](https://ipdnshub.com/traceroute), [Port Scanner](https://ipdnshub.com/portscan) (scan only hosts you own)

## HTTP checks with curl

```bash
curl -sI https://example.com                        # response headers
curl -sIL https://example.com                       # follow redirects
curl -s -o /dev/null -w "%{http_code} %{time_total}s\n" https://example.com   # status + time
curl -s -o /dev/null -w "dns %{time_namelookup}  connect %{time_connect}  tls %{time_appconnect}  ttfb %{time_starttransfer}\n" https://example.com
curl -sI --resolve example.com:443:203.0.113.10 https://example.com   # test a new server before changing DNS
```

The `--resolve` trick sends the request to a specific IP while keeping the correct hostname — useful when migrating a site.

Browser tools: [HTTP Headers](https://ipdnshub.com/httpheaders), [Is My Site Down](https://ipdnshub.com/ismysitedown)

## WHOIS, RDAP and ASN

```bash
whois example.com                                  # registration data
curl -s https://rdap.org/domain/example.com        # RDAP (structured JSON)
whois 8.8.8.8                                      # IP block owner (RIR data)

# Origin ASN of an IPv4 address (reverse the octets)
dig 8.8.8.8.origin.asn.cymru.com TXT +short
dig AS15169.asn.cymru.com TXT +short               # ASN name

# Abuse contact of an IP
whois 203.0.113.25 | grep -i abuse
```

Browser tools: [WHOIS Lookup](https://ipdnshub.com/whois), [ASN Lookup](https://ipdnshub.com/asnlookup), [IP Location](https://ipdnshub.com/iplocation), [Abuse Contact Lookup](https://ipdnshub.com/abuselookup) · Guide: [What is an ASN?](https://ipdnshub.com/resources/what-is-an-asn-and-how-to-find-who-owns-an-ip-address)

## Blacklists (DNSBL)

DNS blacklists are queried with the IPv4 address reversed:

```bash
ip=203.0.113.25
rev=$(echo $ip | awk -F. '{print $4"."$3"."$2"."$1}')
dig +short $rev.zen.spamhaus.org            # 127.0.0.x = listed, empty = not listed
dig +short $rev.b.barracudacentral.org
```

Spamhaus refuses queries sent through large public resolvers; run the check with your own resolver.

Browser tool: [Spam Database Lookup](https://ipdnshub.com/spamdblookup) · Guide: [How to check if your IP is blacklisted](https://ipdnshub.com/resources/how-to-check-if-your-ip-is-blacklisted)

## Censorship checks (DNS level)

Compare a reference resolver with a resolver inside the country:

```bash
# China (114DNS)
dig @114.114.114.114 example.com A +short
# Iran (Shecan)
dig @178.22.122.100 example.com A +short
# Reference
dig @8.8.8.8 example.com A +short
```

A private address in the answer (for example `10.10.34.36`), no answer, or an address that belongs to an unrelated network (compare the origin ASNs above) indicates DNS-level filtering. Different addresses within the same CDN are normal.

Browser tools: [China Firewall Test](https://ipdnshub.com/chinesefirewall), [Iran Firewall Test](https://ipdnshub.com/iranfirewall) · Guides: [China](https://ipdnshub.com/resources/how-to-check-if-a-website-is-blocked-in-china), [Iran](https://ipdnshub.com/resources/how-to-check-if-a-website-is-blocked-in-iran)

---

## Installing the tools

| OS | dig / delv | whois | mtr | nc |
|---|---|---|---|---|
| Debian / Ubuntu | `sudo apt install dnsutils` | `sudo apt install whois` | `sudo apt install mtr-tiny` | `sudo apt install netcat-openbsd` |
| Fedora / RHEL | `sudo dnf install bind-utils` | `sudo dnf install whois` | `sudo dnf install mtr` | `sudo dnf install nmap-ncat` |
| macOS | built in | built in | `brew install mtr` | built in |
| Windows | use `nslookup` / `Resolve-DnsName` | — | — | use `Test-NetConnection` |

Windows 10 and 11 also include `curl.exe`.

## Contributing

Corrections and useful one-liners are welcome — open an issue or a pull request.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) — reuse freely with attribution and a link to this repository.
