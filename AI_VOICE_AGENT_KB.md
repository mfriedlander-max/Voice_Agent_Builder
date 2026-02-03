# AI Voice Agent Knowledge Base

## Last Updated: 2026-02-02

---

## 1. Environment Setup

### Workspace
- **Path:** `/Users/maxfriedlander/code/Voice_Agent_Builder`
- **Branch:** `dev`
- **Node.js:** v25.4.0
- **npm:** 11.7.0

### Retell AI API
- **API Key Location:** `.mcp.json` (gitignored)
- **API Key:** `key_e06b21e747bc7ba5d6ab3e859e2b`
- **Base URL:** `https://api.retellai.com`
- **Auth Header:** `Authorization: Bearer key_e06b21e747bc7ba5d6ab3e859e2b`

### MCP Server (Retell AI)
- **Package:** `@abhaybabbar/retellai-mcp-server` v1.0.2
- **Status:** Working, but with a critical fix applied (see below)
- **Config file:** `.mcp.json` at project root

#### MCP Server Fix (IMPORTANT)
The npm package uses `@modelcontextprotocol/sdk@^1.11.0` which resolves to latest v1.x (currently 1.25.3). The newer SDK versions have a Zod v3 to v4 breaking change that causes `Cannot read properties of undefined (reading '_zod')` when `tools/list` is called. The server shows "connected" but exposes zero tools.

**Fix applied:** We installed the package locally at `.retell-mcp/` with the MCP SDK pinned to v1.11.0:
```
cd .retell-mcp && npm install @abhaybabbar/retellai-mcp-server @modelcontextprotocol/sdk@1.11.0
```

**`.mcp.json` points to the local install, NOT npx:**
```json
{
  "mcpServers": {
    "retellai-mcp-server": {
      "type": "stdio",
      "command": "node",
      "args": [".retell-mcp/node_modules/@abhaybabbar/retellai-mcp-server/build/index.js"],
      "env": {
        "RETELL_API_KEY": "key_e06b21e747bc7ba5d6ab3e859e2b"
      }
    }
  }
}
```

If someone uses `npx -y @abhaybabbar/retellai-mcp-server` directly, it will break. Always use the local install.

### Gitignored Files
- `.mcp.json` (contains API key)
- `.retell-mcp/` (local MCP server install)
- `node_modules/`
- `.env`, `.env.*`

---

## 2. Available Tools and APIs

### MCP Server Tools (24 tools via `mcp__retellai-mcp-server__*`)
These work through Claude Code's MCP integration:

| Tool | Description |
|------|-------------|
| `list_agents` | List all Retell agents |
| `create_agent` | Create a new agent (retell-llm type only) |
| `get_agent` | Get agent by ID |
| `update_agent` | Update agent settings |
| `delete_agent` | Delete an agent |
| `get_agent_versions` | Get all versions of an agent |
| `list_phone_numbers` | List all phone numbers |
| `create_phone_number` | Purchase a new phone number |
| `get_phone_number` | Get phone number details |
| `update_phone_number` | Update phone number config |
| `delete_phone_number` | Delete a phone number |
| `list_voices` | List available voices |
| `get_voice` | Get voice details |
| `create_phone_call` | Create outbound call |
| `create_web_call` | Create web-based call |
| `get_call` | Get call by ID |
| `list_calls` | List all calls |
| `update_call` | Update a call |
| `delete_call` | Delete a call |
| `list_retell_llms` | List all LLMs |
| `create_retell_llm` | Create a new LLM config |
| `get_retell_llm` | Get LLM by ID |
| `update_retell_llm` | Update LLM config |
| `delete_retell_llm` | Delete an LLM |

**LIMITATION:** The MCP server has ZERO conversation flow tools. It cannot create, read, update, or delete conversation flows. It can only create `retell-llm` type agents, not `conversation-flow` type agents.

### Direct REST API (via curl, NOT in MCP)
These must be called directly with curl:

#### Conversation Flow CRUD
```bash
# CREATE a conversation flow
curl -s -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -X POST https://api.retellai.com/create-conversation-flow \
  -d '{ "start_speaker": "agent", "start_node_id": "...", "global_prompt": "...", "nodes": [...], "tools": [...], "model_choice": {...} }'

# GET a conversation flow
curl -s -H "Authorization: Bearer $KEY" \
  https://api.retellai.com/get-conversation-flow/{conversation_flow_id}

# LIST all conversation flows
curl -s -H "Authorization: Bearer $KEY" \
  https://api.retellai.com/list-conversation-flows

# UPDATE a conversation flow
curl -s -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -X PATCH https://api.retellai.com/update-conversation-flow/{conversation_flow_id} \
  -d '{ ...fields to update... }'

# DELETE a conversation flow
curl -s -H "Authorization: Bearer $KEY" \
  -X DELETE https://api.retellai.com/delete-conversation-flow/{conversation_flow_id}
```

#### Creating a Conversation Flow Agent (2-step process)
1. Create the conversation flow via POST (returns `conversation_flow_id`)
2. Create the agent with `response_engine.type: "conversation-flow"`:
```bash
curl -s -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -X POST https://api.retellai.com/create-agent \
  -d '{
    "agent_name": "My Agent",
    "response_engine": {
      "type": "conversation-flow",
      "conversation_flow_id": "conversation_flow_xxxxx"
    },
    "voice_id": "11labs-Cimo",
    "voice_model": "eleven_flash_v2_5",
    "language": "en-US"
  }'
```

---

## 3. Conversation Flow JSON Structure

### Reference File
Full working example exported from Retell dashboard:
`/Users/maxfriedlander/Desktop/Real Estate AI Appointment Setter Personal Project.json`

### Top-Level Flow Fields (required for POST /create-conversation-flow)
```json
{
  "start_speaker": "agent",
  "start_node_id": "start-node-xxx",
  "global_prompt": "...",
  "nodes": [],
  "tools": [],
  "model_choice": {
    "type": "cascading",
    "model": "gpt-4.1"
  },
  "flex_mode": true
}
```

- `start_speaker` — REQUIRED: "agent" or "user"
- `start_node_id` — REQUIRED: ID of the first node
- `global_prompt` — System prompt for the entire flow
- `nodes` — REQUIRED: array of node objects
- `tools` — Tools available to function nodes
- `model_choice` — LLM model config
- `flex_mode` — Allows flexible conversation within nodes

### Node Types (11 total from Retell dashboard)

#### 1. Conversation Node
The most common node. Agent speaks a prompt, listens for response, transitions based on conditions.
```json
{
  "id": "node-unique-id",
  "name": "Human Readable Name",
  "type": "conversation",
  "instruction": {
    "type": "prompt",
    "text": "What the agent should say/do in this node"
  },
  "edges": [
    {
      "id": "edge-unique-id",
      "condition": "Human readable condition",
      "transition_condition": {
        "type": "prompt",
        "prompt": "LLM evaluates this to decide if transition fires"
      },
      "destination_node_id": "node-target-id"
    }
  ],
  "display_position": { "x": 550, "y": 80 },
  "interruption_sensitivity": 0,
  "start_speaker": "agent",
  "skip_response_edge": {
    "condition": "Skip response",
    "id": "edge-skip-id",
    "transition_condition": {
      "type": "prompt",
      "prompt": "Skip response"
    },
    "destination_node_id": "node-next-id"
  },
  "global_node_setting": {
    "condition": "user is annoyed and want to hangup"
  },
  "finetune_conversation_examples": [],
  "finetune_transition_examples": [
    {
      "id": "fe-xxx",
      "transcript": [
        { "content": "agent says...", "role": "agent" },
        { "content": "user says...", "role": "user" }
      ]
    }
  ]
}
```

Field notes:
- `interruption_sensitivity` — Optional: 0 means no interruption allowed
- `start_speaker` — Only needed on the start node
- `skip_response_edge` — Optional: skip agent response and go directly to next node
- `global_node_setting` — Optional: makes this node reachable from anywhere in the flow when condition matches
- `finetune_conversation_examples` — Optional: few-shot examples for how the agent should talk
- `finetune_transition_examples` — Optional: few-shot examples for edge transition decisions

#### 2. Function Node
Calls an external tool (Cal.com, webhook, etc.) and transitions based on result.
```json
{
  "id": "node-unique-id",
  "name": "Check Calendar",
  "type": "function",
  "tool_id": "tool-xxx",
  "tool_type": "local",
  "speak_during_execution": true,
  "wait_for_result": true,
  "instruction": {
    "type": "prompt",
    "text": "What to say while function executes"
  },
  "edges": [
    {
      "id": "edge-xxx",
      "condition": "Success scenario",
      "transition_condition": { "type": "prompt", "prompt": "Success scenario" },
      "destination_node_id": "node-success"
    },
    {
      "id": "edge-yyy",
      "condition": "Failure scenario",
      "transition_condition": { "type": "prompt", "prompt": "Failure scenario" },
      "destination_node_id": "node-failure"
    }
  ],
  "display_position": { "x": 3850, "y": 80 }
}
```

Field notes:
- `tool_id` — References a tool in the flow's top-level `tools[]` array
- `tool_type` — "local" for built-in tools
- `speak_during_execution` — If true, agent talks while waiting for tool result
- `wait_for_result` — If true, waits for tool to return before transitioning

#### 3. Call Transfer Node
Transfers the call to a phone number.
```json
{
  "id": "node-unique-id",
  "name": "Transfer Call",
  "type": "transfer_call",
  "transfer_destination": {
    "type": "predefined",
    "number": "+12134447777"
  },
  "transfer_option": {
    "type": "cold_transfer",
    "show_transferee_as_caller": false,
    "enable_bridge_audio_cue": true
  },
  "instruction": {
    "type": "prompt",
    "text": "Transferring your call now."
  },
  "edge": {
    "condition": "Transfer failed",
    "id": "edge-xxx",
    "transition_condition": { "type": "prompt", "prompt": "Transfer failed" },
    "destination_node_id": "node-fallback"
  },
  "global_node_setting": {
    "condition": "Whenever the user wants to talk to a human agent"
  },
  "display_position": { "x": 2200, "y": 1800 }
}
```

Field notes:
- Uses singular `edge` (not `edges` array) for the failure case only
- `transfer_option.type` — "cold_transfer" or "warm_transfer"
- `global_node_setting` — Makes this node accessible from anywhere in the flow

#### 4. End Node
Terminates the call.
```json
{
  "id": "node-unique-id",
  "name": "End Call",
  "type": "end",
  "instruction": {
    "type": "prompt",
    "text": "Thank you, goodbye!"
  },
  "display_position": { "x": 4950, "y": 0 }
}
```

#### 5-11. Other Node Types (from dashboard, JSON structure not yet confirmed via API)
- **Press Digit** — DTMF tone input (e.g., "press 1 for sales")
- **Logic Split** — Conditional branching without conversation (evaluate variables/conditions silently)
- **Agent Transfer** — Hand off to another Retell agent mid-call (different from Call Transfer which goes to a phone number)
- **SMS** — Send a text message during the call
- **Extract Variable** — Pull data from conversation into `{{dynamic_variables}}`
- **MCP Node** — Call external MCP tools mid-conversation for real-time data
- **Components** — Reusable groups of nodes (like sub-flows)

To discover their JSON structure: export an agent from the Retell dashboard that uses these node types and inspect the JSON.

### Flow Tools (referenced by Function Nodes)
Tools are defined at the flow level in the `tools[]` array and referenced by `tool_id` in function nodes.

#### Cal.com Check Availability
```json
{
  "tool_id": "tool-xxx",
  "type": "check_availability_cal",
  "name": "check_calendar_availability",
  "description": "Check calendar for available slots",
  "event_type_id": 668000,
  "cal_api_key": "cal_live_xxxxx",
  "timezone": "America/Los_Angeles"
}
```

#### Cal.com Book Appointment
```json
{
  "tool_id": "tool-yyy",
  "type": "book_appointment_cal",
  "name": "book_appointment",
  "description": "Book an appointment on the calendar",
  "event_type_id": 668000,
  "cal_api_key": "cal_live_xxxxx",
  "timezone": "America/Los_Angeles"
}
```

#### Other tool types (from MCP server schema, likely work in flows too)
- `end_call` — End the call programmatically
- `transfer_call` — Transfer call (as a tool rather than a node)
- `press_digit` — Send DTMF tone
- `custom` — Custom webhook tool with `url`, `speak_after_execution`, `speak_during_execution`

### Dynamic Variables
Use `{{variable_name}}` in node prompts. These can come from:
- CRM/webhook data passed when creating the call via `retellLlmDynamicVariables`
- Extract Variable nodes during the call
- MCP Node results

Example: `"Say Hello to {{customer_name}}"`

### Edge Rules
- Each edge has a `condition` (human readable) and `transition_condition.prompt` (LLM evaluates this)
- **IMPORTANT:** Edge `destination_node_id` CANNOT be the same as the source node ID. The API will reject with: `Edge destination node id cannot be the same as the source node id`. Use a separate intermediate node instead of looping back to self.
- Edges are evaluated in order; first matching condition wins
- `skip_response_edge` allows a node to immediately transition without the agent speaking
- `global_node_setting.condition` makes a node reachable from anywhere in the flow

---

## 4. Existing Agents in Account

| Agent Name | Type | Agent ID | Notes |
|------------|------|----------|-------|
| Real Estate AI Appointment Setter Personal Project | conversation-flow | `agent_d1c6f41055a6b282b230a4ff26` | Main reference agent, has Cal.com integration |
| Lead Qualification School Fun | retell-llm | `agent_64d953749523b03e571341344c` | Single prompt agent |
| Landscaping Receptionist - Test | retell-llm | `agent_89fb9ef38e0c32af71c9908e63` | Test agent, can delete |
| Landscaping Receptionist Demo | conversation-flow | `agent_a7d74c454f6778f3b36369b438` | First API-created conversation flow, demo |
| GreenScape Landscaping Agent | conversation-flow | `agent_7bfd12a1dea7f5faca1820e180` | Full agent: lead qual, vendor rejection, GCal scheduling (placeholder URLs), 28 nodes |

---

## 5. How to Build a New Conversation Flow Agent (Step by Step)

### Step 1: Gather Requirements
You need all of the following before building:

1. **Company name**
2. **Agent persona** — name, tone, personality
3. **Business hours** — days and times
4. **Services offered** — complete list
5. **Service area** — geographic coverage
6. **Call flow** — step-by-step node sequence (greeting, identify need, collect info, action, end)
7. **Info to collect** — name, phone, email, address, etc.
8. **Scheduling method** — Cal.com (need API key + event type ID) or just collect info for callback
9. **Transfer number** — phone number for human handoff
10. **Pricing info** — what to say about pricing
11. **FAQs** — common questions and scripted answers
12. **Dynamic variables** — CRM fields like `{{customer_name}}`
13. **Voice preference** — voice ID (default: `11labs-Cimo`, model: `eleven_flash_v2_5`)
14. **LLM Model** — default: `gpt-4.1` cascading

### Step 2: Build the JSON
Construct the conversation flow JSON with nodes, edges, and tools following the structure in Section 3.

Key rules:
- Every node needs a unique `id`
- Every edge needs a unique `id`
- `start_node_id` must match exactly one node's `id`
- Edge destinations cannot point back to the same node
- Use `display_position` for dashboard layout (increment x by ~550 per column)
- Function nodes reference tools by `tool_id`

### Step 3: Create via API
```bash
# 1. Create the conversation flow
curl -s -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -X POST https://api.retellai.com/create-conversation-flow \
  -d @flow-payload.json

# Extract the conversation_flow_id from response

# 2. Create the agent attached to the flow
curl -s -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -X POST https://api.retellai.com/create-agent \
  -d '{
    "agent_name": "My Agent",
    "response_engine": {
      "type": "conversation-flow",
      "conversation_flow_id": "conversation_flow_xxxxx"
    },
    "voice_id": "11labs-Cimo",
    "voice_model": "eleven_flash_v2_5",
    "language": "en-US"
  }'
```

### Step 4: Test
- Test in Retell dashboard via web call
- Or create a web call via MCP: `mcp__retellai-mcp-server__create_web_call` with the agent ID
- Review the flow visually in the dashboard to confirm node layout

---

## 6. Known Issues and Gotchas

1. **MCP SDK Zod v4 break** — Do NOT use `npx -y @abhaybabbar/retellai-mcp-server`. Use the local install at `.retell-mcp/` with SDK pinned to v1.11.0. See Section 1 for details.

2. **Edge self-reference** — The API rejects flows where an edge's `destination_node_id` equals the source node's `id`. Error message: `Edge destination node id cannot be the same as the source node id`. Use a separate intermediate node instead of looping.

3. **MCP has no conversation flow tools** — All conversation flow operations (create, read, update, delete, list) must use direct curl to the REST API. The MCP server only handles agents, LLMs, calls, phone numbers, and voices.

4. **VS Code restart required** — After changing `.mcp.json`, you must fully restart VS Code (not just reload window) for MCP tools to load into the session.

5. **Node.js v25 compatibility** — The MCP SDK v1.25.3 has issues on Node.js v25. The pinned v1.11.0 works fine.

6. **Retell API versions** — Some endpoints use `/v2/` prefix, some don't. Conversation flow endpoints do NOT use `/v2/`. Agent endpoints work without `/v2/` (e.g., `/list-agents`, `/create-agent`).

7. **Transfer node uses singular `edge`** — The `transfer_call` node type uses `edge` (singular object) not `edges` (array) for the failure fallback.

---

## 7. File Structure

```
Voice_Agent_Builder/
├── .claude/
│   └── settings.local.json        # Claude Code permissions and MCP config
├── .retell-mcp/                    # Local MCP server install (gitignored)
│   ├── node_modules/
│   │   ├── @abhaybabbar/retellai-mcp-server/  # v1.0.2
│   │   └── @modelcontextprotocol/sdk/          # PINNED to v1.11.0
│   └── package.json
├── .mcp.json                       # MCP server config with API key (gitignored)
├── .gitignore
├── AI_SCRATCHPAD.md                # This file
├── CLAUDE.md                       # Claude Code session instructions
└── docs/
    └── plans/
        └── 2026-02-01-voice-agent-builder-design.md
```

Desktop reference files:
- `/Users/maxfriedlander/Desktop/Real Estate AI Appointment Setter Personal Project.json` — Full exported conversation flow with all node types demonstrated
- `/Users/maxfriedlander/Desktop/Landscaping Receptionist Demo.json` — Generated demo flow JSON

---

## 8. Session Log

### 2026-02-02: Initial Setup and Discovery
- Set up Retell AI MCP server in `.mcp.json`
- Discovered Zod v4 incompatibility, fixed by pinning SDK to v1.11.0
- Confirmed MCP server exposes 24 tools (agents, calls, phones, voices, LLMs)
- Discovered MCP server has NO conversation flow tools
- Found undocumented REST API endpoints for conversation flow CRUD
- Successfully created conversation flow agent entirely via API (2-step: create flow, then create agent)
- Exported Real Estate agent JSON as reference template
- Built and deployed Landscaping Receptionist Demo as proof of concept
- Identified 11 node types available in dashboard (4 confirmed via API, 7 untested)

---

## 9. Next Steps / Open Items

- [x] Test remaining node types via API — **All confirmed working** (see 10.19)
- [ ] Build reusable templates for common agent types
- [ ] Explore Retell API docs for any additional undocumented endpoints
- [ ] Clean up test agents (Landscaping Receptionist - Test, Landscaping Receptionist Demo)
- [x] Investigate if conversation flow update API — **Confirmed: partial field + full nodes replacement** (see 10.20)
- [x] Research custom tool/webhook integration patterns — **Complete** (see 10.21)

---

## 10. Conversation Flow Architecture Report

### Last Updated: 2026-02-02

This section is a self-contained knowledge transfer document. Any LLM or human reading this should be able to design and build Retell conversation flow agents from scratch without prior context.

---

### 10.1 Retell Conversation Flow — How It Works

A conversation flow is a **directed graph** of nodes connected by conditional edges. The LLM evaluates edge conditions after each user turn (or after tool execution) to decide which node to visit next. The agent can have multi-turn conversations *within* a single node before transitioning.

```
[Begin] → [Welcome] → [Qualify Intent]
                           ├─→ [Collect Info] → [Function: Check Calendar] → [Present Slots] → [Function: Book] → [Confirm] → [Goodbye] → [End]
                           ├─→ [Answer Question] → [Goodbye] → [End]
                           └─→ [Transfer Call] (global node, reachable from anywhere)
```

**Key mental model:** Each node is a *task* the agent must complete. The agent stays in that node, conversing freely, until an edge condition fires. Edges are evaluated after the user speaks (or after `skip_response_edge` fires).

#### Core Architecture Rules

| Rule | Detail |
|------|--------|
| Graph is directed | Edges go from source → destination; no implicit back-edges |
| No self-loops | API rejects `destination_node_id == source node id` |
| Multi-turn within node | Agent can have extended conversation inside one node — don't over-split |
| Edge evaluation order | First matching edge wins (order matters) |
| Global nodes | Reachable from *any* node when `global_node_setting.condition` matches |
| Flex mode | Compiles all nodes into one prompt; agent navigates dynamically (limit: ~20 nodes) |

---

### 10.2 Complete Node Type Reference (from API schema)

All 13 node types confirmed via the Create Conversation Flow API schema:

| # | Type (JSON `type` value) | Purpose | Edge Pattern | Key Fields |
|---|--------------------------|---------|--------------|------------|
| 1 | `conversation` | Multi-turn dialogue | `edges[]` array | `instruction`, `finetune_conversation_examples`, `finetune_transition_examples`, `skip_response_edge`, `global_node_setting`, `interruption_sensitivity`, `model_choice`, `knowledge_base_ids` |
| 2 | `function` | Execute tool (Cal.com, webhook) | `edges[]` array | `tool_id`, `tool_type` ("local"\|"shared"), `speak_during_execution`, `wait_for_result` |
| 3 | `transfer_call` | Transfer to phone number | singular `edge` (failure only) | `transfer_destination`, `transfer_option` (cold\|warm\|agentic_warm), `global_node_setting` |
| 4 | `end` | Terminate call | none | `instruction` (optional farewell) |
| 5 | `press_digit` | Send DTMF tone | `edges[]` array | `instruction`, `delay_ms` |
| 6 | `branch` | Silent conditional routing | `edges[]` + required `else_edge` | No instruction — pure logic split |
| 7 | `sms` | Send text message | `success_edge` + `failed_edge` (singular) | `instruction` |
| 8 | `extract_dynamic_variables` | Pull data into `{{variables}}` | `edges[]` array | `variables[]` (AnalysisData: string\|enum\|boolean\|number) |
| 9 | `agent_swap` | Transfer to another Retell agent | singular `edge` | `agent_id`, `agent_version`, `post_call_analysis_setting`, `webhook_setting` |
| 10 | `mcp` | Call external MCP tool | `edges[]` array | `mcp_id`, `mcp_tool_name`, `wait_for_result`, `response_variables` |
| 11 | `component` | Reusable sub-flow | `edges[]` + required `else_edge` | `component_id`, `component_type` ("local"\|"shared") |
| 12 | `bridge_transfer` | Bridge transfer (warm transfer internal) | none | — |
| 13 | `cancel_transfer` | Cancel ongoing transfer | none | — |

#### Edge Type Reference

| Edge Type | Used By | `transition_condition.prompt` Must Be |
|-----------|---------|---------------------------------------|
| Standard `NodeEdge` | conversation, function, press_digit, branch, extract_dv, mcp, component | Any descriptive prompt |
| `SkipResponseEdge` | conversation | Exactly `"Skip response"` |
| `TransferFailedEdge` | transfer_call, agent_swap | Exactly `"Transfer failed"` |
| `ElseEdge` | branch, component | Exactly `"Else"` |
| `SmsSuccessEdge` | sms | Exactly `"Sent successfully"` |
| `SmsFailedEdge` | sms | Exactly `"Failed to send"` |

#### Transition Condition Types

1. **Prompt-based** (most common): `{ "type": "prompt", "prompt": "descriptive condition" }` — LLM evaluates
2. **Equation-based** (for branch nodes): `{ "type": "equation", "equations": [...], "operator": "||"|"&&" }`
   - Equation operators: `==`, `!=`, `>`, `>=`, `<`, `<=`, `contains`, `not_contains`, `exists`, `not_exist`
   - **Warning:** Equation edges are unreliable in Flex Mode — use prompt edges instead

#### Instruction Types

1. **Prompt** (dynamic): `{ "type": "prompt", "text": "..." }` — agent generates response based on prompt
2. **Static text** (fixed): `{ "type": "static_text", "text": "..." }` — agent says exactly this first, then generates dynamically if conversation continues

---

### 10.3 Analysis of the Real Estate AI Flow

The Real Estate agent is an outbound appointment setter with 15 nodes. Here is its architecture analyzed:

#### Flow Topology

```
Welcome → [free?] → Ask For Meeting → [yes?] → Ask Name → [simple?] → Ask Email → Confirm Email → Check Calendar → Tell Slots → Book → Result → Goodbye → End
                                                    └─[complex?]→ Spell Name → Ask Email ...
                                    └─[no?] → Offer Transfer → [yes?] → Transfer Call
                                                              └─[no?] → Polite Goodbye → End
           └─[busy?] → Ask Callback Time → Confirm Callback → Goodbye → End

Global nodes:
  - Transfer Call: reachable when "user wants to talk to a human agent"
  - Polite Goodbye: reachable when "user is annoyed and wants to hangup"
```

#### What This Flow Does Well

**1. Spelling Confirmation Pattern (high value)**
The flow handles name/email accuracy by splitting into a conditional branch:
- IF name is simple/common → skip to next step
- IF name is uncommon → spell it character by character, confirm, loop if wrong

This prevents the #1 problem in voice AI: garbled names/emails from STT errors.

**Example from the flow (email confirmation node):**
```
"Spell out their email address to double confirm.
If user indicates the email is incorrect, ask for email again and spell it out to re-confirm.

## How to spell out
The possible email format is name@company.com
to spell out a email address is n-a-m-e-@-c-o-m-p-a-n-y-dot-com,
@ is pronounced by 'at'."
```

**Why this works:** The instruction includes a concrete example of *how* to spell, not just *that* to spell. The LLM needs this level of specificity for voice output.

**2. Global Escape Nodes**
Two global nodes handle non-happy-path scenarios from anywhere:
- `Transfer Call` with `global_node_setting.condition: "Whenever the user wants to talk to a human agent"` — user can always request a human
- `Polite Goodbye` with `global_node_setting.condition: "user is annoyed and want to hangup"` — graceful exit when user is frustrated

**Why this works:** Without global nodes, a frustrated user would be trapped in a rigid flow with no escape except hanging up.

**3. Tool Failure Recovery**
Every function node has both success AND failure edges:
- Check Calendar: success → show slots, failure → ask for different time
- Book Appointment: success → confirm result, failure → ask for different time

The failure paths loop back to retry with new input, not to dead ends.

**4. Transfer Failure Recovery**
When call transfer fails, the flow doesn't just apologize — it asks for a callback time, creating a productive fallback.

**5. Finetune Transition Examples**
Used in ambiguous scenarios where the LLM might mis-classify:
- Email spelling confirmation: example where user says "No it's Jockie@gmail.com" to help LLM understand correction vs. confirmation
- Time slot selection: example where user says "Can I do 9:41am" (invalid off-hour time) to help LLM not accept it

#### What This Flow Gets Wrong / Could Improve

**1. Unnamed nodes**
Several nodes are named "Conversation" (generic default). This makes debugging and dashboard navigation hard. Every node should have a descriptive name.

**2. No validation of collected data**
Name, email, and time are collected but never validated beyond spelling. An `extract_dynamic_variables` node after each collection point would store the data reliably and enable later confirmation.

**3. Missing edge on some goodbye nodes**
The "Polite Goodbye" node has an edge with `condition: "Describe the transition condition"` (placeholder text that was never filled in). It relies on `skip_response_edge` to reach the End node. This works but is fragile.

**4. No re-engagement after "not interested"**
When the user says they don't want a meeting and don't want a transfer, the flow goes straight to goodbye. No attempt to understand *why* or offer alternatives.

**5. Single model for all nodes**
The entire flow uses `gpt-4.1` cascading. Simpler nodes (goodbye, transfer) could use `gpt-4.1-mini` or `gpt-4.1-nano` to reduce cost.

---

### 10.4 Tool Organization & Usage Patterns

#### Tool Inventory from the API Schema

| Tool Type | JSON `type` Value | Purpose | Key Config |
|-----------|-------------------|---------|------------|
| Custom Webhook | `custom` | Call any HTTP endpoint | `url`, `method`, `parameters`, `headers`, `timeout_ms`, `response_variables` |
| Cal.com Check Availability | `check_availability_cal` | Query calendar slots | `cal_api_key`, `event_type_id`, `timezone` |
| Cal.com Book Appointment | `book_appointment_cal` | Book calendar slot | `cal_api_key`, `event_type_id`, `timezone` |

Tools are defined at the **flow level** in `tools[]` and referenced by `tool_id` from function nodes.

#### Tool Naming Convention (derived from reference flows)

```
Pattern: verb_noun_context
Examples:
  check_calendar_availability
  book_appointment
  lookup_customer_record
  send_confirmation_sms
  qualify_lead_score
```

#### Tool → Node Interaction Pattern

The canonical pattern for using a tool in a conversation flow is a **3-node sandwich**:

```
[Collect Inputs] → [Function: Execute Tool] → [Present Results]
     (conversation)     (function)                (conversation)
```

**Collect Inputs node:** Gathers all required parameters through conversation. One node per parameter for reliability, or one node for all if parameters are simple.

**Function node:** Calls the tool. Key settings:
- `speak_during_execution: true` — say "Let me check that for you..." while waiting
- `wait_for_result: true` — don't transition until tool returns

**Present Results node:** Interprets tool response for the user. Handles both success and failure.

#### Tool Failure Handling Strategy

```
IF tool succeeds → present results, continue flow
IF tool fails → apologize, ask for corrected input, retry tool
IF tool fails again → offer human handoff or callback
```

Every function node MUST have at least two edges: success and failure. The failure edge should loop to a re-collection or alternative path, never to a dead end.

#### Tool Guardrails

| Guardrail | Implementation |
|-----------|---------------|
| Never hallucinate tool results | Function nodes wait for actual results (`wait_for_result: true`) |
| Validate inputs before calling | Use conversation nodes to collect and confirm data before the function node |
| Confirm critical actions | Add a confirmation conversation node between data collection and booking/purchasing |
| Handle missing fields | Node instructions explicitly list required data; don't proceed without it |
| Timeout handling | Custom tools have `timeout_ms` (1000-600000); function node failure edge catches timeouts |

---

### 10.5 System Prompt Design Patterns

#### Global Prompt Structure (template)

```
You are [ROLE_NAME], a voice AI [ROLE_TYPE] for [COMPANY_NAME].
[CONTEXT: why this call is happening]

User input is speech-to-text transcription, it may contain errors. Do your best to understand.

Try your best to only ask one question at a time.

[GUARDRAILS]
- Never make up information you don't have
- If unsure, offer to transfer to a human or take a message
- Keep responses concise and conversational
- Do not read URLs, long numbers, or technical details aloud

[BUSINESS CONTEXT — only if needed globally]
```

**Key principles:**
- First sentence establishes identity and role
- Always include the STT error warning (critical for voice)
- "One question at a time" prevents overwhelming callers
- Guardrails go in global prompt so they apply to every node

#### Node-Level Prompt Patterns

**Collection node:**
```
Ask the caller for [SPECIFIC_DATA].
[VALIDATION_RULES if any]
[EXAMPLES of what to accept/reject]
```

**Confirmation node:**
```
Repeat back [DATA_ITEMS] and ask the caller to confirm everything is correct.
If anything is wrong, correct it.
```

**Presentation node (after tool):**
```
Tell the user [RESULT]. Use [FORMAT_RULES].
[ORAL_FORMAT_INSTRUCTIONS]

Remember to use the oral format:
such as "Monday, January 5th, five pm". Don't say "5 o'clock", use "5pm".
```

**Why oral format instructions matter:** The LLM generates text that is spoken aloud. Without explicit instructions, it produces text-optimized output (dates as "2026-02-03", times as "17:00") that sounds robotic when spoken.

---

### 10.6 Reusable Node Pattern Templates

#### Pattern 1: Data Collection with Spelling Confirmation

```
[Ask for Data] → [simple?] → [Next Step]
                └─[complex?]→ [Spell & Confirm] → [correct?] → [Next Step]
                                                 └─[wrong?] → [Ask for Data again]
```

Use when: collecting names, emails, addresses, anything STT frequently garbles.

JSON template for the confirmation node:
```json
{
  "type": "conversation",
  "instruction": {
    "type": "prompt",
    "text": "Spell out [DATA] character by character to confirm.\nIf user says it's wrong, ask for [DATA] again and re-spell.\n\n## How to spell\n[CONCRETE_EXAMPLE]"
  },
  "edges": [
    { "condition": "User confirms spelled data is correct", "destination_node_id": "next-step" },
    { "condition": "User says data is wrong", "destination_node_id": "ask-data-again" }
  ]
}
```

#### Pattern 2: Tool Execution with Retry

```
[Collect Params] → [Function: Call Tool] → [success?] → [Present Results]
                                          └─[failure?] → [Ask Different Params] → [Function: Call Tool]
```

Use when: any tool call that can fail (calendar check, booking, API lookup).

#### Pattern 3: Qualify Intent → Route

```
[Welcome] → [wants scheduling?] → [Scheduling Sub-flow]
           └─[has question?] → [FAQ Sub-flow]
           └─[wants human?] → [Transfer Call] (global)
```

Use when: inbound calls where caller intent is unknown.

#### Pattern 4: Graceful Exit (Goodbye → End)

```
[Goodbye Node] ──skip_response_edge──→ [End Node]
```

The goodbye node says farewell, then immediately transitions to End without waiting for user response. This prevents awkward silence after goodbye.

```json
{
  "type": "conversation",
  "instruction": { "type": "prompt", "text": "Say: Thank you for calling [COMPANY]! Have a wonderful day." },
  "edges": [],
  "skip_response_edge": {
    "condition": "Skip response",
    "transition_condition": { "type": "prompt", "prompt": "Skip response" },
    "destination_node_id": "node-end"
  }
}
```

#### Pattern 5: Global Safety Nets

Every flow should include these two global nodes:

1. **Human Handoff** — `global_node_setting.condition: "user wants to speak with a human/agent/representative"`
2. **Frustrated Exit** — `global_node_setting.condition: "user is frustrated, annoyed, or wants to end the call"`

---

### 10.7 Flex Mode vs. Standard Mode Decision Guide

| Criterion | Standard Mode | Flex Mode |
|-----------|---------------|-----------|
| Node count | Any | ≤20 nodes (performance degrades beyond) |
| Predictability | High — follows exact graph | Lower — LLM navigates dynamically |
| User deviates from flow | Needs global nodes for escape hatches | Handles naturally |
| Multi-task in one turn | Poor — one edge fires per turn | Good — can skip/combine steps |
| Hallucination risk | Low | Higher with many nodes |
| Knowledge base | Works per-node and globally | Only works at agent level |
| Static text instructions | Reliable | Unreliable — LLM may ignore |
| Equation edges | Work | Unreliable — use prompt edges |
| Cost optimization per-node | Yes (different models per node) | No (single model) |
| Best for | Rigid scripts, compliance, IVR-like flows | Flexible conversations, support agents |

**Heuristic:**
- IF flow is linear and predictable (scheduling, intake, surveys) → Standard Mode
- IF flow is branchy and users frequently deviate → Flex Mode
- IF >20 nodes → Standard Mode (or split into Components for Flex)

---

### 10.8 Component Design Guide

Components are reusable sub-flows. They package a group of nodes into a single insertable unit.

**When to use components:**
- Same sequence appears in multiple agents (identity verification, address collection, payment capture)
- Flow canvas is too cluttered (hide complexity inside a component)
- Want centralized updates across agents (shared library components)

**Rules:**
- Components cannot nest other components
- Tools used inside a component must be defined *within* the component (not visible at agent level)
- The main flow's global prompt applies to nodes inside the component
- **Exit node is mandatory** — without it, the conversation gets stuck inside the component permanently
- Shared components snapshot as local copies when agent is published (prevents breaking production)

**Dynamic variables** are the interface between components and the main flow:
```
Main Flow → sets {{customer_name}} → Component uses {{customer_name}} in prompts
Component → sets {{verified_email}} → Main Flow reads {{verified_email}} after component exits
```

---

### 10.9 Debugging Checklist

When an agent misbehaves:

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| Agent doesn't follow node instructions | Instructions too complex for one node | Split into multiple focused nodes |
| Agent stays in wrong node | Transition conditions too narrow or ambiguous | Broaden conditions; add finetune_transition_examples |
| Agent transitions too early | Transition condition too broad | Make condition more specific; add negative finetune examples |
| Agent says something unexpected | No conversation finetune examples | Add finetune_conversation_examples showing desired behavior |
| Agent can't handle off-script input | No global nodes for escape hatches | Add global nodes for human handoff and frustrated exit |
| Tool call fails silently | No failure edge on function node | Add failure edge to every function node |
| Agent loops between two nodes | Circular edges without exit condition | Add an "else" path or limit via edge conditions |
| User gets stuck in component | No exit node in component | Add exit node connecting back to main flow |

---

### 10.10 Comparison: Current Approach vs. Recommended Approach

#### What We Did (Landscaping Demo)

| Aspect | Current | Issue |
|--------|---------|-------|
| Data collection | One node per field, no spelling confirmation | Names/emails will be garbled by STT |
| Tool usage | No tools — collect-and-callback only | No automated scheduling |
| Failure handling | No failure paths | If user gives bad data, no recovery |
| Global nodes | None | User can't request human or exit gracefully |
| Finetune examples | None | LLM may misinterpret edge conditions |
| Per-node models | Single model everywhere | Paying premium for simple goodbye nodes |
| Dynamic variables | None used | No CRM integration possible |
| Components | Not used | Each agent built from scratch |

#### Recommended Improvements

1. **Add spelling confirmation** for name, email, and address nodes (Pattern 1 from 10.6)
2. **Add 2 global nodes** to every flow: human handoff + frustrated exit (Pattern 5 from 10.6)
3. **Add finetune examples** to any transition that involves nuanced interpretation (e.g., "user confirms" vs "user corrects")
4. **Use `gpt-4.1-nano`** for simple nodes (goodbye, transfer announcement) to cut cost
5. **Use `extract_dynamic_variables` nodes** after collecting name/email/phone to store them reliably as `{{variables}}`
6. **Build reusable components** for: identity collection, address collection, FAQ handling, scheduling
7. **Use Cal.com integration** for scheduling instead of collect-and-callback

---

### 10.11 Heuristic Rules (IF → THEN)

These are deterministic rules for building agents:

```
IF collecting name or email
  THEN add spelling confirmation sub-flow (Pattern 1)

IF using a function node
  THEN always add both success AND failure edges

IF flow has >3 nodes
  THEN add global node for human handoff

IF flow is for inbound calls
  THEN add global node for frustrated exit

IF node instruction is >5 sentences
  THEN split into multiple nodes

IF transition condition is ambiguous
  THEN add finetune_transition_examples (≥2 examples)

IF agent says goodbye
  THEN use skip_response_edge to End node (Pattern 4)

IF flow will be reused across agents
  THEN package as a shared component

IF flow has >20 nodes AND using Flex Mode
  THEN split into components OR switch to Standard Mode

IF node only does routing (no conversation)
  THEN use branch node instead of conversation node

IF collecting structured data (phone, date, amount)
  THEN use extract_dynamic_variables node after collection

IF tool can fail (network, invalid input, no results)
  THEN failure edge must offer retry with corrected input OR human fallback

IF user provides input via STT
  THEN global prompt must include: "User input is speech-to-text transcription, it may contain errors."

IF agent speaks dates, times, or numbers
  THEN node instruction must include oral format rules (e.g., "January fifth" not "01/05")
```

---

### 10.12 Do / Don't Lists

#### DO

- Name every node descriptively (not "Conversation")
- Include STT error handling in global prompt
- Add `interruption_sensitivity: 0` on greeting nodes (let agent finish intro)
- Use `skip_response_edge` for goodbye → end transitions
- Test every failure path, not just the happy path
- Use `finetune_transition_examples` for ambiguous transitions
- Define tools at flow level, reference by `tool_id` in function nodes
- Use equation conditions in branch nodes for variable checks
- Use prompt conditions in conversation nodes for intent classification
- Include oral format instructions for any node that speaks dates/times/numbers

#### DON'T

- Don't create self-referencing edges (API will reject them)
- Don't use equation edges in Flex Mode (unreliable)
- Don't put >20 nodes in a single Flex Mode flow
- Don't skip the failure edge on function nodes
- Don't use `static_text` instructions in Flex Mode (LLM may ignore)
- Don't nest components inside components (not supported)
- Don't define tools inside a component if they need to be visible at agent level
- Don't use `npx` for the MCP server (Zod v4 break — use local install)
- Don't leave placeholder text in edge conditions (e.g., "Describe the transition condition")
- Don't collect multiple data points in a single node if accuracy matters

---

### 10.13 Agent Build Checklist

Use this checklist for every new agent:

#### Requirements Gathering
- [ ] Company name and business context
- [ ] Agent persona (name, tone, role)
- [ ] Business hours and service area
- [ ] Services/products offered
- [ ] Data to collect (name, phone, email, address, etc.)
- [ ] Scheduling method (Cal.com, webhook, collect-for-callback)
- [ ] Human handoff phone number
- [ ] FAQs and scripted answers
- [ ] Dynamic variables from CRM (if outbound)

#### Flow Design
- [ ] Draw node topology (start → intent → collect → action → confirm → end)
- [ ] Include spelling confirmation for name/email
- [ ] Add global node: human handoff
- [ ] Add global node: frustrated exit
- [ ] Add failure edges to every function node
- [ ] Add finetune examples to ambiguous transitions
- [ ] Use skip_response_edge for goodbye → end
- [ ] Select per-node models (nano for simple, standard for complex)

#### JSON Construction
- [ ] Every node has unique `id` and descriptive `name`
- [ ] Every edge has unique `id`
- [ ] `start_node_id` matches first node's `id`
- [ ] No self-referencing edges
- [ ] Tools defined at flow level with unique `tool_id`
- [ ] Function nodes reference correct `tool_id`
- [ ] `display_position` set for dashboard readability (x += 550 per column)
- [ ] Global prompt includes STT error warning
- [ ] Node prompts include oral format instructions where needed

#### Deployment
- [ ] Create flow via POST /create-conversation-flow
- [ ] Create agent via POST /create-agent with conversation_flow_id
- [ ] Test via web call in dashboard
- [ ] Walk through every edge path (happy + failure + global nodes)
- [ ] Verify tool integrations work (calendar, webhooks)
- [ ] Review flow visually in dashboard for layout

#### Post-Deployment
- [ ] Review call transcripts for misrouting
- [ ] Add finetune examples for any observed misclassifications
- [ ] Monitor tool failure rates
- [ ] Update flow via PATCH /update-conversation-flow as needed

---

### 10.14 Available LLM Models (from API schema)

| Model | Use Case | Cost Tier |
|-------|----------|-----------|
| `gpt-4.1` | Default, best balance of quality/cost/latency | Medium |
| `gpt-4.1-mini` | Good for moderate complexity nodes | Low |
| `gpt-4.1-nano` | Simple routing, greetings, goodbyes | Lowest |
| `gpt-5` | Maximum quality | High |
| `gpt-5-mini` | High quality, lower cost | Medium-High |
| `gpt-5-nano` | Good quality at low cost | Low |
| `claude-4.5-sonnet` | Alternative provider, strong reasoning | Medium-High |
| `claude-4.5-haiku` | Alternative provider, fast | Low |
| `gemini-2.5-flash` | Alternative provider, fast | Low |
| `gemini-2.5-flash-lite` | Alternative provider, fastest | Lowest |

**Cost optimization strategy:** Use `cascading` model choice type. Set expensive models only on nodes that need complex reasoning (intent classification, data confirmation). Use nano/lite models for greeting, goodbye, and simple routing nodes.

---

### 10.15 Custom Tool (Webhook) Schema Template

```json
{
  "tool_id": "tool-unique-id",
  "type": "custom",
  "name": "verb_noun_context",
  "description": "One sentence describing when to call this tool (max 1024 chars)",
  "url": "https://api.example.com/endpoint",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer {{api_key}}",
    "Content-Type": "application/json"
  },
  "parameters": {
    "type": "object",
    "properties": {
      "customer_name": {
        "type": "string",
        "description": "Full name of the customer"
      },
      "phone": {
        "type": "string",
        "description": "Phone number in E.164 format"
      }
    },
    "required": ["customer_name", "phone"]
  },
  "response_variables": [
    { "name": "confirmation_id", "type": "string", "description": "Booking confirmation ID" }
  ],
  "timeout_ms": 10000
}
```

**`response_variables`** extract specific fields from the API response into dynamic variables usable in subsequent nodes.

---

### 10.16 MCP Node Configuration Template

```json
{
  "id": "node-mcp-lookup",
  "type": "mcp",
  "name": "CRM Lookup",
  "mcp_id": "mcp-server-id",
  "mcp_tool_name": "lookup_customer",
  "wait_for_result": true,
  "speak_during_execution": true,
  "instruction": {
    "type": "prompt",
    "text": "Let the caller know you're looking up their information."
  },
  "response_variables": [
    { "name": "account_status", "type": "string" }
  ],
  "edges": [
    { "id": "edge-found", "condition": "Customer found", "transition_condition": { "type": "prompt", "prompt": "Customer record was found" }, "destination_node_id": "node-next" },
    { "id": "edge-not-found", "condition": "Customer not found", "transition_condition": { "type": "prompt", "prompt": "Customer record was not found or lookup failed" }, "destination_node_id": "node-fallback" }
  ]
}
```

Flow-level MCP configuration:
```json
{
  "mcps": [
    {
      "name": "crm-server",
      "url": "https://mcp.example.com/sse",
      "headers": { "Authorization": "Bearer {{mcp_api_key}}" },
      "timeout_ms": 30000
    }
  ]
}
```

---

### 10.17 Extract Dynamic Variables Node Template

Use after collecting data to reliably store it as a named variable:

```json
{
  "id": "node-extract-contact",
  "type": "extract_dynamic_variables",
  "name": "Extract Contact Info",
  "variables": [
    { "type": "string", "name": "customer_name", "description": "The caller's full name", "examples": ["John Smith", "Maria Garcia"] },
    { "type": "string", "name": "customer_email", "description": "The caller's email address", "examples": ["john@example.com"] },
    { "type": "string", "name": "customer_phone", "description": "The caller's phone number", "examples": ["+15551234567"] }
  ],
  "edges": [
    { "id": "edge-extracted", "condition": "Variables extracted", "transition_condition": { "type": "prompt", "prompt": "Variables have been extracted" }, "destination_node_id": "node-next" }
  ]
}
```

This node examines the conversation history and extracts the specified variables. Place it after the collection nodes and before any node that needs `{{customer_name}}` etc.

---

### 10.18 Sources

- [Conversation Flow Overview](https://docs.retellai.com/build/conversation-flow/overview)
- [Conversation Node](https://docs.retellai.com/build/conversation-flow/conversation-node)
- [Node Overview](https://docs.retellai.com/build/conversation-flow/node)
- [Debug Guide](https://docs.retellai.com/build/conversation-flow/debug-guide)
- [Flex Mode](https://docs.retellai.com/build/conversation-flow/flex-mode)
- [Components](https://docs.retellai.com/build/conversation-flow/components)
- [Global Settings](https://docs.retellai.com/build/conversation-flow/global-setting)
- [Create Conversation Flow API](https://docs.retellai.com/api-references/create-conversation-flow)
- Real Estate AI Appointment Setter (exported JSON, analyzed)
- Landscaping Receptionist Demo (built and deployed via API)
- [Custom Function (Conversation Flow)](https://docs.retellai.com/build/conversation-flow/custom-function)
- [Dynamic Variables](https://docs.retellai.com/build/dynamic-variables)
- [n8n Custom Function Template](https://n8n.io/workflows/3805-connect-retell-voice-agents-to-custom-functions/)

---

### 10.19 Node Type API Testing Results

All node types tested via `POST /create-conversation-flow` on 2026-02-02:

| Node Type | JSON `type` | API Result | Notes |
|-----------|-------------|------------|-------|
| Conversation | `conversation` | **Working** | Previously confirmed |
| Function | `function` | **Working** | Previously confirmed |
| Transfer Call | `transfer_call` | **Working** | Previously confirmed |
| End | `end` | **Working** | Previously confirmed |
| Extract Dynamic Variables | `extract_dynamic_variables` | **Working** | `variables[]` array with `type`, `name`, `description` |
| Branch (Logic Split) | `branch` | **Working** | Requires `else_edge`; equation conditions work |
| SMS | `sms` | **Working** | Requires `success_edge` + `failed_edge` (singular, not array) |
| Press Digit | `press_digit` | **Working** | Uses `edges[]` array + optional `delay_ms` |
| Agent Swap | `agent_swap` | **Working** | `post_call_analysis_setting` must be `"both_agents"` or `"only_destination_agent"` (string enum, NOT object) |
| Component | `component` | **Partial** | Node validates, but `component_id` must reference a real component created in dashboard first. Local components via `components[]` array need correct ID matching. Error: `invalid local component id` when ID doesn't exist |
| MCP | `mcp` | **Not tested** | Requires a configured MCP server (`mcp_id`) — needs MCP setup in Retell dashboard first |
| Bridge Transfer | `bridge_transfer` | **Not tested** | Internal warm transfer node, rare use case |
| Cancel Transfer | `cancel_transfer` | **Not tested** | Internal transfer cancellation, rare use case |

**Key discoveries during testing:**

1. **`agent_swap` schema correction:** The API schema shows `post_call_analysis_setting` accepts an object, but the actual API requires a string enum: `"both_agents"` or `"only_destination_agent"`. This is a documentation/schema mismatch.

2. **Component creation flow:** Components must either:
   - Be created in the Retell dashboard first, then referenced by real `component_id`
   - Be defined in the `components[]` array of the create request with matching `component_id` (but the IDs must be valid format)

3. **All node types that use `edges[]` array:** conversation, function, press_digit, branch, extract_dynamic_variables, mcp, component
4. **Node types that use singular `edge`:** transfer_call, agent_swap (for failure only)
5. **Node types that use named singular edges:** sms (`success_edge` + `failed_edge`)
6. **Node types with required `else_edge`:** branch, component

---

### 10.20 PATCH /update-conversation-flow Behavior

Tested on 2026-02-02. The PATCH endpoint supports **partial field updates** but **full array replacement** for nodes.

#### What works:

| Operation | Method | Result |
|-----------|--------|--------|
| Update `global_prompt` only | `PATCH { "global_prompt": "..." }` | Works — only prompt changes, nodes/tools untouched |
| Update any top-level scalar field | `PATCH { "field": "value" }` | Works — other fields untouched |
| Replace all nodes | `PATCH { "nodes": [...full array...] }` | Works — **replaces entire nodes array** |
| Modify one node's instruction | Send full `nodes[]` with one node changed | Works — full replacement but achieves the effect |

#### What does NOT work:

| Operation | Result |
|-----------|--------|
| Send partial `nodes` (just one node) | **Replaces all nodes with just that one** — destroys the rest |
| Append a node to existing nodes | **Not supported** — must send complete array |
| Remove a single node | **Must send full array minus that node** |

#### Conclusion:

The PATCH API uses **merge semantics for top-level fields** but **replacement semantics for arrays**. To modify nodes:

1. GET the current flow
2. Modify the nodes array in memory
3. PATCH with the full modified `nodes` array

This is the safe workflow:
```bash
# 1. Get current flow
FLOW=$(curl -s -H "Authorization: Bearer $KEY" \
  https://api.retellai.com/get-conversation-flow/$FLOW_ID)

# 2. Modify in memory (e.g., with jq or python)
UPDATED=$(echo "$FLOW" | python3 -c "
import json, sys
flow = json.load(sys.stdin)
# ... modify flow['nodes'] ...
print(json.dumps({'nodes': flow['nodes']}))
")

# 3. PATCH with full nodes array
curl -s -H "Authorization: Bearer $KEY" -H "Content-Type: application/json" \
  -X PATCH https://api.retellai.com/update-conversation-flow/$FLOW_ID \
  -d "$UPDATED"
```

---

### 10.21 Custom Tool / Webhook Integration Guide

This section covers how to integrate Retell conversation flow agents with ANY external system (calendars, CRMs, databases, APIs) via custom function tools.

#### Architecture Overview

```
[Conversation Node: Collect Data] → [Function Node: Call Your API] → [Conversation Node: Present Results]
                                           |
                                    Retell sends POST to your endpoint
                                           |
                                    Your server processes + responds
                                           |
                                    Response flows back to agent as context
```

#### How Custom Functions Work

1. **You define a tool** at flow level in `tools[]` with type `"custom"`
2. **A function node** references that tool by `tool_id`
3. **When the flow reaches that node**, Retell sends an HTTP request to your endpoint
4. **Your endpoint processes the request** and returns a result
5. **The result becomes available** to the next conversation node as context

#### Request Format (sent FROM Retell TO your endpoint)

```json
{
  "name": "your_function_name",
  "args": {
    "param1": "value collected from conversation",
    "param2": "another value"
  },
  "call": {
    "call_type": "phone_call",
    "call_id": "Jabr9TXYYJHfvl6Syypi88rdAHYHmcq6",
    "agent_id": "oBeDLoLOeuAbiuaMFXRtDOLriTJ5tSxD",
    "from_number": "+12137771234",
    "to_number": "+12137771235",
    "direction": "inbound",
    "call_status": "registered",
    "metadata": {},
    "retell_llm_dynamic_variables": {
      "customer_name": "John Doe"
    }
  }
}
```

**Headers sent by Retell:**
- `X-Retell-Signature` — HMAC signature for request verification
- `Content-Type: application/json`

**Retell's IP (for allowlisting):** `100.20.5.228`

#### Response Requirements

- Status code: `200-299` for success
- Body: string, JSON object, or buffer (all converted to string for LLM)
- **Max 15,000 characters** — responses are truncated beyond this
- Timeout: your configured value, or **2 minutes default**
- **Retries: up to 2 times** on failure

#### Tool Definition Schema (in flow `tools[]`)

```json
{
  "tool_id": "tool-check-calendar",
  "type": "custom",
  "name": "check_calendar_availability",
  "description": "Check calendar availability for a given date range and service type",
  "url": "https://your-api.com/check-availability",
  "method": "POST",
  "headers": {
    "Authorization": "Bearer {{calendar_api_key}}",
    "X-Custom-Header": "static-value"
  },
  "query_params": {
    "timezone": "America/Los_Angeles"
  },
  "parameters": {
    "type": "object",
    "properties": {
      "date": {
        "type": "string",
        "description": "The date to check in YYYY-MM-DD format"
      },
      "service_type": {
        "type": "string",
        "description": "Type of service (e.g., 'consultation', 'installation')"
      },
      "duration_minutes": {
        "type": "number",
        "description": "Desired appointment duration in minutes"
      }
    },
    "required": ["date", "service_type"]
  },
  "response_variables": [
    { "name": "available_slots", "type": "string", "description": "Comma-separated available time slots" },
    { "name": "next_available_date", "type": "string", "description": "Next date with availability if none today" }
  ],
  "timeout_ms": 15000
}
```

#### Key Configuration Details

| Field | Purpose | Notes |
|-------|---------|-------|
| `parameters.properties` | JSON Schema for args the LLM should extract | Follows OpenAI function-calling format |
| `parameters.required` | Which args are mandatory | LLM will ask for these before calling |
| `response_variables` | Extract specific fields from response into `{{dynamic_variables}}` | Optional; without these, entire response goes to LLM as text |
| `headers` | Custom HTTP headers | Support `{{dynamic_variable}}` syntax for secrets |
| `query_params` | URL query parameters | Can be const or LLM-described |
| `timeout_ms` | Request timeout (1000-600000) | Default: 120000 (2 min) |

#### Signature Verification (your endpoint)

**Node.js:**
```javascript
import { Retell } from "retell-sdk";

app.post("/your-endpoint", async (req, res) => {
  const isValid = Retell.verify(
    JSON.stringify(req.body),
    process.env.RETELL_API_KEY,
    req.headers["x-retell-signature"]
  );
  if (!isValid) return res.status(401).json({ error: "Unauthorized" });

  const { name, args, call } = req.body;
  // Process and respond...
  res.json({ result: "your data here" });
});
```

**Python (FastAPI):**
```python
from retell import Retell

@app.post("/your-endpoint")
async def handle(request: Request):
    body = await request.json()
    valid = retell.verify(
        json.dumps(body, separators=(",", ":"), ensure_ascii=False),
        api_key=os.environ["RETELL_API_KEY"],
        signature=request.headers.get("X-Retell-Signature"),
    )
    if not valid:
        return JSONResponse(status_code=401, content={"error": "Unauthorized"})

    args = body["args"]
    # Process and respond...
    return JSONResponse(status_code=200, content={"result": "your data here"})
```

#### Calendar Integration Pattern (Any Provider)

To integrate with **any** calendar provider (Google Calendar, Outlook, Calendly, Acuity, custom), you build a middleware endpoint that Retell calls as a custom tool.

**Architecture:**
```
Retell Agent → [Custom Tool: check_availability] → Your Server → Calendar API → Response → Agent
Retell Agent → [Custom Tool: book_appointment] → Your Server → Calendar API → Response → Agent
```

**Your middleware handles:**
1. Receiving Retell's request (with `args` like date, time, service)
2. Translating to the calendar provider's API format
3. Calling the provider (Google Calendar API, Outlook Graph API, Calendly API, etc.)
4. Formatting the response for voice (oral format, not raw JSON)
5. Returning to Retell

**Example middleware endpoint (check availability):**
```javascript
app.post("/check-availability", async (req, res) => {
  // Verify signature...
  const { date, service_type } = req.body.args;

  // Call your calendar provider
  const slots = await googleCalendar.freebusy({
    timeMin: `${date}T09:00:00`,
    timeMax: `${date}T17:00:00`,
    items: [{ id: "primary" }]
  });

  // Format for voice
  const available = formatSlotsForVoice(slots);
  res.json({ available_slots: available });
});

function formatSlotsForVoice(slots) {
  // Convert "2026-02-03T14:00:00" to "Tuesday February third at two pm"
  return slots.map(s => formatOral(s)).join(", ");
}
```

**Tool schema for calendar check:**
```json
{
  "tool_id": "tool-cal-check",
  "type": "custom",
  "name": "check_calendar_availability",
  "description": "Check what appointment times are available for a given date and service",
  "url": "https://your-server.com/check-availability",
  "method": "POST",
  "parameters": {
    "type": "object",
    "properties": {
      "date": { "type": "string", "description": "Date in YYYY-MM-DD format" },
      "service_type": { "type": "string", "description": "Service the caller wants" }
    },
    "required": ["date"]
  },
  "response_variables": [
    { "name": "available_slots", "type": "string", "description": "Available time slots formatted for voice" }
  ],
  "timeout_ms": 15000
}
```

**Tool schema for booking:**
```json
{
  "tool_id": "tool-cal-book",
  "type": "custom",
  "name": "book_appointment",
  "description": "Book an appointment at the specified date and time",
  "url": "https://your-server.com/book-appointment",
  "method": "POST",
  "parameters": {
    "type": "object",
    "properties": {
      "date": { "type": "string", "description": "Date in YYYY-MM-DD format" },
      "time": { "type": "string", "description": "Time in HH:MM format (24hr)" },
      "customer_name": { "type": "string", "description": "Caller's full name" },
      "customer_email": { "type": "string", "description": "Caller's email" },
      "customer_phone": { "type": "string", "description": "Caller's phone number" },
      "service_type": { "type": "string", "description": "Service requested" }
    },
    "required": ["date", "time", "customer_name"]
  },
  "response_variables": [
    { "name": "booking_confirmation", "type": "string", "description": "Confirmation message" },
    { "name": "booking_id", "type": "string", "description": "Booking reference ID" }
  ],
  "timeout_ms": 20000
}
```

#### CRM Integration Pattern

Same architecture — your middleware translates between Retell and your CRM.

**Common CRM tools to build:**

| Tool | Purpose | When Called |
|------|---------|------------|
| `lookup_customer` | Find customer by phone/email | At call start (use `call.from_number`) |
| `create_lead` | Create new lead record | After qualifying a new caller |
| `update_contact` | Update contact details | After collecting/confirming info |
| `log_call_notes` | Save call summary to CRM | At call end (via webhook, not tool) |

**Example: CRM lookup tool using caller's phone number:**
```json
{
  "tool_id": "tool-crm-lookup",
  "type": "custom",
  "name": "lookup_customer_by_phone",
  "description": "Look up customer record using their phone number",
  "url": "https://your-server.com/crm/lookup",
  "method": "POST",
  "parameters": {
    "type": "object",
    "properties": {
      "phone_number": { "type": "string", "description": "Customer phone number" }
    },
    "required": ["phone_number"]
  },
  "response_variables": [
    { "name": "customer_name", "type": "string", "description": "Customer's name from CRM" },
    { "name": "account_status", "type": "string", "description": "Active, inactive, or new" },
    { "name": "last_service_date", "type": "string", "description": "Date of last service" }
  ],
  "timeout_ms": 10000
}
```

#### Integration with Automation Platforms

Instead of building your own server, you can use:

- **n8n** (self-hosted or cloud) — Retell sends webhook to n8n, which orchestrates the API calls and returns response
- **Make.com** — drag-and-drop automation connecting Retell to 1000+ apps
- **Zapier** — similar to Make, with Retell webhook trigger

These platforms handle the middleware layer, so you don't need to code a server.

#### Custom Tool Guardrails

```
IF your endpoint might be slow (>5s)
  THEN set speak_during_execution: true on the function node
  THEN set a reasonable timeout_ms (don't use 2min default)

IF your endpoint returns structured data
  THEN use response_variables to extract specific fields
  THEN format for voice in your middleware (not raw JSON)

IF LLM might hallucinate function args
  THEN validate all args on your server
  THEN return clear error messages the LLM can relay to the user
  THEN note in tool description what formats are expected

IF your endpoint handles sensitive actions (booking, payment, deletion)
  THEN always verify X-Retell-Signature
  THEN always have a confirmation conversation node BEFORE the function node

IF your endpoint is external (not your own server)
  THEN proxy through your own server for signature verification
  THEN add API key management on your side
```

#### System Variables Available in Custom Tool Calls

The `call` object in every request includes:
- `call.from_number` — caller's phone (use for CRM lookup without asking)
- `call.to_number` — your agent's phone number
- `call.direction` — "inbound" or "outbound"
- `call.call_id` — unique call identifier
- `call.metadata` — custom metadata set when creating the call
- `call.retell_llm_dynamic_variables` — all dynamic variables set for this call

---

### 10.22 Native Integrations: HubSpot

Retell has an **official native HubSpot integration** — no custom middleware required.

#### How It Works

1. **HubSpot Workflow triggers Retell outbound call** — supported triggers:
   - Form submission
   - Contact record creation
   - Property value change
   - Date-based (e.g., follow-up after X days)

2. **Retell makes the call** using a specified agent and phone number

3. **Call results branch back into HubSpot workflow** — you can branch on:
   - Call success/failure
   - Sentiment (positive, neutral, negative)
   - Custom post-call analysis fields
   - Whether the call was answered

4. **Call data auto-syncs to HubSpot contact records**:
   - Recording URL
   - Transcript
   - Call duration
   - Post-call analysis results (lead score, intent, etc.)

#### Setup

- Connect via Retell dashboard → Integrations → HubSpot (OAuth flow)
- In HubSpot, add a "Retell AI Call" action in any workflow
- Configure: which agent, which phone number, dynamic variables from HubSpot fields

#### When to Use Native vs Custom Tool

| Scenario | Approach |
|----------|----------|
| Outbound calls triggered by HubSpot events | Native integration |
| Inbound call needs to look up HubSpot contact | Custom tool (webhook to HubSpot API) |
| Call results need to update HubSpot properties | Native integration (automatic) |
| Complex multi-step CRM logic during a call | Custom tool via middleware |
| Simple "trigger call on form submit" | Native integration |

#### Also Available Via

- **Make.com** — Retell + HubSpot modules for custom automation flows
- **n8n** — webhook-based integration with full control over data mapping

#### Key Takeaway

For outbound campaign-style calls triggered by CRM events, use the native integration. For inbound calls that need to query/update HubSpot mid-conversation, use custom tools with the webhook pattern from section 10.21.

---

### 10.23 Session Log Update

### 2026-02-02 (continued): Node Testing, PATCH Behavior, Custom Tool Research

- Tested 6 additional node types via API: `extract_dynamic_variables`, `branch`, `sms`, `press_digit`, `agent_swap`, `component`
- Discovered `agent_swap.post_call_analysis_setting` must be `"both_agents"` or `"only_destination_agent"` (API schema says object, actual API requires string enum)
- Discovered `component` node requires pre-existing `component_id` — cannot create components purely via API in one shot (dashboard-first)
- Confirmed PATCH behavior: partial updates for scalar fields, full replacement for `nodes[]` array
- Documented complete custom tool/webhook integration patterns for any calendar, CRM, or external API
- Cleaned up test flows (deleted `conversation_flow_36d439718146` and `conversation_flow_f5f09d8aa338`)
- Researched and documented HubSpot native integration with Retell (section 10.22): outbound calls triggered by HubSpot workflows, call results branch back into HubSpot, auto-sync of recordings/transcripts/analysis to contact records
