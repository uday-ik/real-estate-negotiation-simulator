# Instructor Guide — MCP, Google ADK & A2A

**MCP · Google ADK · A2A**

This guide covers running the negotiation simulator live: the **MCP** section first,
then **Google ADK and A2A**.

Work through [`SETUP.md`](SETUP.md) before the session so everything on your machine
already runs.

---

## What this module is about

Agents reach the outside world through tools. This project replaces hand-wired
integrations with two open protocols:

- **MCP** — how an agent reaches a *tool*: a pricing feed, a database, an API.
- **A2A** — how an agent reaches *another agent*: one with its own goals, its own
  private data, running in its own process.

Everything is taught through one running story. Two agents — a **Buyer** and a
**Seller** — negotiate the sale of a house at 742 Evergreen Terrace, Austin TX. The
buyer wants the lowest price it can get and walks away above $460K. The seller has a
mortgage floor of $445K and will not go below it. Both pull live market data through
MCP to justify every offer, and the negotiation is capped at five rounds so it ends in
either a deal or a clean deadlock.

Keep returning to that story. Every demo is a piece of it.

---

## How the session runs

The rhythm is the same throughout: **teach an idea from the slides, then run the demo
that makes it concrete.** Nothing here is a lecture block followed by a lab block —
they alternate, and the demos are short.

The room needs the repo cloned and installed before the first demo. Give that its own
moment rather than assuming it happened; someone always has a Python version problem.

```
MCP
  the problem a protocol solves ......... slides
  the negotiation scenario .............. slides
  the handshake and the tool loop ....... learners run demos 01 and 02
  what else MCP carries ................. learners run demos 03 and 04
  MCP against a real service ............ you demo the GitHub agent, then SSE
  who is allowed to see what ............ slides, then the two servers
  what MCP leaves to you ................ slides

Google ADK and A2A
  why tools are not enough .............. slides
  the ADK building blocks ............... you demo d01 through d09
  MCP next to A2A ....................... slides
  the whole system running .............. you run the negotiation live
  the same thing across processes ....... you run it over A2A
```

---

## Section: MCP

### Opening the section, from the slides

Start with the question the slides open on: *what is the market price of this house
right now?* A model on its own cannot answer it. A real answer means reaching a
pricing feed, county tax records and a listings database — each with its own API, its
own auth, its own format.

Then the counting argument: three agents against four systems is twelve bespoke
integrations. With a shared protocol it is seven connections. That is the entire
case for MCP, and it is worth letting the room sit with the arithmetic.

Land the three roles before any code, because every demo refers back to them:

- **Host** — the AI application, which coordinates clients
- **Client** — holds one connection to one server
- **Server** — the program that actually provides tools and data

Then introduce the negotiation scenario properly. Do not rush this. Every demo from
here refers to the buyer, the seller, and the floor price.

### The learners' first two demos

Have the room run these themselves.

```bash
python d1_mcp/demos/01_initialize_handshake.py
```

Exactly five frames go past, and the last one lists two tools whose schemas were
generated from Python type hints. **Nobody wrote a JSON schema.** That is the argument
for the protocol in one screen. This demo needs no API key.

```bash
python d1_mcp/demos/02_tool_loop_trace.py
```

A timestamped trace across three actors — Model, Host, Server. Before you show the
numbers, ask the room where they think the time goes. Almost all of it is spent
waiting on the model, not on the server. This one needs `OPENAI_API_KEY`.

### Going deeper — the learners run these too

```bash
python d1_mcp/demos/03_list_all_primitives.py
python d1_mcp/demos/04_content_types.py
```

**Demo 03 is the one people miss.** Most of the room believes MCP means "tools."
It carries more than that: `inventory://floor-prices` is a **Resource**, readable data
fetched by URI, and `negotiation-tactics` is a **Prompt** template the host can
expand. This demo is where that misconception gets corrected, so give it a moment.

Demo 04 shows that a tool result does not have to be text — MCP standardises result
blocks so a tool can return an image or an embedded resource just as easily.

### MCP against something real — you drive these

```bash
python d1_mcp/github_agent_client.py     # needs GITHUB_TOKEN
python d1_mcp/sse_agent_client.py        # three terminals; see d1_mcp/README.md
```

The GitHub one lands hardest: one model, 20+ tools discovered from GitHub's own MCP
server, and not one line of integration code written for it.

The SSE one runs the same tool loop over a different transport. **Say that explicitly**
or it looks like a repeat of what they just watched. The point is that the transport
is pluggable and nothing above it changed.

### Who is allowed to see what

This is the most important idea in the section and the easiest to skip past.

The seller's floor price lives **in the MCP server** — not in a prompt, not in the
code. The buyer's agent is wired to `pricing_server` only. The seller's agent gets
`pricing_server` **and** `inventory_server`, and that second one is where
`get_minimum_acceptable_price` lives.

Ask the room directly: **what stops the buyer's model from just calling the secret
tool?**

Let them answer before you do. The answer is not "the prompt tells it not to." It is a
`before_tool_callback` allowlist — code the model cannot talk its way past. Policy
lives in code, not in prompting. That distinction comes back later in the ADK
callbacks demo, so plant it here.

### Closing the section

Walk the slides on what MCP does **not** solve: idempotency, cost accounting,
determinism, prompt injection, audit. The message is that MCP standardises the
protocol shape and everything else remains the host's job.

---

## Section: Google ADK and A2A

### Why tools are not enough

Open with the gap. MCP solved reaching data. It did not solve reaching **another
agent** — one with its own goals, its own private state, in its own process. Without
that, a human has to relay messages between the buyer and the seller.

A2A closes it: one agent discovers another over HTTP, learns what it can do from a
published **Agent Card**, and hands it work.

Make the framework point explicitly, because it is easy to miss: A2A is the wire
format. Whatever runs inside each agent is nobody else's business. A LangGraph agent
and a CrewAI agent can negotiate with each other.

### The nine building blocks — you drive these

```bash
cd d2_adk_multiagents
adk web adk_demos
```

All nine appear in one dropdown at `http://localhost:8000`. Work down them in order,
one line each — this is a tour, not nine deep dives.

| Demo | What it shows | The line to say |
|---|---|---|
| d01 | The simplest ADK agent | Model, instruction, tools. That is an agent |
| d02 | `MCPToolset` auto-discovery | The hand-written loop from the last section is now one line |
| d03 | Sessions and state | State does not leak between sessions |
| d04 | `SequentialAgent` | A's output becomes B's input via `output_key` |
| d05 | `ParallelAgent` | Independent work runs at the same time |
| d06 | `LoopAgent` | Bounded by `max_iterations` or an escalate signal |
| d07 | Agent-as-tool | One agent delegates to another |
| d08 | Callbacks | Allowlists and redaction the model cannot bypass |
| d09 | The event stream | What the Runner actually emits, event by event |

**Three of these carry the rest of the session** — `d02`, `d06` and `d08`. The full
negotiation is those three composed: MCP tools, a bounded loop, and a callback that
enforces policy. If the room is flagging, move quickly through the others and spend
your attention here.

Then put MCP and A2A side by side from the slides. Both standardise
discover-then-invoke over JSON-RPC. The difference is what is waiting at the other
end — a capability with no goals of its own, or a peer with its own private state.
That difference is why A2A needs Agent Cards and a `contextId` when MCP never did.

### The whole system, running

```bash
cd d2_adk_multiagents
adk web negotiation_agents
```

Pick **`negotiation`** from the dropdown and send **"Start negotiation"**.

**Watch the Events tab, not the chat.** The chat shows prose. The events show the
machinery, and the machinery is what you are teaching.

Narrate it as it happens:

1. The buyer calls `get_market_price` and `calculate_discount` **before** making an
   offer — the offer is grounded in data, not invented.
2. The seller calls `get_minimum_acceptable_price`, the tool the buyer cannot reach.
3. The seller replies with `submit_decision(COUNTER, ...)` — a **structured** call, not
   free text. The callback reads a dict, not a sentence.
4. The buyer moves, and eventually the seller accepts.
5. An `after_agent_callback` sets `escalate=True` and the loop stops.

At the end, open the **State** tab:

```json
{"seller_decision": {"action": "ACCEPT", "price": 445000}}
```

That dict is what ended the negotiation — a value that code checked, not a phrase
somebody found in a transcript. It is the single best thing to leave on screen.

**Two things to say out loud before you run it.** First, escalation stops the *next*
round, not the current one, so a few more messages appear after the acceptance — that
is correct behaviour, not a bug. Second, the run is not deterministic: the number of
rounds and the final price change between runs. The walk-away limits hold every time,
and that is the guarantee worth pointing at.

### The same negotiation, across processes

```bash
# Terminal 1
cd d2_adk_multiagents && adk web --a2a negotiation_agents

# Terminal 2
python d2_adk_multiagents/a2a_13_orchestrated_negotiation.py
```

Same negotiation, now between two separately-running agents talking over HTTP. The
in-process loop has been replaced by a script calling each agent's A2A endpoint.

Show the **Agent Card** first — it is plain JSON at a well-known URL, and it is how one
agent discovers what another can do without sharing any code.

If there is appetite, `a2a_10`, `a2a_11` and `a2a_12` break down the wire lifecycle,
context threading and artifacts individually.

---

## Where the learners take it further

Both demos ship exercises with complete, runnable solutions in their `exercises/`
and `solution/` folders. They are written for learners to attempt on their own, with
the instructor walking through the solutions afterwards.

A learner has understood this module when they can answer three questions:

1. Where does the seller's floor price live, and why not in the prompt?
2. What actually stopped the negotiation loop?
3. What changes when the same negotiation runs over A2A instead of in one process?
