# Trust Zones — Enterprise-in-a-Box Phase 1

| Zone | Who Lives There | Trust Level | Comments |
|------|-----------------|--------------|-----------|
| **Management** | Admin tools, jump hosts | Tier 0 — Highest trust | Can RDP anywhere inside. No Internet. |
| **Server / AD** | Domain Controllers, DNS | Tier 1 | Only speaks to Mgmt + Workstations. Never DMZ. |
| **Workstation** | User machines | Tier 2 | Authenticates to AD, can reach DMZ over HTTPS. |
| **DMZ** | Public-facing stuff | Untrusted | One-way inbound from Internet; blocked from inside. |
| **Internet** | Everything outside | Hostile by default | Only allowed through specific ports (80 → 443). |

---

**Summary:**  
This tiered layout keeps the blast radius small and defines who can talk to whom.  
Compromise in one zone cannot directly escalate to Tier 0 or the DC.
