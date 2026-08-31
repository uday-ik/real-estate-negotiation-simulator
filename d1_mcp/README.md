# Demo 1 — MCP Servers (`d1_mcp`)

MCP (Model Context Protocol) is the standard that lets an AI agent call external tools
without knowing where the data behind them comes from. This demo builds that up from
the raw protocol frames to a live agent using someone else's MCP server.

Run everything from the **repo root**, with your virtual environment active.

---

## Run these in order

| # | Command | Needs | What you should see |
|---|---|---|---|
| 1 | `python d1_mcp/demos/01_initialize_handshake.py` | nothing | Five JSON-RPC frames, ending with two tools whose schemas came from Python type hints |
| 2 | `python d1_mcp/demos/02_tool_loop_trace.py` | `OPENAI_API_KEY` | A timestamped trace across Model, Host and Server |
| 3 | `python d1_mcp/demos/03_list_all_primitives.py` | nothing | Tools **and** Resources **and** Prompts — MCP carries more than tools |
| 4 | `python d1_mcp/demos/04_content_types.py` | nothing | Text, image and embedded-resource result blocks |
| 5 | `python d1_mcp/github_agent_client.py` | `OPENAI_API_KEY`, `GITHUB_TOKEN`, Node.js | One model discovering 20+ tools from GitHub's own MCP server |
| 6 | `python d1_mcp/sse_agent_client.py` | `OPENAI_API_KEY`, 3 terminals | The same tool loop over HTTP instead of stdio |

Demos 1, 3 and 4 need no keys at all — start there to confirm your environment works.

---

## Getting the GitHub token (demo 5 only)

Demo 5 is the only thing here that needs GitHub. It runs GitHub's official MCP server
through `npx`, and that server needs a token to talk to the GitHub API. It is a normal
**Personal Access Token** on your own GitHub account — nothing to do with this repo.

1. Go to <https://github.com/settings/tokens> → **Generate new token** →
   **Generate new token (classic)**.
2. Give it a name like `mcp-demo` and a short expiry.
3. Tick **`public_repo`** under `repo`. That is enough — the demo only reads public
   repositories. Tick the full `repo` scope only if you want it to reach your private
   ones.
4. Generate it and copy the value. It starts with `ghp_`, and GitHub shows it once.
5. Put it in your `.env` beside your OpenAI key:

```env
OPENAI_API_KEY=sk-your-key-here
GITHUB_TOKEN=ghp_your-token-here
```

You also need **Node.js 18+** for this one, because the GitHub MCP server is a Node
package that `npx` fetches on first run. Check with `node --version`. The first run is
slower while `npx` downloads it.

**Skipping it is fine.** Nothing else in either demo depends on GitHub — it is here
because watching a model use a server that somebody else wrote is the clearest
argument for the protocol.

---

## The two servers in this folder

`pricing_server.py` and `inventory_server.py` are the MCP servers the buyer and seller
agents use in Demo 2. Run either one directly and it will sit there waiting for a
client — that is correct behaviour for a server, not a hang. Ctrl+C to stop it.

The split between them is deliberate: the seller's floor price lives in
`inventory_server.py`, which the buyer's agent is never wired to.

---

## What this module teaches

A naive negotiation agent uses hardcoded prices — the agents have no real market data and just make up numbers.

MCP fixes that. It is a protocol that:
1. Lets a server **expose tools** (functions with typed inputs/outputs)
2. Lets a client **discover** those tools automatically (`list_tools`)
3. Lets a client **call** those tools over a standard interface (`call_tool`)

The agent doesn't need to know if the data comes from a database, an API, or a spreadsheet. It just calls the tool by name.

---

## File breakdown

### `github_agent_client.py` — An LLM agent that uses GitHub via MCP

This file connects to **GitHub's official MCP server** and lets GPT-4o decide which tools to call based on your natural language query.

Why GitHub? Because you already know what GitHub does. Seeing an LLM agent use MCP with a familiar tool makes the protocol click before you write your own servers.

**What it demonstrates:**
- Connecting to an MCP server over `stdio` transport
- Auto-discovering tools via `list_tools` and converting schemas to OpenAI function-calling format
- The LLM **choosing** which tools to call (agentic, not scripted)
- The ReAct-style tool loop: LLM calls tools, gets results, calls more tools or answers
- This is the **same pattern** used by our ADK buyer/seller agents in Demo 2 (via `MCPToolset`)

**Prerequisites:**
- Node.js 18+ installed (`node --version`)
- A GitHub Personal Access Token (`repo` or `public_repo` scope)
- An OpenAI API key

```bash
export GITHUB_TOKEN=ghp_your_token_here
export OPENAI_API_KEY=sk-your_key_here
python d1_mcp/github_agent_client.py

# Or with a custom query:
python d1_mcp/github_agent_client.py "Find popular MCP server implementations"
```

---

### `sse_agent_client.py` — SSE agent client (connects over HTTP)

An LLM-powered agent that connects to the pricing and/or inventory servers running in **SSE mode** (HTTP) and lets GPT-4o decide which tools to call — the same agentic pattern as `github_agent_client.py`, just a different transport.

**What it demonstrates:**
- Connecting to MCP servers via SSE (Server-Sent Events) transport
- The LLM **choosing** which tools to call (agentic, not scripted)
- Connecting to multiple MCP servers from a single agent
- Proving the transport is irrelevant — same protocol, same agent loop

**Prerequisites:**
- Start the servers in SSE mode first (in separate terminals)
- An OpenAI API key

```bash
# Terminal 1:
python d1_mcp/pricing_server.py --sse --port 8001

# Terminal 2:
python d1_mcp/inventory_server.py --sse --port 8002

# Terminal 3 — run the agent:
python d1_mcp/sse_agent_client.py                                          # all sample queries
python d1_mcp/sse_agent_client.py "Is this property overpriced?"            # custom query
python d1_mcp/sse_agent_client.py --both "Use inventory and pricing data"   # both servers
```

---

### `pricing_server.py` — Custom MCP server for market pricing

This is the first custom MCP server. It wraps simulated real estate pricing data and exposes it as MCP tools.

**Tools it exposes:**

| Tool | What it does | Who uses it |
|---|---|---|
| `get_market_price(address, property_type)` | Returns comparable sales, estimated value, and market analysis | Both buyer and seller |
| `calculate_discount(base_price, market_condition, days_on_market, property_condition)` | Returns suggested offer ranges and negotiation tips | Both buyer and seller |

**Two transport modes (same server, different usage):**

```bash
# stdio — default, client spawns this as a subprocess (used by Modules 3 + 4)
python d1_mcp/pricing_server.py

# SSE — HTTP server, multiple clients can connect at once
python d1_mcp/pricing_server.py --sse --port 8001
```

In Modules 3 and 4, the agents start this server automatically as a subprocess. You don't need to run it manually — but you *can* to inspect it.

---

### `inventory_server.py` — Custom MCP server for inventory + seller constraints

This server simulates an MLS (Multiple Listing Service) system. It has one public tool and one **seller-only** tool.

**Tools it exposes:**

| Tool | What it does | Who uses it |
|---|---|---|
| `get_inventory_level(zip_code)` | Returns active listings, days on market, market condition | Both buyer and seller |
| `get_minimum_acceptable_price(property_id)` | Returns the seller's absolute floor price | **Seller only** |

**The information asymmetry lesson:**

The buyer agent never connects to `get_minimum_acceptable_price`. This is intentional — in real estate, only the seller's agent knows the seller's walk-away price. The seller uses this tool to set a hard floor; the buyer has to negotiate without knowing it.

This is the same pattern used in real production systems: MCP access control means different agents get different tools.

```bash
# stdio — default
python d1_mcp/inventory_server.py

# SSE
python d1_mcp/inventory_server.py --sse --port 8002
```

---

## MCP in one diagram

```
AGENT                       MCP Protocol              SERVER
─────────────────           ────────────────          ──────────────────
"What tools exist?"
await session.list_tools() ─────────────────────────> Returns tool schemas
                                                       [{name, description,
                                                         inputSchema}]

"Call this tool"
await session.call_tool(    ─────────────────────────> Executes Python fn
  "get_market_price",
  {"address": "742..."}
)
                           <───────────────────────── Returns result dict
"Comps avg $462K,
listing is 4.9% above
market. I'll offer $425K."
```

The agent never imports your Python functions directly. It talks to the server over the protocol — the same way whether the server is local or remote.

---

## How to run

```bash
# GitHub MCP agent (needs GITHUB_TOKEN + OPENAI_API_KEY + Node.js)
export GITHUB_TOKEN=ghp_your_token_here
export OPENAI_API_KEY=sk-your_key_here
python d1_mcp/github_agent_client.py

# Inspect pricing server tools (no API key needed)
python d1_mcp/pricing_server.py
# Then Ctrl+C to stop (it's a server, it waits for connections)

# Inspect inventory server tools (no API key needed)
python d1_mcp/inventory_server.py

# SSE mode — run in one terminal, connect from another
python d1_mcp/pricing_server.py --sse --port 8001
python d1_mcp/inventory_server.py --sse --port 8002

# Then connect with the SSE agent client (in another terminal)
python d1_mcp/sse_agent_client.py                    # all sample queries (pricing only)
python d1_mcp/sse_agent_client.py --both              # includes inventory queries too
```

**What to expect from the GitHub agent:**
- It connects to GitHub's server via `npx`
- Lists all available tools (there are ~20+)
- GPT-4o decides which tools to call based on your query
- Executes the tool calls via MCP and feeds results back
- Produces a natural language summary

**What to expect from the pricing/inventory servers:**
- They start and wait for a client to connect
- On their own they don't print much — they're servers
- In Modules 3 and 4, the agents connect to them automatically

---

## Deep-dive demos (`d1_mcp/demos/`)

Standalone, runnable scripts that crack open the MCP protocol so you can see what's happening on the wire. Each one is self-contained and prints what it sends/receives. Read them in order; companion notes live in [d1_mcp/notes/mcp_deep_dive.md](d1_mcp/notes/mcp_deep_dive.md).

| Demo | What it shows | Run |
|---|---|---|
| [`01_initialize_handshake.py`](d1_mcp/demos/01_initialize_handshake.py) | Raw JSON-RPC frames of the MCP `initialize` handshake (no SDK) — capability negotiation, `notifications/initialized`, then `tools/list` | `python d1_mcp/demos/01_initialize_handshake.py` |
| [`02_tool_loop_trace.py`](d1_mcp/demos/02_tool_loop_trace.py) | The full **model ↔ host ↔ server** tool-calling loop, narrated step by step with timestamps (uses the `mcp` SDK + OpenAI function calling) | `python d1_mcp/demos/02_tool_loop_trace.py` |
| [`03_list_all_primitives.py`](d1_mcp/demos/03_list_all_primitives.py) | Lists **Tools, Resources, and Prompts** from both of the servers here — proves MCP carries more than just tools | `python d1_mcp/demos/03_list_all_primitives.py` |
| [`04_content_types.py`](d1_mcp/demos/04_content_types.py) | A tiny inline server that returns each `Content` block kind (text / image / embedded resource) so you can see the JSON shape of each | `python d1_mcp/demos/04_content_types.py` |
| [`05_streamable_http_transport.py`](d1_mcp/demos/05_streamable_http_transport.py) | Same MCP protocol, **Streamable HTTP** transport (the spec's recommended replacement for raw SSE) | Server: `python d1_mcp/demos/05_streamable_http_transport.py --serve --port 8765`<br>Client: `python d1_mcp/demos/05_streamable_http_transport.py --client --port 8765` |

> Demos 01–04 spawn their own server subprocess — no manual setup. Only demo 05 needs two terminals.

---

## Inspect visually with the MCP Inspector

The demos above show the protocol in your terminal. The **MCP Inspector** is the official browser-based GUI host — it connects to any MCP server and renders **Tools, Resources, and Prompts in separate tabs**, so you can *see* a generic host treat each primitive type differently (exactly what Claude Desktop does when it wires tools into the model loop, lists prompts as slash-commands, and shows resources in its attachment picker).

No Python code or API key needed — it runs via `npx` (Node.js 18+). Point it at either server:

```bash
# Inventory server — 2 tools + 1 resource (inventory://floor-prices)
npx @modelcontextprotocol/inspector python d1_mcp/inventory_server.py

# Pricing server — 2 tools + 1 prompt (negotiation-tactics)
npx @modelcontextprotocol/inspector python d1_mcp/pricing_server.py
```

> On Windows, if `python` is not the project venv, use its interpreter explicitly:
> `npx @modelcontextprotocol/inspector .venv\Scripts\python.exe d1_mcp\inventory_server.py`

Open the printed `http://localhost:6274?...` URL, click **Connect** (this runs the same `initialize` + `*/list` discovery as demo 03), then explore the tabs:

| Tab | Try this | The lesson |
|---|---|---|
| **Tools** | Run `get_minimum_acceptable_price` with `742-evergreen-austin-78701` | A model-invokable action with typed args — the host gives you a form + "Run" |
| **Resources** | Read `inventory://floor-prices` | You *read* it; there's no args form. The host never "calls" a resource |
| **Prompts** *(pricing server)* | Render `negotiation-tactics` with `role=seller` | A user-selected template — surfaced for a human to pick, not the model |

This is the visual companion to [`03_list_all_primitives.py`](d1_mcp/demos/03_list_all_primitives.py): that script *lists* the primitives on the wire; the Inspector *renders* them the way a real host would — sorting them into tabs purely from the `type` each was declared with, with zero Inspector-specific code from you.

---

## Try it in a full chat host (Claude Desktop)

The Inspector is a debug GUI. **Claude Desktop** is a production MCP host — wiring these servers into it shows the primitives inside a real chat, exactly as an end user would meet them.

**1. Install** Claude Desktop from [claude.ai/download](https://claude.ai/download) and sign in.

**2. Register the servers.** Open **Settings → Developer → Edit Config** (this opens the correct `claude_desktop_config.json`) and add an `mcpServers` block — use **absolute paths** and the **venv's** `python.exe` so the `mcp` dependency is available:

```json
{
  "mcpServers": {
    "real-estate-pricing": {
      "command": "C:\\path\\to\\real-estate-negotiation-simulator\\.venv\\Scripts\\python.exe",
      "args": ["C:\\path\\to\\real-estate-negotiation-simulator\\d1_mcp\\pricing_server.py"]
    },
    "real-estate-inventory": {
      "command": "C:\\path\\to\\real-estate-negotiation-simulator\\.venv\\Scripts\\python.exe",
      "args": ["C:\\path\\to\\real-estate-negotiation-simulator\\d1_mcp\\inventory_server.py"]
    }
  }
}
```

Config file location (the **Edit Config** button always opens the right one):
- **Windows (standalone install):** `%APPDATA%\Claude\claude_desktop_config.json`
- **Windows (Microsoft Store install):** `%LOCALAPPDATA%\Packages\Claude_*\LocalCache\Roaming\Claude\claude_desktop_config.json`
- **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`

> JSON on Windows needs **doubled backslashes** (`\\`). If the file already has other keys (preferences, etc.), just add `mcpServers` alongside them — don't overwrite the file.

**3. Fully restart** Claude Desktop — quit from the **system tray** (right-click → Quit), not just the window. Config is read only at launch.

**4. Verify:** the **tools icon** in the chat input bar lists both servers and their tools; **Settings → Developer** shows each server's status (click a red one for its error log — almost always a wrong path or non-venv Python).

**5. See each primitive type in a real chat:**
- **Tool** — ask: *"What's the market price for 742 Evergreen Terrace, Austin TX 78701?"* Claude asks to run `get_market_price`, then answers from the result.
- **Prompt** — the **`+` / attachment** menu surfaces `negotiation-tactics`; pick it, set `role=seller`, and it drops into the chat.
- **Resource** — the same menu lets you attach `inventory://floor-prices` as context. You *attach* it — Claude never "calls" it.

Same servers, same primitives you saw in the Inspector — now inside a production host.

---

## Exercises

Three hands-on exercises, as optional practice. Try them on your own, then check yourself against the worked solution in `solution/`.

| Exercise | Difficulty | Task |
|---|---|---|
| [`ex01_walk_score_tool.md`](exercises/ex01_walk_score_tool.md) | `[Starter]` | Add a `get_walk_score(zip_code)` tool to `pricing_server.py`. Restart `adk web`, ask a walkability question, watch GPT-4o auto-discover and call the new tool with zero agent-side changes. |
| [`ex02_multi_server_agent.md`](exercises/ex02_multi_server_agent.md) | `[Core]` | Build an `LlmAgent` from scratch that connects to BOTH `pricing_server` and `inventory_server` simultaneously. Test with cross-server queries; understand how ADK merges tool catalogs across multiple `MCPToolset`s. |
| [`ex03_server_failure_handling.md`](exercises/ex03_server_failure_handling.md) | `[Core]` | Build a multi-server agent that gracefully handles MCP server crashes — detects tool failures via `after_tool_callback` and degrades with fallback responses instead of crashing. The #1 production resilience pattern. |

Each solution lives in `solution/<exercise_name>/` as a self-contained, runnable package.

---

## Quick mental model
- If you want to see *how to build* an MCP server, read `pricing_server.py` (simpler, 2 tools) then `inventory_server.py` (adds the seller-only tool).
- The `@mcp.tool()` decorator is all you need to expose a Python function as an MCP tool.
- In Modules 3 and 4, both agents use these servers — you don't need to start them manually.
