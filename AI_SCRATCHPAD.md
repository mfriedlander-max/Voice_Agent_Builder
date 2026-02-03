# AI Scratchpad - Voice Agent Builder

Session state, change log, decisions, and next steps. Updated after every implementation.

For Retell platform knowledge, see **AI_VOICE_AGENT_KB.md**.

---

## Current State

- **Branch:** `dev`
- **Last session:** 2026-02-02

---

## Change Log

### 2026-02-02: Initial Setup and Discovery
- Set up Retell AI MCP server with SDK pinned to v1.11.0 (Zod v4 fix)
- Confirmed MCP exposes 24 tools but has NO conversation flow tools
- Found REST API endpoints for conversation flow CRUD
- Created Landscaping Receptionist Demo via API (2-step: create flow, then agent)
- Exported Real Estate AI agent JSON as reference

### 2026-02-02 (continued): Research and Testing
- Tested 10 of 13 node types via API (all working except component partial, mcp/bridge/cancel untested)
- Confirmed PATCH behavior: merge for scalars, full replacement for arrays
- Documented custom tool/webhook integration patterns for any calendar, CRM, or API
- Documented HubSpot native integration with Retell
- Wrote comprehensive architecture report (AI_VOICE_AGENT_KB.md sections 10.1-10.22)

### 2026-02-02 (continued): File Reorganization
- Moved all knowledge to AI_VOICE_AGENT_KB.md
- AI_SCRATCHPAD.md now tracks session state and changes only

### 2026-02-02 (continued): GreenScape Landscaping Agent Build
- Designed and built full landscaping service agent with 28 nodes
- Features: lead qualification, vendor rejection ("chopping block"), Google Calendar scheduling (custom webhook tools with placeholder URLs), service area check, spelling confirmation for name/email, FAQ, emergency handler, existing customer path
- Two global nodes: Transfer Call (human handoff/emergency), Frustrated Exit
- Two custom tools: `check_calendar_availability`, `book_estimate_appointment` (placeholder webhook URLs — user to wire up middleware)
- Deployed as new agent: `agent_7bfd12a1dea7f5faca1820e180` / flow: `conversation_flow_f5405d6f0eba`
- Flow JSON saved at `flows/landscaping-service-agent.json`
- Plan document at `.claude/plans/mutable-cooking-lynx.md`

### 2026-02-02 (continued): Integration Brainstorm — Google Calendar & HubSpot

#### Google Calendar Integration Options Evaluated
1. **n8n** — visual workflow builder, self-hosted (free) or cloud. Native Google Calendar nodes. Receives Retell webhook, calls GCal API, returns result. Good for non-technical users but adds a dependency.
2. **Cloudflare Worker** (chosen approach) — serverless JS function on Cloudflare's edge network. ~100 lines of code. Free tier: 100K requests/day. Production-grade, auto-scaling, no server maintenance. Lowest cost and complexity.
3. **Custom Node.js/Python server** — full control, deploy on Railway/Render/Fly.io. More flexible but requires server management.

#### Chosen Architecture
```
Retell Agent → Cloudflare Worker (webhook) → Google Calendar API (via service account) → Response → Agent
```
- **Google Cloud service account** for auth — free, no credit card required
- One service account serves all clients; each client just shares their Google Calendar with the service account email
- Cloudflare Worker stores service account JSON key as a secret
- Multi-client: one Worker handles all clients via route-based client identification

#### HubSpot Integration Options Evaluated
1. **Retell native HubSpot integration** — $0.07/min extra. Best for outbound campaigns triggered by HubSpot workflows. Auto-syncs call data to contact records. Zero code.
2. **Custom webhook via Cloudflare Worker** (recommended for inbound) — $0. Mid-call CRM lookups, create/update leads during or after calls. Same Worker pattern as Google Calendar.

**Decision:** For inbound agents (like landscaping receptionist), custom webhook is the right choice — no per-minute surcharge. Native integration only makes sense for HubSpot-triggered outbound campaigns where the automation value justifies $0.07/min.

#### Cost Summary (per client)
| Service | Cost |
|---------|------|
| Retell | Per-minute call pricing (only real cost) |
| Cloudflare Workers | $0 (free tier) or $5/mo (paid) — one account for all clients |
| Google Calendar API | $0 |
| Google Cloud project + service account | $0 |
| HubSpot integration (custom webhook) | $0 |

#### Client Onboarding Requirements
From the client:
- Share their Google Calendar with service account email (e.g., `retell-calendar@your-project.iam.gserviceaccount.com`)
- Calendar ID (usually their email)
- Business info: company name, services, hours, service area, transfer phone number, email, appointment duration, qualification criteria
- Voice/tone preferences

From us (one-time setup):
- Google Cloud project with Calendar API enabled
- Service account with JSON key
- Cloudflare Workers account
- Retell account

---

## Next Steps / Open Items

- [ ] Set up Google Cloud project + service account + Calendar API
- [ ] Build Cloudflare Worker middleware (check availability + book appointment endpoints)
- [ ] Update Retell flow with real Cloudflare Worker URLs
- [ ] Set service area city (replace `[CITY_PLACEHOLDER]` in global prompt and location node)
- [ ] Set transfer phone number (replace `+10000000000` in transfer node)
- [ ] Test all 4 conversation paths via web call in Retell dashboard
- [ ] Clean up old test agents (Landscaping Receptionist - Test, Landscaping Receptionist Demo)

---

## Active Decisions / Blockers

- Google Cloud service account setup pending (next step)
- Service area city TBD
- Transfer phone number TBD
