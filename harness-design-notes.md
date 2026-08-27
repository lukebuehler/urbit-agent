# Harness on Urbit — working design notes

Working document behind the proposal. Unlike the proposal, this file *may* reference
Lightspeed, AOS, PRs, people and prior art. Nothing here goes into the pitch verbatim.

## 1. The thesis, sharpened

A frontier agent harness is a deterministic state machine whose only job is deciding
what goes into the next context window. Everything else — LLM turns, tool calls, shells,
sandboxes, web fetches — is I/O that happens elsewhere and comes back as a result.

That is Arvo's shape exactly: a pure function of the event log, effects out, results back
in as events. No other mainstream runtime has that shape natively; Lightspeed had to
borrow it from Temporal, AOS had to build it from scratch. Urbit has had it for a decade
and never had a workload that needed it. This one does.

Corollary for the pitch: **the harness is the first application for which Urbit's
complexity earns its keep.** Messaging never needed a solid-state interpreter; a
long-lived, forkable, replayable, self-modifying agent does.

## 2. What survived two harness projects (the invariants worth keeping)

Distilled from Lightspeed (harness on Temporal, enterprise) and AOS (whole runtime,
governed self-modification):

1. **Event-sourced core with no I/O.** State = replay(session log). The core replays,
   decides the next step, emits effect *intents*. It never blocks on the outside world.
2. **Closed event vocabulary.** Input admitted, turn planned, LLM requested/completed,
   tool calls observed, tool completed, compaction requested/completed, run
   started/idle/cancelled, config put. A small, fixed alphabet; products change around
   it, the core stays still.
3. **Thin boundary.** Intents and receipts carry refs plus *only the fields needed to
   branch* (stop reason, tool-call ids/names, usage, output ref). Provider-native
   payloads stay opaque and content-addressed. Never ship the transcript across the
   seam.
4. **Provider-native, not a universal message model.** Once a session starts its API
   kind is fixed. Compaction is provider-native and a first-class core event that marks
   the window pending until the result lands.
5. **Durable before dispatch.** Record the open work, *then* execute. Results re-enter
   only as admitted input. Nothing inside an executor waits on the core.
6. **Hands are borrowed.** Frontier models are RL-tuned for a POSIX box; the harness
   must not live on that box. Sandboxes, VMs, MCP servers are attached for exactly as
   long as a task needs them.
7. **Sub-agents are just sessions.** External controllers (bots, channels) never run
   inside a session; they admit events and receive emissions.
8. **Config as data, capabilities absent by default.** Toolset is derived from a
   config document; the default session can do nothing but talk.
9. **Prompt-cache stability.** The context prefix must be byte-stable turn to turn
   (catalogs append-with-supersede, breakpoints placed by the adapter).

What AOS over-built and must *not* be rebuilt on Urbit: a durable-execution engine
(Urbit *is* one), a custom control-plane IR (Hoon types + Clay + desks are the IR), a
WASM sandbox (Nock is the sandbox), signed receipts (the event log is the receipt).

## 3. Concept mapping

| Harness concept | Lightspeed / AOS | Urbit |
|---|---|---|
| Durable log | Postgres session log / SQLite journal | the ship's event log; the *session log* is a noun in agent state |
| Deterministic core | Rust `engine` crate driven by a Temporal workflow / WASM reducer | Hoon library + Gall agent (userspace) |
| Effect intent / receipt | activity call / `sys/*` effect + receipt | card out, sign/poke in; a versioned noun protocol |
| Effect executors | Temporal activities, adapters | vere drivers (Iris) or a sidecar over Lick; later a native vere LLM driver |
| CAS / blob offload | Postgres + S3 CAS, blob refs in history | blob store (vere PR #985) + vere64 loom; Hoon-level `(map hash atom)` |
| Fork / clone | by-reference log fork | noun structural sharing — O(1), free |
| Timers, schedules | Temporal Schedules | Behn |
| Web UI, webhooks | JSON-RPC gateway | Eyre |
| Chat channels | Channels app over managed sessions | Tlon Messenger channels (Ames) |
| Bot ↔ bot federation | events through admission, `hops`, allowlists | pokes between ships; identity and encryption are free |
| Sub-agents | more sessions, `SubagentExecutionWorkflow` | more sessions; moons for separate identity |
| Secrets | AEAD store, resolved at provider-send time | live in the runtime/sidecar, **never** in Arvo state |
| Self-modification | AIR patches, propose → shadow → approve → apply | desks + Clay + hot reload (`+on-load`); shadow = fork the noun, run against new code |
| Skills, catalogs | VFS catalogs | desk files, published in the scry namespace |
| Tenancy | universes | one ship per person; moons per agent |
| Compute | Incus VMs, env daemon, jobs | borrowed: sandbox via runner; a friend's GPU over Ames (White Marble pattern) |

## 4. Architecture sketch

```
                      ┌────────────────────────────── ship (Mars) ──────────────────────────────┐
  Eyre  (web UI,      │  %harness Gall agent                                                    │
  webhooks) ─────────▶│   ┌───────────────────────┐   intents (refs only)   ┌────────────────┐ │
  Ames  (other ships, │   │ session log (noun)    │ ───────────────────────▶│ effect outbox  │ │
  Tlon channels) ────▶│   │ core FSM / decider    │                         │ (open work)    │ │
  Behn  (schedules) ─▶│   │ turn planner          │ ◀───────────────────────│                │ │
  Lick/Khan (CLI) ───▶│   │ config, catalogs      │   receipts (refs +      └───────┬────────┘ │
                      │   └───────────────────────┘    branch fields)               │          │
                      │   CAS: (map hash atom) — big atoms are bob atoms (off-loom) │          │
                      └──────────────────────────────────────────────────────────────┼──────────┘
                                                                                     │ Lick socket
                                                                                     │ (v0) / vere
                                                                                     │ driver (v1)
                      ┌────────────────────────────── runtime (Earth) ───────────────▼──────────┐
                      │  effect runner: provider clients (Anthropic/OpenAI/OpenRouter/local),   │
                      │  streaming to UI, API keys, request assembly from refs (cache-stable),  │
                      │  tool dispatch → sandbox / VM / MCP / web; writes payloads as blobs     │
                      └─────────────────────────────────────────────────────────────────────────┘
```

Key property: the runtime is *dumb and replaceable*. The protocol between the two boxes
is the real artifact. A Tlon-hosted runner, a laptop sidecar, a native vere driver, and a
remote GPU box are all the same thing from Mars' point of view.

## 5. Why vere64 and blobs are the precondition

Numbers (refining the raw draft): 1M-token window cycled hourly ≈ 24M tokens/day ≈
100 MB/day *through* the window. But most of that is the re-sent cacheable prefix. New
bytes the ship must actually retain per day (inputs, outputs, tool results) are more like
10–30 MB for an intense agent. That's still **5–10 GB/year, tens of GB over a
multi-year agent** — and the whole point is an agent that lives for years.

Today: 32-bit vere, loom 2 GB default / 8 GB max. A year of memory doesn't fit; `|pack`,
`|meld`, snapshots all scale with loom size. Dead on arrival.

With the two open runtime PRs:

- **vere64** (urbit/vere#970, `-Dvere64`, `c3_w` becomes 64-bit): loom up to 2^46 bytes.
  Functionally unlimited. Benchmarks show parity with 32-bit.
- **Blob store** (urbit/vere#985, "noun: blob storage for large atoms"): atoms over
  `U3_BLOB_THRESH` (32 MiB) live as content-addressed files under `.urb/bob/` and sit on
  the loom as *bob atoms* (indirect atoms pointing at small metadata). Jam/cue, the event
  log, IPC (`ram`/`tap` wire format), Clay sync, HTTP bodies, and Mesa all handle them
  transparently; bytes are `mmap`ed only when a jet needs them. Event-log refcounts +
  durable leases + boot-time GC. Earth is the sole writer; Mars is read-only.
  Future work listed in the PR: a `%blob` Nock hint, automatic blobification after each
  poke, a blobify sweep in `|pack`.

Together they make "keep the agent's entire provider-native history on-ship" viable:
payloads enter as blob refs, the event log stays small, the loom stays small, the pier
stays portable. **The harness is the concrete consumer that justifies prioritizing both.**

Two things to raise with the runtime authors:

1. The 32 MiB threshold is far above typical LLM payloads (KB to low MB). For this
   workload we want either a much lower threshold, the `%blob` hint, or the post-poke
   detector — otherwise transcripts stay loom-resident (tolerable with vere64, but
   snapshots/pack get heavy).
2. Lick isn't in the PR's list of blob-aware I/O (HTTP, Unix, Mesa are). If v0 uses a
   Lick sidecar, inbound results over threshold should land as blobs too.

## 6. Design decisions I'd commit to

1. **Userspace first.** A Hoon library (`/lib/harness`) + a Gall agent, not a vane.
   Upgradable via desks, no kernel politics, ships faster. A vane is a later question if
   ever.
2. **The effect protocol is the product.** Versioned nouns: `%llm-generate`,
   `%llm-compact`, `%llm-count`, `%tool-exec` (sandbox/env/mcp/web), `%blob-put/get`.
   Receipts: status + refs + branch fields. Behn covers timers natively.
3. **Earth parses, Mars branches.** Provider protocol knowledge (JSON, SSE streaming,
   retries, cache breakpoints) lives in the runtime, exactly as TLS/HTTP/Ames crypto
   already do. Nock string handling is slow and each event is a blocking transaction;
   keep per-event work small. Mars receives pre-digested nouns.
4. **Refs, not bytes, cross the seam.** The runtime assembles the provider request from
   blob refs so the prefix stays byte-identical turn to turn (prompt caching). Mars never
   re-sends the transcript.
5. **Session log is a legible, forkable noun, exposed via scry.** Any front-end (Tlon,
   terminal, Hawk, a Context-Lens-style inspector) renders from the namespace. Fork =
   share the prefix. Branching, "what if", shadow runs are free.
6. **Core loop is kernel-grade and fixed.** Self-modification targets tools, skills,
   prompts, policies, and config — all desk files — through a governed loop: agent
   writes to a staging desk → shadow run on a forked session → user approves → commit.
   Failed events roll back for free. Do *not* let the agent edit its own decider.
7. **Keys never in Arvo.** API keys and OAuth tokens live in the runtime and are resolved
   at send time; the event log must never contain a secret.
8. **Streaming bypasses the log.** Tokens stream runtime → UI; only terminal results
   become events. Otherwise the event log bloats with partial frames.
9. **Capabilities absent by default.** A session is a model that can process runs; every
   tool family is an explicit grant in config.

## 7. Where the ecosystem already is (August 2026)

- **Tlonbot** (Tlon): every Messenger account gets an OpenClaw-powered agent on a moon;
  runs as "a sidecar service alongside your hosted Tlon planet that bridges to your
  agent's OpenClaw instance." Memory, keys, relationships currently on Tlon's servers;
  stated direction is user-controlled infra. **The harness is outside Urbit; Urbit is
  identity + channel.** Our proposal inverts this: the harness lives on the ship, the
  sidecar becomes a dumb executor. This is the path to what Tlon already says it wants.
- **Groundwire / ~niblyx-malnus, "LLMs on Urbit" (Jan 2026):** `urbit-master` desk —
  Claude chat with conversation branching, MCP server exposing ship tools, `sailbox`
  fibers, alarms, open-loops, "state should be legible and portable as conventional
  files." An on-ship agent loop exists and validates the direction; it's an app, not a
  kernel-grade harness with a runtime seam and blob offloading. Adopt its legibility
  principle.
- **White Marble Syndicate:** `%privateer` (membership + data) and `%computeer`
  (inference gate) — **Ames as a compute network**, pointing your ship at a provider's
  Urbit identity instead of a public endpoint. Directly reusable as "borrowed compute
  over Ames."
- **Dalten / ~sicdev-pilnup:** "Urbit should be the identity and memory layer for AI,
  not its compute substrate"; "nearly a perfect substrate for an independent agent."
  Same thesis, community voice.
- **Tlon July 2026:** Notebooks redesigned as Markdown "enhancing AI agent
  compatibility"; **Context Lens** shows agent tool uses and provider info. Tlon is
  already reshaping content and UI for agents.
- **urbit-skills alpha, Yamoon, Lua-Hoon, nockasm:** agents can write Hoon (or compile
  to it). Matters for self-authored tools.
- **Directed Messaging (408k):** content-centric, scry-based networking — the shape of
  "an agent requests named data from a peer."

Nobody has built the harness *as a kernel-grade component with the event-sourced /
thin-effect / CAS discipline*, and nobody has tied it to the runtime work that makes
multi-year memory feasible. That's the gap.

## 8. Risks and open questions

- **Nock cost for text.** JSON/string processing in Hoon is slow. Mitigation: runtime
  parses, Mars branches on small nouns. Needs a performance budget per event.
- **Single-threaded ship.** A heavy event blocks everything (including Messenger).
  Bound per-event work; large results arrive as refs.
- **Iris limits** (timeouts, streaming, large bodies) — probable reason to go
  sidecar/driver rather than Iris for LLM calls. Verify current state.
- **Event-log growth vs `|chop`.** Session deletion must drop blob refs so GC can
  reclaim; chop semantics with long-lived sessions need thought.
- **Blob threshold** (see §5).
- **Hosting memory footprint** of vere64 looms; blobs + mmap help, needs measurement.
- **Politics.** Vane vs userspace; Tlon's OpenClaw investment; core-team bandwidth
  (both PRs still open). The proposal should give the core team a *reason* to land them.
- **Product gap.** A harness isn't a consumer product. Positioning: UF/core ship the
  kernel-grade harness + runtime; Tlon ships the surface (and migrates Tlonbot onto it);
  community ships tools/skills desks.

## 9. Phasing (draft)

- **Phase 0 — protocol + prototype.** Spec the session log and effect protocol as
  nouns. Gall agent + Rust sidecar over Lick (provider clients can be lifted from
  existing Rust code). Terminal chat, forking, compaction. Runs on 32-bit vere with
  short sessions.
- **Phase 1 — durable memory.** vere64 + blobs land; runtime writes payloads as blobs;
  lower threshold / `%blob` hint. Sessions that live for months. Tlon Messenger as a
  channel; Context-Lens-style inspector reads the scry namespace.
- **Phase 2 — hands and society.** Borrowed compute (sandbox exec), MCP, sub-agents,
  schedules/webhooks/bot layer, agent-to-agent over Ames, skills desks, governed
  self-modification loop.
- **Phase 3 — native.** Optional vere LLM driver, hosting economics, marketplace of
  tools/skills desks, Tlonbot fully on-ship.

## 10. Notes on the raw draft

- Keep the opening question ("what is Urbit for?") and the one-line answer.
- Soften the "messaging failed" passage: Tlon just launched publicly and is a key
  audience. Reframe as *messaging didn't need Urbit; an agent does* — messaging was the
  on-ramp, the agent is the reason. Same argument, no dunking.
- Replace the 100 MB/day estimate with the retained-bytes framing (§5): the point is
  years of memory, not a day of throughput — that's what makes vere64/blobs load-bearing.
- The "Pi is just a TypeScript app with hot reloading" point is strong; pair it with the
  guardrail (§6.6): ambition in *what* the agent may rewrite, discipline in the core.
- Add the "hands vs brain" split explicitly; it's now industry consensus (OpenAI's
  "separate harness from compute", Anthropic's "decoupling the brain from the hands",
  April 2026) — Urbit is the only substrate that *is* the brain's shape.
