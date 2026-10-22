>[!danger] AI formatted this guide

>[!hint] Redacted :)


> [!tldr] Full DNS stack for the ZaySec homelab: **Pi-hole** (ad/tracker sinkhole) → **Unbound** (recursive resolver + DNSSEC) → **Dnsmasq** (DHCP + local `lan.internal` names), all on OPNsense, with a transparent NAT redirect that forces every client through Pi-hole. No DNS leaves the network to a third-party resolver.

---

# Overview

### Query flow

```
Client (192.168.1.x)
  └─► Pi-hole  192.168.1.101 : 53      ← blocks ads/trackers
        └─► Unbound  192.168.1.1 : 53  ← recursive resolution + DNSSEC
              ├─ lan.internal? ──► Dnsmasq  127.0.0.1 : 53053   (local hostnames)
              └─ everything else ──► root / authoritative servers
```

### Roles

- **Pi-hole** — the only DNS server clients are told to use. Filtering layer.
- **Unbound** — does the real work: recursive lookups from the root servers + DNSSEC validation.
- **Dnsmasq** — DHCP server + resolves local hostnames (`*.lan.internal`).

> [!info] Why recursive instead of DoH/DoT to a public resolver Unbound resolves **recursively** from the root servers, so no single company (Google/Cloudflare/NextDNS) ever sees the full query history. That is _more_ private than encrypted DNS to a third party. The only plaintext hops (`client → Pi-hole`, `Pi-hole → Unbound`) are on owned hardware inside the LAN, where encryption would only protect against an attacker already inside the network. DNSSEC protects the internet-facing hop from tampering.

---

# Addresses & Ports

> [!info]- Reference
> 
> |Service|Host|Port|
> |---|---|---|
> |OPNsense (gateway)|`192.168.1.1`|—|
> |Unbound (resolver)|`192.168.1.1`|`53`|
> |Dnsmasq (DHCP + local DNS)|`192.168.1.1`|`53053`|
> |Pi-hole (sinkhole)|`192.168.1.101`|`53`|
> |LAN subnet|`192.168.1.0/24`|—|
> |Local domain|`lan.internal`|—|
> 
> **IP plan:** `.1–.4` infrastructure (router, Proxmox) · `.5–.99` DHCP pool · `.100+` services (Pi-hole `.101`, etc.)

---

# 1. DHCP (Dnsmasq)

Only **Dnsmasq** runs DHCP — Kea and ISC must stay disabled or they grab the same ports.

> [!info]- General settings (Services → Dnsmasq DNS & DHCP → General)
> 
> - **Listen Port:** `53053` (intentional — lets Dnsmasq run alongside Unbound on `:53`)
> - **DNS Query Forwarding → Do not forward to system defined DNS servers:** `✓` (Dnsmasq only handles local names; stops local lookups leaking out)
> - **DHCP FQDN:** `✓`
> - **DHCP default domain:** `internal`
> - **DHCP local domain:** `✓`

> [!info]- DHCP range (Services → Dnsmasq DNS & DHCP → DHCP ranges)
> 
> - **Interface:** `LAN`
> - **Start address:** `192.168.1.5`
> - **End address:** `192.168.1.99`

> [!info]- Push Pi-hole as the DNS server to all LAN clients (Services → Dnsmasq DNS & DHCP → DHCP options) The range does **not** set DNS. By default Dnsmasq hands out option-6 = the OPNsense IP, so this DHCP option overrides it network-wide:
> 
> - **Interface:** `LAN` _(auto-creates the interface tag → applies to every LAN client)_
> - **Type:** `Set`
> - **Option:** `dns-server [6]`
> - **Tag:** _(none — the interface handles it)_
> - **Value:** `192.168.1.101`

> [!warning] Existing leases keep the old DNS until they renew (~50% of lease time). Toggle Wi-Fi / reboot a device to force it, or rely on the NAT redirect in §4 which catches them anyway.

---

# 2. Local hostname resolution (Unbound → Dnsmasq)

So `*.lan.internal` names and reverse (PTR / device-name) lookups resolve through the chain.

> [!info]- Query Forwarding (Services → Unbound DNS → Query Forwarding) **Forward lookup (A records):**
> 
> - Domain: `lan.internal` · Server IP: `127.0.0.1` · Port: `53053` · Forward first: `off`
> 
> **Reverse lookup (PTR / device names in Pi-hole):**
> 
> - Domain: `1.168.192.in-addr.arpa` · Server IP: `127.0.0.1` · Port: `53053` · Forward first: `off`

Because Pi-hole forwards _everything_ to Unbound, and Unbound peels off `lan.internal` to Dnsmasq, local names resolve with no extra Pi-hole conditional-forwarding config.

---

# 3. DNSSEC

> [!info] DNSSEC is validated in **exactly one place — Unbound**.
> 
> - **Unbound:** DNSSEC `ON` ✅
> - **Pi-hole:** DNSSEC `OFF` — Unbound already validates; enabling it here is redundant and can break resolution.
> - **Dnsmasq:** DNSSEC `OFF` — only serves unsigned local names.
> 
> The validated `ad` (authentic data) flag still flows through Pi-hole to clients, so the full security benefit is kept.

---

# 4. Force ALL DNS through Pi-hole (transparent redirect)

Catches devices with hardcoded DNS (Chromecast/Android → `8.8.8.8`) and any client still on an old lease. This is **Phase 4 hardening**.

### Step 1 — Outbound NAT for reflection (one-time)

`Firewall → Settings → Advanced` → enable **Automatic outbound NAT for Reflection**.

> Needed so the redirected packet's **source** is rewritten to the firewall IP — otherwise Pi-hole replies directly over the LAN, the client sees a reply from the "wrong" IP, and DNS breaks (asymmetric path).

### Step 2 — Destination NAT rule

`Firewall → NAT → Destination NAT` → `+` _(this is the renamed "Port Forward")_

> [!info]- Rule fields
> 
> |Field|Value|
> |---|---|
> |Interface|`LAN`|
> |TCP/IP Version|`IPv4`|
> |Protocol|`TCP/UDP`|
> |**Invert Source**|**`✓`**|
> |Source Address|`Single host or Network` → `192.168.1.101/32`|
> |Source Port|any|
> |Destination Address|any|
> |Destination Port|`53`|
> |Redirect Target IP|`192.168.1.101`|
> |Redirect Target Port|`53`|
> |NAT Reflection|`Enable`|
> |Firewall rule|`Register rule`|

> [!important] **Invert Source + Pi-hole IP** = "every LAN device _except_ Pi-hole." This is the loop prevention — Pi-hole's own upstream queries to Unbound (`192.168.1.1:53`) are excluded, so they aren't redirected back into Pi-hole.

> [!info] New-UI name mapping
> 
> - `Firewall rule` = old _Filter rule association_
> - `Register rule` = old _Add associated filter rule_ (creates the visible linked pass rule in **Rules [new] → LAN**)
> - No reordering needed — NAT is always evaluated before filter rules.

---

# 5. Optional hardening

- **Block DoT:** add a LAN firewall rule rejecting destination port `853`, so devices trying DNS-over-TLS fall back to plain `:53` (then caught by the redirect).
- **DoH (port 443):** hard to block — looks like normal HTTPS. Mitigate with Pi-hole blocklists that target known DoH endpoints.
- **MAC alias** `MAC_Pihole` (`BC:…`) — kept for the alternative "block all non-Pi-hole DNS" filter-rule method. _Not_ used by the NAT redirect (NAT matches on IP, and Pi-hole is static at `.101`).

---

# Verification

```bash
# From a client — should be answered by Pi-hole, NOT Google:
dig blocked-domain.com @8.8.8.8        # expect 0.0.0.0 / NXDOMAIN for a blocked domain

# DNSSEC validation (run on OPNsense or via dig to Unbound):
dig sigok.verteiltesysteme.net @192.168.1.1     # NOERROR + 'ad' flag
dig sigfail.verteiltesysteme.net @192.168.1.1   # SERVFAIL

# Local hostname resolution:
dig <device-name>.lan.internal @192.168.1.101
```

- Pi-hole dashboard shows queries + active clients > 0.
- New DHCP leases hand out DNS = `192.168.1.101`.

---

> [!quote]- References
> 
> - OPNsense — Dnsmasq DNS & DHCP: `docs.opnsense.org/manual/dnsmasq.html`
> - OPNsense — NAT / Reflection & Hairpin: `docs.opnsense.org/manual/how-tos/nat_reflection.html`
> - OPNsense — Unbound DNS: `docs.opnsense.org/manual/unbound.html`
> - Pi-hole — OPNsense router guide: `docs.pi-hole.net/routers/opnsense/`
> - Pi-hole — Unbound recursive guide: `docs.pi-hole.net/guides/dns/unbound/`
> - lguruprasad.in — OPNsense DNS redirect to Pi-hole (Jan 2025)

