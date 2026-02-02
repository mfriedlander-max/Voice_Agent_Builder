# AI Scratchpad - Voice Agent Builder

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

- [ ] Test remaining node types via API (Press Digit, Logic Split, Agent Transfer, SMS, Extract Variable, MCP Node, Components)
- [ ] Build reusable templates for common agent types
- [ ] Set up Cal.com integration for scheduling
- [ ] Explore Retell API docs for any additional undocumented endpoints
- [ ] Clean up test agents (Landscaping Receptionist - Test, Landscaping Receptionist Demo)
- [ ] Investigate if conversation flow update API can modify individual nodes or requires full replacement
