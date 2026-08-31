# Setup

**MCP · Google ADK · A2A**

The short path to a working demo. Each step ends with something you can check, so you
find out immediately if it went wrong rather than three steps later.

The repo's [`README.md`](README.md) is fuller reference material — use it *after* you
are running. To teach the session, read
[`INSTRUCTOR_GUIDE.md`](INSTRUCTOR_GUIDE.md) once you finish here.

> **Do this the day before, not the morning of.** Step 2 downloads several hundred
> megabytes, and step 5 needs a working OpenAI key.

---

## Step 0 · Windows only — two things first

Skip this section on macOS or Linux.

**Clone somewhere short.** A few files in this repo have long paths. Combined with a
deep folder they cross Windows' 260-character limit and git starts reporting phantom
changes. Clone to `C:\dev\` or similar, not to a nested Downloads folder.

```powershell
git config --global core.longpaths true
```

**Set the console to UTF-8.** The demos draw box-drawing characters; the default
`cp1252` console cannot encode them and the script dies on its first line of output.

```powershell
$env:PYTHONIOENCODING = "utf-8"     # PowerShell
set PYTHONIOENCODING=utf-8           # cmd
```

---

## Step 1 · Check what you already have

```bash
python --version    # need 3.10 or newer
node --version      # need 18 or newer
```

Node is needed for **one** demo — the GitHub MCP agent in Demo 1. Everything else
runs without it, so a missing Node is not a blocker on the day.

**Checkpoint:** both print a version at or above those numbers.

---

## Step 2 · Virtual environment and dependencies

From the repo root:

```bash
# macOS / Linux
python3 -m venv .venv && source .venv/bin/activate

# Windows PowerShell
python -m venv .venv
Set-ExecutionPolicy -Scope Process -ExecutionPolicy Bypass; .\.venv\Scripts\Activate.ps1

# Windows cmd
python -m venv .venv && .venv\Scripts\activate.bat
```

Your prompt should now start with `(.venv)`. Then:

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

This pulls in MCP, the OpenAI SDK, Google ADK, LiteLLM and the A2A SDK. It takes a
few minutes.

**Checkpoint:**

```bash
python -c "import mcp, openai, google.adk, litellm; print('imports ok')"
adk --version
```

Both must succeed. If `adk` is not found, your venv is not active — activate it and
try again. `which adk` (or `where adk`) should point inside `.venv`.

---

## Step 3 · Your API key

```bash
cp .env.example .env          # Windows: Copy-Item .env.example .env
```

Edit `.env` and set:

```env
OPENAI_API_KEY=sk-your-key-here
GITHUB_TOKEN=ghp-your-token-here   # optional — one demo only
```

**Which GitHub token?** A Personal Access Token on your own GitHub account, needed by
exactly one demo (`d1_mcp/github_agent_client.py`). Create one at
<https://github.com/settings/tokens> → *Generate new token (classic)* → tick
`public_repo`. Skip it if you are not running that demo — nothing else needs it.

`.env` is already in `.gitignore`. Never commit it.

**Cost:** a full capstone negotiation is a handful of GPT-4o calls — cents, not
dollars. To spend less while rehearsing, add `AGENT_MODEL=openai/gpt-4o-mini`.

---

## Step 4 · First demo — no API key needed

```bash
python d1_mcp/demos/01_initialize_handshake.py
```

**Checkpoint:** five JSON-RPC frames scroll past and the last one lists **two tools**
with their schemas. That proves MCP works end to end without touching a model.

If this works, your Python environment is correct.

---

## Step 5 · Second demo — proves your key works

```bash
python d1_mcp/demos/02_tool_loop_trace.py
```

**Checkpoint:** a timestamped trace across three actors — Model, Host, Server —
ending in an answer. This is the first thing that calls OpenAI, so it is where a bad
key shows up.

| Error | Meaning |
|---|---|
| `AuthenticationError` | Key wrong or not loaded from `.env` |
| `RateLimitError` | Key valid, no quota |
| `UnicodeEncodeError` | Windows console — see Step 0 |
| `No module named 'mcp.server.fastmcp'` | `mcp` 2.x got installed. `requirements.txt` pins `mcp>=1.24,<2` for a reason — reinstall from it |

---

## Step 6 · The ADK web UI

```bash
cd d2_adk_multiagents
adk web adk_demos
```

Open **http://localhost:8000**.

**Checkpoint:** a dropdown listing nine agents, `d01_basic_agent` through
`d09_event_stream`. Pick `d01_basic_agent`, say hello, get a reply.

If the dropdown is empty, you are in the wrong directory — `adk web` needs the folder
*containing* the agent packages, not one of the packages.

Ctrl+C to stop.

---

## Step 7 · The capstone — the run you will teach

```bash
cd d2_adk_multiagents
adk web negotiation_agents
```

At **http://localhost:8000**, pick **`negotiation`** and send:

```
Start negotiation
```

**Checkpoint:** the buyer makes an offer backed by market data, the seller counters,
and within five rounds you get an accept or a clean deadlock. Open the **State** tab
and you should see something like:

```json
{"seller_decision": {"action": "ACCEPT", "price": 445000}}
```

**You are set up.** That state dict is what ended the negotiation — a value code
checked, not a phrase in the transcript.

> The exact prices and round count vary between runs; the model is not deterministic.
> The walk-away limits ($460K buyer, $445K seller) hold every time, and that is the
> guarantee worth pointing at. Don't re-run trying to match the deck.

---

## Optional · A2A across processes

Only needed for the last part of Section 3. Two terminals, both with the venv active.

```bash
# Terminal 1 — agents with A2A endpoints and Agent Cards
cd d2_adk_multiagents && adk web --a2a negotiation_agents

# Terminal 2 — drive the negotiation over HTTP
python d2_adk_multiagents/a2a_13_orchestrated_negotiation.py
```

**Checkpoint:** the same negotiation, now between two independently-running agents
talking over HTTP instead of in one process.

---

## Optional · The GitHub MCP demo

The only piece that needs Node and a `GITHUB_TOKEN`.

```bash
python d1_mcp/github_agent_client.py
```

**Checkpoint:** one LLM discovers 20+ tools from GitHub's own MCP server and answers a
question about a repository — with no integration code written for it. It is the
strongest single argument for the protocol, so it is worth the extra setup.

---

## A quick check before you teach

```bash
source .venv/bin/activate                      # or the Windows equivalent
python d1_mcp/demos/01_initialize_handshake.py # env is fine
cd d2_adk_multiagents && adk web negotiation_agents
```

If the handshake prints and the dropdown loads, you are ready to teach.
