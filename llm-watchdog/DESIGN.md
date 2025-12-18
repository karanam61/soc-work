# Security Log Worker - Design Thinking

## How I Thought Through This Design

### Understanding Logs

**Me:** How do applications interpret logs? How does the worker take logs in from Zeek?

**Answer:** 
- Logs arrive via HTTPS endpoints or message queues
- Worker parses them (extract IPs, ports, timestamps, event types)
- Normalize to common schema
- Match against predefined rules
- Trigger actions based on matches

**Me:** How is a log actually created?

**Answer:**
- Application code writes logs (e.g., `log.info("User login")`)
- Network devices (Zeek) inspect traffic and generate entries
- Written to files or sent to collectors

**Me:** Do Zeek/Splunk give alerts or raw logs?

**Answer:**
- Zeek: Raw logs (passive observer)
- Splunk: Both raw logs AND alerts you configure

---

### Worker Design (No AI)

**My Decision:** Worker receives logs via HTTP endpoint from Zeek

**Processing Flow:**
1. Log arrives
2. Parse it
3. Map to MITRE technique (using Sigma rules)
4. Look up severity from local database
5. Route to priority or standard queue

---

### Severity Classification

**Me:** How do we decide severity?

**Answer:** Use MITRE ATT&CK mapping

**My Decision:** 
- Categories: Critical/High vs Medium/Low (not numbers)
- Critical/High = filesystem damage, infrastructure disruption
- Medium/Low = reconnaissance, discovery

**Me:** Should it be static or dynamic?

**Answer:** 
- Static = you define rules upfront
- Dynamic = adjust based on threat intel (70% of companies hit by X)

**My Decision:** Use dynamic severity with threat intel feeds

---

### MITRE Mapping

**Me:** How do we map a raw log to MITRE technique?

**Answer:** 
- Keyword matching
- Sigma rules (pre-written with MITRE mappings)
- Field-based logic

**My Decision:** Use Sigma rules

---

### Queue Logic

**My Decision:**
- Two queues: Priority (Critical/High) and Standard (Medium/Low)
- Critical/High → send immediately
- Medium/Low → wait 2-3 minutes, batch check for more severe alerts
- If Critical arrives while Low is waiting, pause Low queue, send Critical first

---

### Storage & Backup

**My Decision:**
- MITRE mappings stored in local database (Supabase)
- If database down: store unprocessed logs on AWS, process later

---

### Data Sources

**My Decision:**
- Use Sigma rules for log-to-MITRE mapping
- Threat intel source: TBD (MISP, AlienVault OTX, or CISA)
- Update frequency: Daily batch updates

---

### Output

**My Decision:**
- Frontend: Loveable
- Alerts displayed in order of priority
- Human gets alerts after filtering

---

## Final Architecture

```
Zeek Logs → HTTP Endpoint → Worker
                               ↓
                          Parse Log
                               ↓
                    Apply Sigma Rules
                               ↓
                   Map to MITRE Technique
                               ↓
                  Query Local DB for Severity
                               ↓
                    Critical/High? Medium/Low?
                      ↓                    ↓
              Priority Queue        Standard Queue
              (immediate)          (wait 2-3 min)
                      ↓                    ↓
                   Loveable Frontend
                               ↓
                      Human Analyst
```

---

## What's Next

1. Learn Python basics (needed to build this)
2. Build log parser
3. Set up Supabase with MITRE severity table
4. Implement queue logic
5. Create HTTP endpoint
6. Build basic frontend
7. Test with real Zeek logs

---

## Phase 1: No AI
- Pure rule-based
- Manual updates based on human feedback
- Sigma rules + MITRE mapping

## Phase 2: Add AI (Later)
- AI handles unknown patterns
- Confidence scoring
- Learning from human feedback
