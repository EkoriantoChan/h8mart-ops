# 04 · Governance & Integration Readiness Checklist · H8Mart Quick-Commerce Ops

**Author:** H8Mart Operations & Governance Team  
**Evaluation Target Date:** November 2026  
**Compliance Standard:** Indonesian UU No. 27/2022 (UU PDP), ISO 27001 Access Control Principles  

---

## 1. Credential & Access Control Readiness

Every item in this checklist has been verified against the production export workflows, database policies, and repository commit audits.

| Inspection Item | Status | Verification Evidence & Audit Details |
| :--- | :---: | :--- |
| **1. Credential Storage Isolation**<br/>All credentials stored in n8n Credentials store; zero hardcoded credentials in nodes, Code scripts, or repo files. | **READY (Siap)** | `git grep -i "sk-proj-"` and `git grep -i "bot[0-9]"` return zero hits in repo. Workflows use credential entity references (`cred-pg-reader`, `cred-pg-ops`, `cred-header-auth`, `cred-mock-api-key`, `cred-openai`, `cred-telegram`). |
| **2. Principle of Least Privilege Database Roles**<br/>Separate DB roles: `mcp_reader` (read-only for WF2 & WF3 reading) vs `ops_writer` (write access for `ops.*` and stock upsert). | **READY (Siap)** | Verified via `evidence/wf2/04-mcp-reader-insert-rejected.txt`. Database command `INSERT INTO core.merchant_stock ...` executed as `mcp_reader` is strictly rejected with `ERROR: permission denied for table merchant_stock`. |
| **3. Webhook Authentication Security**<br/>All ingress webhooks (`order.created`, `ops-events`, `merchant-assistant`) protected with Header Authentication. | **READY (Siap)** | Verified via Test Case O5. Calls to `/webhook/order-created` without header `X-H8Mart-Token: h8mart-webhook-lab` are immediately rejected with HTTP 403 Forbidden. Export JSON files use placeholder `"id": "GANTI-ID-HEADER-AUTH"`. |
| **4. Zero Leaked Secrets in Repository**<br/>`.env` is uncommitted and listed in `.gitignore`; `.env.example` contains only non-secret mock placeholders. | **READY (Siap)** | `git status` confirms `.env` is ignored by Git. `.env.example` contains only generic template values (`h8mart-mock-key`, `h8mart-webhook-lab`). `OPENAI_API_KEY` is omitted or empty. |
| **5. Uptime Kuma Push Token Sanitization**<br/>Uptime Kuma push token replaced with `ISI_TOKEN_PUSH_KUMA` in workflow export prior to Git commit. | **READY (Siap)** | Inspected `workflows/wf1-stock-sync-order-intake.json` and `workflows/wf5-reliability-monitoring.json`. Node `Heartbeat Uptime Kuma` URL contains placeholder `http://uptime-kuma:3001/api/push/ISI_TOKEN_PUSH_KUMA?status=up&msg=OK&ping=`. |
| **6. Telegram Bot Token Isolation**<br/>Telegram Bot token resides solely in n8n credential database; never exposed in `.env`, Git history, or screenshots. | **READY (Siap)** | `.env` has no Telegram token variable (`SETUP.md` line 32). In `workflows/*.json`, Telegram nodes reference `"id": "GANTI-ID-TELEGRAM"`. Live token is managed exclusively through n8n Credentials Manager. |
| **7. Telegram Bot Chat Allowlist**<br/>Chat assistant access restricted strictly to registered `chat_id`s in `ops.telegram_merchants`; unauthorized IDs rejected. | **READY (Siap)** | Table `ops.telegram_merchants` maps user `chat_id` to authorized `merchant_id` (`M003`). Incoming messages from unregistered `chat_id`s receive a polite decline response and are blocked from calling LLM tools. |
| **8. Instance & Perimeter Ingress Restriction**<br/>n8n owner authentication active; internal services (Postgres, Mock API, Kuma) bound to loopback / Docker network. | **READY (Siap)** | AWS Security Group allows ingress only on ports 80 and 443 (HTTP/HTTPS) and 22 (SSH from My IP). Ports 5690, 5440, 8080, and 3011 are bound strictly to `127.0.0.1` and inaccessible from the public internet. |

---

## 2. Existing Systems & External Dependencies Readiness

| External / Internal System | Connection Method & Protocol | Known Rate Limits & SLAs | System Behavior during Outage | Escalation / Support Contact |
| :--- | :--- | :--- | :--- | :--- |
| **Merchant Stock API** | HTTP REST (`GET /merchants/{id}/stock`), Header `X-Api-Key` | 100 req/min per merchant. M007 has 2.5s latency; M009 returns permanent 500; M004 fails odd calls. | WF1 retries transient errors up to 3 times with exponential backoff. Permanent failure enters `ops.dead_letter`. Stock sync continues processing remaining 11 merchants. | Merchant Partner Integration Desk / Store Onboarding Manager (`ops-partners@h8mart.local`) |
| **Payment Gateway API** | HTTP REST (`POST /payments/refunds`, `/payments/vouchers`), Header `Idempotency-Key` | 300 transactions/min. SLA 99.9%. Order `-099` simulates permanent gateway 503 outage. | WF4 applies `maxTries: 3` with 1,000ms delay. If gateway remains down, transaction records in `ops.dead_letter`, and WF5 dispatches an immediate incident alert to Telegram Ops. | FinTech Payment Operations & Merchant Acquirer Support (`payments-ops@h8mart.local`) |
| **Notification API (Mock / Push)** | HTTP REST (`POST /notify`), Header `X-Api-Key` | 500 messages/min. In-memory buffer. | WF1, WF4, WF5 configure `onError: continueRegularOutput`. A notification failure does not roll back financial transactions or database upserts. | Internal Notification Infrastructure Team (`platform-notif@h8mart.local`) |
| **Telegram Bot API** | HTTPS (`api.telegram.org`), Bot Token Authentication | 30 messages/sec globally; 1 message/sec per individual chat. Markdown entity parsing requirements. | Node handles network drops via n8n retry. Messages use plain text or HTML entity mode to avoid Markdown parsing crashes on underscores (`_`). | Telegram Platform Support / Internal Bot Admin (`@BotFather` / H8Mart Tech Lead) |
| **PostgreSQL Database** | TCP Native Protocol (`postgres:5432`), internal Docker network | Maximum 100 concurrent connections. Target query response time < 50ms. | Container configured with Docker restart policy `always`. Database failures trigger Central Error Workflow (WF5), alerting Ops immediately. | H8Mart Database Administrator (DBA) (`dba@h8mart.local`) |
| **LLM Provider (OpenAI API)** | HTTPS REST (`api.openai.com/v1`), Bearer Token Authentication | Model `gpt-6-luna` / `gpt-4o-mini`. Rate limit: 200,000 TPM / 500 RPM. Default sampling temperature. | If API rate-limits or experiences outage, WF3 error branch triggers fallback, setting status to `eskalasi_cs`, logging ticket in `ops.tickets`, and responding gracefully to merchant. | OpenAI Platform Status (`status.openai.com`) / Course Instructor Technical Support |

---

## 3. Pre-Production Governance & Operational Verification

| Governance & Reliability Check | Status | Verification Evidence & Production Guardrail |
| :--- | :---: | :--- |
| **1. Central Error Workflow Attachment**<br/>WF5 configured as the designated Error Workflow for WF1, WF2, WF3, and WF4. | **READY (Siap)** | Verified via SQLite query on `workflow_entity.settings`. All 4 core workflows have `settings.errorWorkflow = "I9i62Q2X83vlS8A4"` (WF5 ID). Uncaught exceptions invoke WF5 instantly. |
| **2. Deliberate Failure & Dead-Letter Verification**<br/>Retry logic and dead-letter logging proven with synthetic failure test cases. | **READY (Siap)** | Verified via `evidence/wf5/03-dead-letter-m009-and-orders.txt`. Merchant `M009` and payment test order `HM-SP-099` correctly logged into `ops.dead_letter` with full payload and business key. |
| **3. Proactive Heartbeat & Silent Failure Detection**<br/>Push monitor tracking and zero-row anomaly interception active. | **READY (Siap)** | Verified via `evidence/wf5/05-uptime-kuma-push.txt` (Uptime Kuma push status: UP). Node `Ada Baris Stok?` halts silent outages (`/admin/stock-outage?on=1`) and routes to `Gagal: Sinkron 0 Baris`. |
| **4. Customer Data Minimization & Privacy Masking**<br/>No sensitive customer PII (credit cards, passwords, full NIK) logged in plain text. | **READY (Siap)** | Customer IDs (`C0100`) and order IDs (`HM-TEST-001`) are pseudonymized. No credit card numbers or raw bank credentials pass through webhook payloads. |
| **5. Rollback & Emergency Deactivation Runbook**<br/>Documented emergency procedures to deactivate misbehaving triggers without data loss. | **READY (Siap)** | Schedule triggers in WF1 can be deactivated immediately via n8n UI (right click $\rightarrow$ Deactivate $\rightarrow$ Publish) or CLI `n8n update:workflow --id=... --active=false`. Database state persists in Docker volume `h8mart_pgdata`. |
| **6. LLM Token Cost & Quota Monitoring**<br/>Token consumption tracked and bounded with prompt efficiency controls. | **READY (Siap)** | WF3 agent prompt engineered to keep input tokens $\approx 1,120$ and output $\approx 185$ tokens per run. `max_tokens` bounded. Monthly LLM cost projected at under US\$0.35/month. |
| **7. Timezone Standardization (`Asia/Jakarta`)**<br/>All schedule triggers and date evaluations explicitly configured to Western Indonesia Time. | **READY (Siap)** | WF1 Schedule triggers explicitly set to Timezone `Asia/Jakarta` (UTC+7) in workflow settings. All KPI queries anchor to `core.data_info.hari_data_terakhir` (`2026-11-10`). |
