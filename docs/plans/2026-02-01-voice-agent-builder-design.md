# Voice Agent Builder -- Design Document

**Date:** 2026-02-01
**Status:** Approved

---

## Purpose

A template library and MCP-powered workflow for building full-stack voice agent solutions for clients. Each client delivery includes a Retell AI voice agent, n8n automation workflows, and integrations with the client's existing tools (CRM, calendar, communication, etc.).

## Target User

The builder (Max). This is a professional tool for delivering client projects faster -- not a SaaS product or end-user application.

## Architecture

```
Builder (prompting in Claude Code)
  |
  +-- Retell AI MCP Server --> create/configure voice agents
  |
  +-- n8n MCP Server (node docs) --> understand n8n nodes and build workflows correctly
  |
  +-- n8n MCP Server (API) --> deploy/manage workflows on n8n instance
  |
  +-- Templates repo (this project)
       +-- System prompt templates (by use case)
       +-- n8n workflow templates (JSON skeletons)
       +-- Client config records (YAML)
```

### What This Project Contains

- **Template library** -- reusable system prompts, n8n workflow skeletons, and node configurations organized by use case
- **Client records** -- YAML files tracking what was deployed for each client (agent IDs, workflow IDs, phone numbers, integration details)
- **Setup scripts** -- shell scripts to install and configure the MCP servers

### What This Project Does NOT Contain

- No application code, no server, no API, no UI
- No custom wrapper around Retell or n8n APIs
- MCP servers handle all service interaction directly

## Template Library Structure

```
templates/
+-- system-prompts/
|   +-- _base.md                    # Common instructions all agents share
|   +-- appointment-booking.md
|   +-- lead-qualification.md
|   +-- customer-support.md
|   +-- outbound-sales.md
|   +-- follow-up-call.md
|
+-- n8n-workflows/
|   +-- appointment-to-calendar.json
|   +-- lead-to-crm.json
|   +-- post-call-summary-email.json
|   +-- sms-confirmation.json
|   +-- webhook-receiver.json
|
+-- client-configs/
    +-- _template.yaml              # Template for new client records
```

### System Prompts

Markdown files with placeholder variables (`{{business_name}}`, `{{business_hours}}`, `{{services}}`). Builder describes client details, Claude fills them in and pushes to Retell via MCP.

### n8n Workflows

JSON skeletons with placeholder nodes. Integration-specific nodes (which CRM, which calendar) are swapped in per client during setup.

### Client Configs

YAML files tracking deployed resources per client. One file per client for auditability.

## Typical Client Setup Workflow

```
1. Builder describes client needs:
   "New client -- Peak Dental, dentist in Austin.
    Voice agent that books appointments and sends confirmations.
    Uses Google Calendar and Twilio for SMS."

2. Claude (via MCP):
   - Selects and fills appointment-booking system prompt template
   - Creates Retell voice agent with that prompt
   - Builds n8n workflow: webhook -> Google Calendar -> Twilio SMS
   - Activates the workflow
   - Saves client config to client-configs/peak-dental.yaml

3. Builder tests the agent, requests tweaks

4. Claude adjusts prompt or workflow as needed via MCP
```

## Infrastructure Requirements

### Retell AI

- Account: exists (API key needed from dashboard)
- MCP Server: official Retell MCP server (45+ tools)
- Cost: per-minute calling on Retell platform

### n8n

- Instance: self-hosted via Docker (local machine or VPS, $5-10/mo)
- MCP Servers (two):
  - `n8n-mcp` (czlonkowski) -- node documentation for 1,084 nodes
  - `mcp-n8n-server` (ahmadsoliman) -- CRUD operations on workflows
- Cost: free (self-hosted) or n8n Cloud pricing

### Claude Code

- MCP server configs added via `claude mcp add`
- No additional tooling required

## Implementation Phases

### Phase 1: Infrastructure Setup

- Set up n8n instance (Docker)
- Configure all three MCP servers (Retell, n8n-mcp, mcp-n8n-server)
- Verify connectivity

### Phase 2: Template Library

- Create base system prompt template
- Create 5 initial use-case prompt templates
- Create n8n workflow skeletons for common patterns
- Create client config YAML template

### Phase 3: First Client Delivery

- Use the system to deliver an actual client project
- Refine templates based on real usage
- Document lessons learned

## Design Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| No custom application | MCP-first | Builder is always in the loop; no unattended automation needed |
| No UI | CLI/conversational | Builder is comfortable prompting; UI adds maintenance burden |
| Templates as files | Git-tracked markdown/JSON/YAML | Simple, versionable, diffable |
| Two n8n MCP servers | Docs + API separation | One for understanding nodes, one for CRUD operations |
| Client configs as YAML | Flat files per client | Easy to read, no database needed |
