# llm watchdog — phase 0

## problem
SOCs drown in noisy alerts; analysts miss the ones that actually matter. We need a **local** triage engine that filters routine/low-risk alerts, gives short evidence-based summaries, and proposes safe actions so humans focus on real incidents.

## outcome (what “done” means)
- **input:** one alert  
- **output:** JSON with `summary`, `confidence` (0–1), `mapped_mitre[]`, `proposed_actions[]`  
- **rule:** if confidence is high and action is on a safe list, auto-handle (or dry-run). Otherwise ask a human  
- **always:** write an audit record; anything we do is reversible; model has **no internet**

## scope (phase 0/1)
Alerts are ingested, triaged **locally** by an **offline** model with signed, fixed context, and produce a small JSON answer. A simple gate decides `simulate` (dry run), `approve` (ask a human), or `execute` (do it). Default is **simulate**. Every step is audited; any allowed action has a rollback; the model host has **no outbound network**; all components run with least privilege.

---

## how it works (end-to-end)

### setup (once)
1. write `policy.yaml` (house rules):  
   `default_mode: simulate`, `confidence_threshold: 0.85`, `allowlist_actions`,  
   `blast_radius.max_targets_per_action: 1`, `rate_limits`, a couple of rules
2. sign the policy (policy maintainer signs; app verifies with public key). if verify fails → **simulate** everything
3. freeze interfaces: exact JSON for incoming alerts and model output

### each alert
4. **ingest**  
   source sends alert with HMAC header  
   - bad HMAC → reject and log `ingest.reject`  
   - good HMAC → store alert + integrity hash and log `ingest.accept`
5. **triage (local model)**  
   worker pulls new alerts, calls **offline** model, expects strict JSON:  
   `summary, confidence, mapped_mitre[], proposed_actions[], evidence[], policy_tags[]`  
   - invalid JSON → don’t mark processed; log `worker.parse_error`; retry later
6. **gate (the bouncer)**  
   checks in order: policy signed, action allowlisted, blast radius = one explicit target, idempotent, confidence ≥ threshold, under rate limits  
   decision: any check fails → **simulate**; all pass + rule needs human → **approve**; all pass + rule allows → **execute**
7. **act (phase 0/1 defaults to simulate)**  
   - **simulate:** plan only; log what would happen and the rollback; touch nothing  
   - **approve:** show button to a human; if they click and checks still pass, proceed  
   - **execute (later phases):** do the action; idempotent, single-target, with a defined rollback

### receipts
8. **telemetry (breadcrumbs for engineers)**  
   emit at each step:  
   `ingest.accept/reject`, `worker.model_call`, `worker.parse_error`,  
   `policy.decision {mode, confidence, checks}`, `soar.exec/rollback` (when real),  
   `audit.snapshot.created`
9. **audit (formal receipts)**  
   append one row per decision/change: `simulate_plan`, `approve_request`, `execute`, `rollback` with who/what/when/why/target + row hash; nightly signed snapshot of the day’s rows

---

## not doing this in phase 0 (the fence)
1. no SOAR connectors (simulate-only; no state-changing APIs)  
2. no “AI runs the SOC” (AI handles boring, repeatable alerts only; riskier stuff to a human)  
3. no complex alert logic (single alert in → single decision out)  
4. no fine-tuning or training (local Ollama, stock model)  
5. no centralized/shared LLM service (worker talks to local model process)  
6. no extra cloud infra beyond Supabase (no VMs/K8s/serverless; worker is local)  
7. no production tenants or real org data (dev/sandbox only)  
8. no state-changing actions (disable/isolate/delete/block forbidden; gate forces simulate)  
9. no wide-blast operations (more than one target → refused)  
10. no schema drift (model must return valid triage JSON)  
11. no external lookups (no VT/OSINT/SaaS; model host has **no outbound network**)  
12. no UI polish or multi-tenant auth (minimal Lovable UI; no SSO/RBAC)  
13. no bespoke ingestion frameworks (use the open-source ingestor as-is; no bus/broker circus)  
14. no policy edits without signature (unsigned policy ignored; simulate)  
15. no silent daemons (every component emits telemetry)

---

## telemetry vs audit 
**telemetry:** small, structured events so engineers can see what happened. chatty, rotates out  
example:
json
{"event":"policy.decision","alert_id":"a1","mode":"simulate","confidence":0.92,"checks":{"allowlisted":true,"blast_radius_ok":true}}

**audit:** append-only receipts for decisions/changes. who/what/when/why/target, tamper-evident
example:

{"ts":"2025-10-27T21:40:00Z","actor":"system/policy_gate","action":"simulate_plan","obj":"alert:a1","details":{"rule":"auto-close-benign","confidence":0.92,"proposed_action":{"action":"close_alert","target":"alert:a1"},"rollback":{"action":"reopen_alert","target":"alert:a1"}}}


# audit vs telemetry 

**Rule of thumb:** if it **changes state** or **commits a decision**, it’s **audit**.  
Everything else useful to see is **telemetry**.

## glossary (short and sharp)

- **simulate:** dry run. plan it, log it, change nothing  
- **allowlist:** the actions allowed to auto-run  
- **blast radius:** max impact per action; capped at one target  
- **idempotent:** safe to retry; doing it twice doesn’t stack damage  
- **confidence:** model’s 0–1 “how sure”; you set the pass mark  
- **rate limit:** caps per hour so nothing stampedes  
- **signed policy:** `policy.yaml` approved and cryptographically stamped; unsigned = simulate all  
- **rollback:** the exact undo step for any real action

---

# sources — phase 0 (definition only)

We are defining sources, not implementing pipelines. Exactly **two** sources. No third “for fun.”

## source A: zeek dns
- **Purpose:** basic network signal to distinguish routine DNS (time sync, update servers) from noisy junk.
- **Why chosen:** easy to generate locally or mock; low risk; good for “close-as-known-good.”
- **Scope note:** payloads small; used only for triage demos in simulate mode.

## source B: entra id sign-ins
- **Purpose:** authentication signal (failed→success patterns, unusual IP/geo) to test triage summaries.
- **Why chosen:** common enterprise noise pattern; simple fields; easy to mock for demos.
- **Scope note:** dev/sandbox or mocked data only in Phase 0; simulate mode only.

## fence (reminder)
- Only **Zeek DNS** and **Entra ID sign-ins** in Phase 0.
- No new sources until Phase 0 acceptance is done.
- 

# success metrics — phase 0/1

Numbers, not vibes. Three metrics: one proves receipts (audit), one proves speed (performance), one proves restraint (safety).

---

## 1) integrity — audit coverage

**Goal:** every accepted alert gets at least one **audit row** (a formal receipt).

**Audit row = one decision receipt** with:
- **who** → `actor` (e.g., `system/policy_gate`, `user:anish`)
- **what** → `action` (`simulate_plan` | `approve_request` | `execute` | `rollback`)
- **when** → `ts`
- **target** → `obj` (e.g., `alert:a1`)
- **why** → `details` (rule, confidence, checks, reason, proposed/rollback)
- **row_hash** → SHA-256 over the row contents (tamper-evident)

**Metric (with SLA):**
- **Audit coverage % =**  
  `(# unique alerts with ≥ 1 audit row within SLA) / (# alerts accepted) * 100`
- **SLA window:** 60 seconds from `ingest.accept` to first audit row.

**Target:** **≥ 95%** over any rolling 24h window.

**Counts that satisfy “has an audit row”:**
- First audit `action` is any of: `simulate_plan`, `approve_request`, `execute` (rare in P0/1).  
  `rollback` rows exist, but the first row should normally be one of the three above.

**Pass/Fail example:**
- Accepted alerts today: **A = 120**  
- Alerts with a first audit row ≤ 60s: **B = 116**  
- Coverage = `116/120 * 100 = 96.7%` → **PASS**  
- Misses (4) are defects to fix: parse errors beyond SLA, policy verify not downgrading to simulate, audit write failure, etc.

**Why this metric matters:**  
An accepted alert without a decision receipt (audit row) is a pipeline hole, not “still thinking.”

---

## 2) performance — triage latency (p95)

**Goal:** be faster than a human skim for small alerts.

**Definition:** time from worker picking an alert to valid model JSON returned.

**Scope:** only alerts with payload size **≤ 4 KB**.

**Metric:**
- **p95 triage latency ≤ 2.0 s**

**How to compute p95 (simple):**
1. Collect `latency_ms` for eligible alerts in the window.
2. Sort the list.
3. Take the value at the 95th percentile index (round up).

**Example:**
- Latencies (ms): `[600, 700, 820, 900, 1100, 1300, 1700, 1900, 2100, 2400]`  
- 10 samples → 95th percentile = value at index 10 → **2400 ms (2.4 s)** → **FAIL**  
- If max were 1800 ms → **1.8 s** → **PASS**

**Fail threshold:** p95 > 2.0 s for **2 consecutive days**.

---

## 3) safety — unsigned policies

**Goal:** never act on rules we can’t trust.

**Metric:**  
- **Unsigned policy loads (last 24h) = 0**

**How to count:**
- Count telemetry events `policy.verify_fail` (policy signature invalid/missing).
- System behavior must also force **simulate** on verify failure, but the metric still fails if any occur.

**Target:** **0** every day.

---


## trust zones

| Zone     | What it is               | Inbound from            | Outbound to      | Guardrails                 | If broken…               |
|----------|---------------------------|-------------------------|------------------|----------------------------|--------------------------|
| Sources  | Zeek DNS, Entra sign-ins  | n/a                     | Worker (HTTP)    | small messages (≤ 4 KB)    | Noisy/fake alerts enter  |
| Worker   | your small local app      | Sources (HTTP)          | Database (writes)| **no internet**, simulate  | Bad/slow summaries       |
| Database | Supabase tables           | Worker (writes/reads)   | UI (read-only)   | least-privilege roles      | Data integrity risk      |
| UI       | Lovable (read-only)       | Database (read-only)    | n/a              | read-only access           | Misleading views only    |

**Flow:** Sources → Worker → Database → UI  
**Model:** Worker ↔ Local model (no internet)

---

## identities & permissions (minimum)

| Name       | DB permissions                                      | Purpose                                | Rotate |
|------------|------------------------------------------------------|----------------------------------------|-------:|
| `app_svc`  | **INSERT:** `alerts`, `triage`<br>**SELECT:** `alerts` | Worker writes and re-reads alerts for processing/retry | 90d    |
| `ui_reader`| **SELECT (RO):** `alerts`, `triage`                   | UI display only                        | 90d    |

Rules: no `UPDATE`, no `DELETE`, no DDL. If you think you need them, you don’t.



## telemetry (minimal so metrics work)

ingest.accept{alert_id, source, ts, size_bytes}

ingest.reject{source, reason, ts}

app.model_call{alert_id, model, latency_ms, payload_bytes, ts}  app = worker 

app.parse_error{alert_id, reason, ts}

app.triage_write{alert_id, triage_id, ts}

---


