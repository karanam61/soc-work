# LLM Watchdog - Autonomous SOC AI System

Security decision system that automates triage and safe low-risk actions with strict policy gates and full auditability.

## Layers
- Data: Supabase tables (alerts, triage, actions, labels)
- Model: Local Ollama (Qwen2.5/Llama3.1) strict JSON
- RAG: Chroma with MITRE, Sigma, Playbooks, Policies, OSINT
- Policy: Verifier + YAML allowlists/thresholds/blast radius
- SOAR: Simulated tools with rollback

Directories: worker/, prompts/, supabase/, chroma/, docs/
