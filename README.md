# DropMeds — AI Cold Email Sales Operating System (n8n)

> Production-grade, agentic, multi-workflow B2B outbound system. Drop the JSONs into n8n, plug in API keys, and start generating qualified meetings on autopilot.

---

## Overview

DropMeds is a **multi-agent cold email sales OS** built entirely in **n8n**. One Parent Orchestrator coordinates ten specialized AI sub-agents that source, qualify, enrich, verify, personalize, write, send, follow up, sync, and report on every B2B prospect — end-to-end, 24/7.

It replaces a $30/yr SDR + a $150/mo Outreach.io + a $100/mo enrichment stack with a single n8n instance you control.

**Designed for:** B2B SaaS founders, agencies, RevOps leaders, and consultants who want a deployable outbound machine.

---

## Features

- **10 specialized AI agents** with full system prompts (no placeholders)
- **Parent Orchestrator** with CRON, manual, and webhook triggers
- **Apollo.io + Proxycurl + SerpAPI** enrichment stack
- **Dual email verification** (NeverBounce primary, MillionVerifier fallback)
- **AI-graded personalization** with auto-rewrite loop on low-quality first lines
- **4-touch sequence generator** that writes the entire arc in one call
- **Reply sentiment classifier** with 6-way routing (Positive / Neutral / Negative / OOO / Unsubscribe / Referral)
- **Hot-reply alerts** to Slack with embedded Calendly link
- **Daily send caps + human-jitter timing** for inbox protection
- **Postgres-backed deduplication, suppression, and DNC enforcement**
- **HubSpot CRM sync** with auto-Deal creation on positive replies
- **Weekly executive memo** AI-written for the CEO every Friday at 6 PM
- **Google Sheets dashboard** auto-append for finance/board reporting
- **Full retry logic, error branches, and Slack alerting**

---

## Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    01 - PARENT ORCHESTRATOR                              │
│   (CRON 8AM Mon-Fri | Manual | Webhook /dropmeds-launch-campaign)        │
└──────────────────────────────────────────────────────────────────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        ▼                          ▼                          ▼
┌──────────────┐          ┌──────────────┐          ┌──────────────┐
│ 02. Finder   │  ───►    │ 03. Qualify  │  ───►    │ Quality Gate │
│ Apollo.io    │          │ AI Score 0-100│          │ score ≥ 70   │
└──────────────┘          └──────────────┘          └──────────────┘
                                                            │
        ┌──────────────────────────┬────────────────────────┘
        ▼                          ▼
┌──────────────┐          ┌──────────────┐
│ 04. Enrich   │  ───►    │ 05. Verify   │ ───► Email Gate
│ Web+LI+News  │          │ NB→MV        │      (valid only)
└──────────────┘          └──────────────┘
                                  │
        ┌─────────────────────────┘
        ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│ 06. Persona- │ ─► │ 07. Email    │ ─► │ 08. Sender   │ ─► │ 09. Follow-  │
│ lization +   │    │ Writer       │    │ + jitter+cap │    │ up + reply   │
│ rewrite loop │    │ E1+E2+E3+E4  │    │              │    │ classifier   │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                   │
                              ┌────────────────────────────────────┘
                              ▼
                    ┌──────────────┐    ┌──────────────┐
                    │ 10. CRM Sync │ ─► │ 11. Reporting│
                    │ HubSpot+Deal │    │ Slack+Sheets │
                    └──────────────┘    └──────────────┘
```

### Stage state machine (Postgres)

```
sourced → qualified → enriched → verified → personalized → sequence_ready
       → sent_t1 → sent_t2 → sent_t3 → sent_t4
       → hot_reply | do_not_contact | paused_ooo | unsubscribed
```

---

## Setup Guide

### 1. Prerequisites

- n8n self-hosted (Docker) or n8n.cloud account, version 1.60+ recommended
- PostgreSQL 14+ instance (Supabase, Neon, RDS — anything works)
- Gmail Workspace inbox dedicated to outbound (or any SMTP)
- Slack workspace with two channels: `#sales-ops-alerts`, `#sales-hot-replies`, `#sales-ops-reports`
- HubSpot account (free tier works)
- Google Sheet for daily reporting

### 2. Database setup

Run the SQL in [`/docs/schema.sql`](docs/schema.sql) on your Postgres to create three tables:

- `dropmeds_leads` — master record per prospect
- `dropmeds_outreach_log` — every send (touch 1-4)
- `dropmeds_suppression` — global do-not-contact list

### 3. Import workflows (in this order)

```
1. 02-Lead-Finder.json
2. 03-Qualification-Agent.json
3. 04-Enrichment-Agent.json
4. 05-Email-Verification.json
5. 06-Personalization-Agent.json
6. 07-Cold-Email-Writer.json
7. 08-Outreach-Sender.json
8. 09-Followup-Agent.json
9. 10-CRM-Agent.json
10. 11-Reporting-Agent.json
11. 01-Parent-Orchestrator.json   ← import LAST
```

### 4. Wire Parent Orchestrator → Children

After all 11 are imported, open `01 - Parent Orchestrator` and inside each `Execute Workflow` node, replace `REPLACE_WITH_*_WF_ID` with the actual workflow ID from your n8n instance (visible in the workflow URL).

### 5. Connect credentials

Replace `REPLACE_WITH_*_CRED_ID` placeholders in every node by attaching the relevant credential in the n8n UI.

### 6. Set environment variables

In n8n → Settings → Variables (or your `.env`):

```env
APOLLO_API_KEY=...
PROXYCURL_API_KEY=...
SERPAPI_KEY=...
NEVERBOUNCE_API_KEY=...
MILLIONVERIFIER_API_KEY=...
DROPMEDS_PRIMARY_MAILBOX=outbound@yourdomain.com
DROPMEDS_REPLY_TO=outbound@yourdomain.com
DROPMEDS_SENDER_NAME=Alex from DropMeds
DROPMEDS_DAILY_CAP=80
```

### 7. Activate

- Activate all 10 children
- Activate `01 - Parent Orchestrator` last
- The first run kicks off Mon 8AM, or trigger manually

---

## API Keys Needed

| Provider | Purpose | Cost (starter) |
|---|---|---|
| OpenAI | All AI agents (GPT-4o + 4o-mini) | ~$0.05/lead |
| Apollo.io | Lead sourcing | $49/mo (Basic) |
| Proxycurl | LinkedIn enrichment | $0.01/profile |
| SerpAPI | News signals | $50/mo |
| NeverBounce | Email verification | $0.008/email |
| MillionVerifier | Verification fallback | $0.0005/email |
| Gmail / Workspace | Sending inbox | $6/user/mo |
| HubSpot | CRM | Free tier OK |
| Slack | Alerts | Free |

**Total per 1,000 leads:** ~$60 all-in.

---

## Import Steps (one-liner version)

```bash
# 1. In n8n UI → Workflows → Import from File → upload each JSON
# 2. Open each → set credentials → save
# 3. Open Parent Orchestrator → set sub-workflow IDs → save → activate
# 4. Trigger via webhook:
curl -X POST https://your-n8n.com/webhook/dropmeds-launch-campaign \
  -H "Content-Type: application/json" \
  -d '{"icp":{"industry":"SaaS","employee_range":"50-500","titles":["VP Sales","CRO"],"geos":["United States"]},"daily_send_limit":80}'
```

---

## How Clients Can Use It

**Solo founder selling B2B SaaS:**
1. Define ICP once (titles, industry, geo, tech stack).
2. Trigger Parent Orchestrator daily.
3. Wake up to a Slack channel of hot replies with Calendly links pre-attached.

**Outbound agency:**
1. Clone this entire system per client.
2. Each client = own n8n folder, own Postgres schema, own Gmail inbox.
3. Bill $3-5k/mo per client; agency margin sits >70%.

**Mid-market RevOps team:**
1. Replace Outreach.io / Apollo Sequences entirely.
2. Plug into existing HubSpot — no migration needed.
3. SDRs spend 100% of time on positive replies, 0% on list building.

---

## ROI Explanation

Conservative model — 1,000 well-targeted contacts/month:

| Metric | Without DropMeds | With DropMeds |
|---|---|---|
| Reply rate | 1.5% | 4-7% |
| Meetings booked | 4 | 18-25 |
| SDR hours/week | 40 | 2 (review only) |
| Stack cost/mo | $1,200 | $250 (APIs) |
| Time-to-first-meeting | 3 weeks | 3 days |

**At $20k average ACV and 25% close rate from meeting → ~$100k+ incremental ARR per quarter from the same 1,000 contacts.**

---

## Future Upgrades

- LinkedIn DM agent (using PhantomBuster)
- Video personalization (Tavus / HeyGen) appended to email_1
- Multi-mailbox load balancing (rotate across 5+ Gmail accounts)
- Pinecone vector DB for semantic dedup at the company level
- Webhook listener for Calendly bookings → auto-update Deal to "Meeting Held"
- Anthropic Claude as cheaper / safer fallback model
- Voice agent (ElevenLabs + Twilio) for top-tier replies that go silent
- Self-learning: feed reply rate back into Personalization agent's prompt every Sunday

---

## License

MIT — fork it, sell it, ship it. Just don't email people who said no.

