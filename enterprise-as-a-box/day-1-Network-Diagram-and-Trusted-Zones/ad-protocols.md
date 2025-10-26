\# Phase 1 – Day 1 Report — AD Core Protocols + Trust Zones



\## Objective

Start building the base of the Enterprise-in-a-Box lab on Azure.  

Today’s focus was just understanding how the domain actually talks — what ports, what protocols, and where the trust boundaries lie before I start locking anything down.



---



\## 1. Network Layout Recap

We’ve got four main subnets right now inside one VNet:



| Subnet | Purpose | Example IP Range | Notes |

|---------|----------|------------------|-------|

| \*\*Server / AD\*\* | Runs Active Directory DS + DNS | 10.0.2.0/24 | Handles all auth and directory stuff |

| \*\*Workstation\*\* | Domain-joined employee machine | 10.0.1.0/24 | Talks to AD for tickets, DNS, policies |

| \*\*DMZ\*\* | Exposed web server | 10.0.3.0/24 | Only HTTPS open to Internet |

| \*\*Management\*\* | Admin zone | 10.0.4.0/24 | Where RDP/WinRM originate from (Tier 0) |



The diagram (`diagrams/phase1-topology.png`) shows this layout and ports between them.



---



\## 2. Trusted Zone Logic

I split the environment mentally into trust layers instead of just subnets:



| Zone | Who Lives There | Trust Level | Comments |

|------|-----------------|--------------|-----------|

| \*\*Management\*\* | Admin tools, jump hosts | Tier 0 — Highest trust | Can RDP anywhere inside. No Internet. |

| \*\*Server / AD\*\* | Domain Controllers, DNS | Tier 1 | Only speaks to Mgmt + Workstations. Never DMZ. |

| \*\*Workstation\*\* | User machines | Tier 2 | Authenticates to AD, can reach DMZ over HTTPS. |

| \*\*DMZ\*\* | Public-facing stuff | Untrusted | One-way inbound from Internet; blocked from inside. |

| \*\*Internet\*\* | Everything outside | Hostile by default | Only allowed through specific ports (80 → 443). |



This keeps the “blast radius” small — if DMZ or a workstation gets owned, it can’t climb straight into Tier 0 or the DC.



---



\## 3. Protocol Breakdown (Simple Explanations)



\*\*Kerberos – Port 88 (TCP/UDP)\*\*  

Used for domain authentication. Instead of sending passwords every time, the DC issues tickets. Everything inside the domain depends on this staying in sync with time.



\*\*LDAP / LDAPS – Ports 389 / 636 (TCP)\*\*  

Directory queries and updates. LDAP is clear-text, LDAPS is the encrypted version. Used by domain joins, GPOs, and management tools.



\*\*DNS – Port 53 (TCP/UDP)\*\*  

AD-aware DNS. Every client resolves service records here (\_ldap.\_tcp.dc.\_msdcs.corp.local\_). Forwarders handle external lookups.



\*\*SMB – Port 445 (TCP)\*\*  

File sharing and GPO delivery (SYSVOL, NETLOGON). Needs protection; that’s where pass-the-hash lives.



\*\*RPC – Port 135 + Dynamic 49152–65535 (TCP)\*\*  

Used by WMI, replication, and most admin actions. Must stay open between Mgmt ↔ Server, not from DMZ.



\*\*NTP – Port 123 (UDP)\*\*  

Time sync for Kerberos and log correlation. DC PDC Emulator acts as time master.



\*\*RDP – Port 3389 (TCP)\*\*  

Admin channel. Only Mgmt → others. No Workstation → DC, no DMZ → anything.



---



\## 4. Port Summary



| From | To | Ports | Why |

|------|----|-------|-----|

| \*\*Workstation\*\* | AD | 53, 88, 389/636, 445, 135 + 49152–65535, 123 | Authentication + DNS + GPOs |

| \*\*Management\*\* | AD / WS / DMZ | 3389, 5985–5986, 22 | Remote admin only |

| \*\*Workstation\*\* | DMZ | 443 | Access company web apps |

| \*\*Internet\*\* | DMZ | 80 → 443 only | Public web access |

| \*\*DMZ\*\* | Internal | None | Blocked entirely |



---



\## 5. Files Added (Day 1)



\- `diagrams/phase1-topology.png` – updated network diagram  

\- `docs/ad-protocols.md` – this write-up  

\- `docs/trust-zones.md` – short summary table (same as above)



