# 🧠 SOC-WORK — Enterprise SOC & AI Security Architecture

This repository is your **central workspace** for building a defensible enterprise-grade SOC (Security Operations Center) that combines  
**cloud engineering, AI automation, and detection engineering** into one cohesive system.

---

## 🚀 Overview

| Area | Description |
|------|--------------|
| **Enterprise-as-a-Box** | Complete Azure SOC lab built via Terraform — Active Directory, Sentinel, Sysmon, detections, and automation. |
| **LLM Watchdog** | Autonomous SOC AI system that triages alerts, enforces policy, and verifies actions through Supabase + Ollama + local RAG. |
| **Foundations-TryHackMe** | Blue-team case library with Sigma/KQL detections and real evidence from TryHackMe labs. |
| **Infra-Terraform** | Shared Terraform provider and backend used to deploy and manage Azure resources securely. |
| **Personal-Docs** | Communication practice, interview prep, progress trackers, and personal development notes. |

---

## 🧩 Repository Structure


---

## 🔗 Quick Links

- [Enterprise-as-a-Box](enterprise-as-a-box/README.md)  
- [LLM Watchdog](llm-watchdog/README.md)  
- [Foundations – TryHackMe](foundations-tryhackme/README.md)  
- [Infra – Terraform](infra-terraform/README.md)  
- [Personal Docs](personal-docs/README.md)

---

## 🧱 Tech Stack

- **Cloud:** Azure, Sentinel, Entra ID, Defender for Cloud  
- **Infra as Code:** Terraform, PowerShell  
- **Telemetry:** Sysmon, AMA, Log Analytics, KQL  
- **AI Layer:** Ollama (Qwen 2.5 / Llama 3), Supabase, Chroma DB  
- **Automation:** Logic Apps, SOAR stubs, Python worker  
- **Governance:** RLS, policy-as-code, auditable metrics

---

## 📅 Timeline & Focus

- **Daily Split:** 3 h SOC  +  3 h Watchdog  +  1 h TryHackMe  
- **Final Goal:** Job-ready by **Feb 9 2026**, demonstrating both enterprise SOC and AI-security integration.

---

## 🧠 Learning Philosophy

> Build, break, observe, harden — repeat.  
> Every phase must have logs, metrics, and evidence.  
> Progress is measured in **artifacts**, not hours spent.

---

## ✅ Milestone Tracking

| Week | Focus | Core Deliverables |
|------|--------|------------------|
| 1-2 | Trust Fabric & Architecture | Azure VNet, AD, DNS, docs/system-architecture.md |
| 3-4 | Policy & Observability | NSG rules, Sysmon config, Kerberoast detections |
| 5-6 | Cloud Integration | Defender for Cloud, Entra ID logging, hybrid correlation |
| 7-8 | AI Integration | LLM Watchdog policy engine + SOAR loop |
| 9-10 | Verification & Chaos | CI/CD tests + security resilience metrics |

---

## 🧩 Outcome

By the end of this roadmap, you’ll have:
- A **hybrid SOC** spanning on-prem + cloud.  
- A **secure AI agent** that triages incidents safely.  
- A full **audit trail** proving integrity and automation value.  
- Public GitHub artifacts showing **enterprise-grade maturity**.

---

*Last updated: %date%*
