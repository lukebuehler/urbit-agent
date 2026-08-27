# Harness on Urbit: working design notes

These notes accompany the proposal in [README.md](README.md). They are more technical and more opinionated than the proposal, and they reference prior work freely. They describe *one* way this could work in order to show that it can, not the way it should be built. The actual design belongs to the people who know Arvo and Vere best.

Vocabulary follows the proposal: the **head** is the agent loop (the state machine that decides what goes into the next context window), the **hands** are everything that touches the world (LLM providers, sandboxes, shells, MCP servers, APIs). Where runtime internals come up, **Mars** is the serf (Nock and Arvo) and **Earth** is the king (the I/O side of Vere).

## 1. The thesis, sharpened

A frontier agent harness is a deterministic state machine whose only job is deciding what goes into the next context window. Everything else, LLM turns, tool calls, shells, sandboxes, web fetches, is I/O that happens elsewhere and comes back as a result.

That is Arvo's shape: a pure function of the event log, effects out, results back in as events. No mainstream runtime has that shape natively. Durable workflow engines such as Temporal approximate it by replaying workflow code against a recorded history. Urbit has had it for a decade and never had a workload that needed it. An agent loop does.

Corollary for the pitch: the harness is the first application for which Urbit's complexity earns its keep. Messaging never needed a solid state interpreter; a long-lived, forkable, replayable, self-modifying agent does.

## 2. What survived two harness projects

I have built this shape twice outside of Urbit: [Lightspeed](https://github.com/smartcomputer-ai/lightspeed), a Rust harness that runs agents as durable workflows on Temporal, and [AgentOS](https://github.com/smartcomputer-ai/agent-os), a deterministic, event-sourced runtime with governed self-modification. The invariants that survived both:

1. **An event-sourced core with no I/O.** State is a replay of the session log. The core replays, decides the next step, and emits effect *intents*. It never blocks on the outside world.
2. **A closed event vocabulary.** Input admitted, turn planned, LLM requested and completed, tool calls observed, tool completed, compaction requested and completed, run started, idle, cancelled, config replaced. A small fixed alphabet; products change around it while the core stays still.
3. **A thin boundary.** Intents and receipts carry references plus only the fields needed to branch: stop reason, tool call ids and names, token usage, an output reference. Provider-native payloads stay opaque and content-addressed. The transcript never crosses the seam.
4. **Provider-native, not a universal message model.** Once a session starts, its API kind is fixed. Compaction is provider-native and a first-class core event that marks the window pending until the result lands.
5. **Durable before dispatch.** Record the open work, then execute. Results re-enter only as admitted input. Nothing inside an executor waits on the core.
6. **The hands are borrowed.** Frontier models are trained for a POSIX box; the head must not live on that box. Sandboxes, VMs and MCP servers are attached for exactly as long as a task needs them.
7. **Sub-agents are just sessions.** External controllers (bots, channels, schedulers) never run inside a session; they admit events and receive emissions.
8. **Config is data, capabilities are absent by default.** The toolset is derived from a config document; the default session can do nothing but talk.
9. **Prompt-cache stability.** The context prefix must be byte-stable turn to turn, so anything that changes mid-context is appended with a supersede marker rather than edited in place.

What AgentOS built that must *not* be rebuilt on Urbit: a durable-execution engine (Urbit is one), a custom control-plane IR (Hoon types, Clay and desks are the IR), a WASM sandbox (Nock is the sandbox), signed effect receipts (the event log is the receipt).

## 3. Concept mapping

| Harness concept | Outside Urbit | On Urbit |
|---|---|---|
| Durable log | a session log in Postgres, or a workflow history | the ship's event log; the *session log* is a noun in agent state |
| Deterministic core | a Rust engine driven by a workflow, or a WASM reducer | a Hoon library plus a Gall agent, in userspace |
| Effect intent and receipt | activity call, effect and receipt records | cards out; signs and pokes in; a versioned noun protocol |
| Effect executors | workflow activities, adapters | Vere's I/O drivers (the HTTP client behind Iris), or an external process over Lick; possibly a dedicated driver later |
| Blob offload | Postgres plus S3, blob refs in history | the blob store (vere #985) on a vere64 loom; a Hoon-level `(map hash atom)` |
| Fork and clone | by-reference log fork, built explicitly | keep two nouns; they share structure and are durable at the end of the event (head only, see §6) |
| Timers and schedules | Temporal Schedules, cron | Behn |
| Web UI and webhooks | a JSON-RPC gateway | Eyre |
| Chat channels | a channels app over managed sessions | Tlon Messenger over Ames |
| Agent to agent | events through an admission pipeline, allowlists, hop limits | pokes between ships; identity and encryption come with the network |
| Sub-agents | more sessions, a supervising workflow | more sessions; moons where a separate identity is wanted |
| Secrets | an encrypted store, resolved at provider-send time | on the Earth side only, never in Arvo state (the pier and the event log are plaintext) |
| Self-modification | typed patches, propose then shadow then approve then apply | desks, Clay builds, `+on-load` migrations; a rehearsal loop the harness has to provide (§6) |
| Skills and catalogs | files in a virtual filesystem | desk files, readable through the scry namespace |
| Tenancy | universes, tenants | one ship per person; moons per agent |
| Borrowed compute | VMs from a provider, an environment daemon, jobs | sandboxes via the executor; a peer's GPU over Ames (the White Marble pattern) |

## 4. One possible shape

```
                      +------------------------------ ship (Mars) ------------------------------+
  Eyre  (web UI,      |  harness Gall agent                                                     |
  webhooks) --------->|   +-----------------------+   intents (refs only)   +----------------+  |
  Ames  (other ships, |   | session log (noun)    | ----------------------->| open work      |  |
  Tlon channels) ---->|   | core state machine    |                         | (pending       |  |
  Behn  (schedules) ->|   | turn planner          | <-----------------------|  effects)      |  |
  Lick / Khan (CLI) ->|   | config, catalogs      |   receipts (refs +      +-------+--------+  |
                      |   +-----------------------+    branch fields)               |           |
                      |   payload store: (map hash atom); big atoms are blobs       |           |
                      +----------------------------------------------------------+--+-----------+
                                                                                 |
                                                  Iris / Lick today; a dedicated driver later
                                                                                 |
                      +------------------------------ runtime (Earth) -----------v--------------+
                      |  executor: provider clients, streaming to the UI, API keys, request   |
                      |  assembly from refs (cache-stable), tool dispatch to sandbox / VM /   |
                      |  MCP / web; stores payloads, hands back references                    |
                      +-----------------------------------------------------------------------+
```

The executor is deliberately dumb and replaceable. The protocol between the two boxes is the real artifact: a Tlon-hosted executor, a laptop sidecar, a native Vere driver and a remote GPU box are all the same thing from Mars' point of view. For a first version, plain HTTP calls through Iris are enough to prove the loop.

## 5. Why vere64 and the blob store are the precondition

Rough numbers, to be verified. A 1M-token window cycled roughly hourly is on the order of 24M tokens a day, or about 100 MB a day *through* the window. Most of that is the re-sent, cacheable prefix. The new bytes the ship has to retain (inputs, outputs, tool results) are more like 10 to 30 MB a day for an intensely used agent. That is still several gigabytes a year, and the whole point is an agent that lives for years.

Today's 32-bit Vere has a 2 GB default loom and an 8 GB maximum. A year of memory does not fit, and `|pack`, `|meld` and snapshots all scale with loom size.

Two open runtime PRs change this:

- **vere64** ([urbit/vere#970](https://github.com/urbit/vere/pull/970)). `c3_w` becomes 64-bit when built with `-Dvere64`, which lifts the loom ceiling to 2^46 bytes: functionally unlimited. The PR's benchmarks are at parity with 32-bit.
- **Blob storage for large atoms** ([urbit/vere#985](https://github.com/urbit/vere/pull/985)). Atoms over a threshold (32 MiB in the PR) are stored as content-addressed files under `.urb/bob/` and sit on the loom as *bob atoms*, indirect atoms pointing at small metadata. Bytes are `mmap`ed only when a jet needs them; the hash and byte-level jets read from the file directly. `jam` and `cue` are unchanged (`|cram` expands blob bytes so rocks stay portable); the event log and king/serf IPC use a new `ram`/`tap` encoding that carries blob references instead of bytes. Blob lifetime is tracked by event-log refcounts and durable leases, with GC at boot and on `|chop`. Earth is the sole writer, Mars reads. Inbound HTTP bodies, Clay syncs and Mesa frames over the threshold go straight to the store. Listed as future work in the PR: a `%blob` Nock hint, automatic blobification after each event, and a blobify sweep in `|pack`.

Together they make "keep the agent's entire provider-native history on-ship" viable: payloads enter as references, the event log and the loom stay small, the pier stays portable. The harness is a concrete consumer that justifies landing both.

Two details worth raising with the runtime work:

1. The 32 MiB threshold is far above typical LLM payloads (kilobytes to a few megabytes). This workload wants a lower threshold, the `%blob` hint, or the post-event detector; otherwise transcripts stay loom-resident, which is tolerable on vere64 but makes snapshots and `|pack` heavier than they need to be.
2. The PR makes the HTTP, Unix and Mesa drivers blob-aware. Lick is not mentioned. If an early version uses an external executor over Lick, results over the threshold should land as blobs on that path too.

## 6. Design directions I would argue for

1. **Userspace first.** A Hoon library plus a Gall agent, not a vane. Upgradable through desks, no kernel changes, ships faster. Whether any of it should ever move into the kernel is a later question.
2. **The effect protocol is the product.** Versioned nouns for "generate a turn", "compact", "count tokens", "execute a tool" (sandbox, environment, MCP, web), "store and fetch a payload". Receipts carry status, references and the few branch fields. Timers are Behn's job already.
3. **Earth parses, Mars branches.** Provider protocol knowledge (JSON, server-sent events, retries, cache breakpoints) lives on the runtime side, exactly as TLS, HTTP framing and Ames cryptography already do. Nock is slow at text and every event is a blocking transaction for the whole ship, so per-event work must stay small. Mars receives pre-digested nouns.
4. **References, not bytes, cross the seam.** The executor assembles the provider request from references so the prefix stays byte-identical turn to turn (prompt caching). Mars never re-sends the transcript.
5. **The session log is a legible noun, exposed through the scry namespace.** Any front-end (Tlon, a terminal, a Context-Lens-style inspector) renders from the namespace rather than from a private API.
6. **Forking is a head-only property.** A session's state is one noun; keeping two of them shares everything up to the fork, and both are durable when the event completes, with no serialization step. That is a genuine advantage over harnesses that keep sessions in a database. It does not extend to the hands: a sandbox mid-task is not part of the noun. Rehearsing a change on a fork also needs a rule for effects issued from the fork (stub them, or run them and pay).
7. **The core loop stays fixed; self-modification targets desks.** Tools, skills, prompts, policies and config are files. The rehearsal loop (write to a staging desk, build it, run a copy of the session against it, approve, commit) is something the harness has to provide: Clay can build code from any desk on demand and Gall reloads agents on commit with `+on-load` for state migration, but running a *new* version of the loop against a *copy* of state requires the core to be a pure library the agent can call with either version. What Arvo gives for free is the other half: an event that crashes leaves no trace.
8. **Keys never enter Arvo.** API keys and OAuth tokens live on the Earth side and are resolved at send time. The event log must never contain a secret.
9. **Streaming bypasses the log.** Tokens stream from the executor to the UI; only terminal results become events. Partial frames as events would bloat the log for no gain.
10. **Capabilities are absent by default.** A session is a model that can process turns; every tool family is an explicit grant in config.

## 7. Where the ecosystem already is (August 2026)

- **Tlonbot** ([tlon.io/posts/tlonbot](https://tlon.io/posts/tlonbot)). Every Tlon Messenger account gets an OpenClaw-powered agent on a moon, run as "a sidecar service alongside your hosted Tlon planet that bridges to your agent's OpenClaw instance". Memory, keys and relationships are on Tlon's servers for now; the stated direction is user-controlled infrastructure. The head is outside Urbit; Urbit provides identity and a channel. The proposal inverts this: the head moves onto the ship and the sidecar becomes a replaceable executor, which is the path to what Tlon already says it wants.
- **"LLMs on Urbit"** ([urbit.org/blog/llms-on-urbit](https://urbit.org/blog/llms-on-urbit), Groundwire, January 2026). An experimental desk with a Claude chat interface and conversation branching, an MCP server exposing ship tools, an async wrapper for agents, alarms and open loops, and the principle that "state should be legible and portable as conventional files and directories". An on-ship agent loop exists and validates the direction. It is an application rather than a kernel-grade harness with a runtime seam and payload offloading; its legibility principle is worth adopting.
- **White Marble** ([urbit.org/blog/building-white-marble](https://urbit.org/blog/building-white-marble)). `%privateer` for membership and data, `%computeer` gating an inference engine, connected over Ames: a ship points at a provider's Urbit identity instead of a public endpoint. Directly reusable as borrowed compute over Ames.
- **~sicdev-pilnup** ([contributor spotlight](https://urbit.org/blog/contributor-spotlight-sicdev-pilnup)): Urbit as "the identity and memory layer for AI systems, not their compute substrate", and "nearly a perfect substrate for an independent agent". The same thesis, in a community voice.
- **Tlon, July 2026** ([This Month in Urbit](https://urbit.org/blog/this-month-in-urbit-july-2026)). Notebooks redesigned as Markdown for "AI agent compatibility"; Context Lens shows an agent's tool uses and provider details. Tlon is already reshaping content and UI for agents.
- **Urbit skills alpha, Yamoon, Lua-Hoon, nockasm** ([June 2026](https://urbit.org/blog/this-month-in-urbit-june-2026)). Agents can write Hoon, or compile to it. This matters for self-authored tools.
- **Directed Messaging** (408k, July 2026). Content-centric, scry-based networking: the shape of "an agent requests named data from a peer".

Nobody has built the head as a kernel-grade component with the event-sourced, thin-effect, offloaded-payload discipline, and nobody has tied it to the runtime work that makes multi-year memory feasible. That is the gap.

## 8. Risks and open questions

- **Nock and text.** JSON and string processing in Hoon is slow. Mitigation is §6.3, plus a per-event compute budget.
- **One ship, one thread.** A heavy event blocks everything, Messenger included. Bound per-event work; large results arrive as references.
- **Iris in practice.** Timeouts, large bodies and streaming behaviour of the HTTP client vane and its driver need checking for this workload; this is the likely reason to move to an external executor or a dedicated driver after the first version.
- **Event-log growth and `|chop`.** Deleting a session must drop its blob references so GC can reclaim them; `|chop` semantics with very long-lived sessions need thought.
- **Blob threshold** (§5).
- **Hosted memory footprint** of vere64 looms; blobs and `mmap` help, but it needs measuring.
- **Sequencing.** Both runtime PRs are open. The proposal should give the core team a concrete reason to land them, not another abstract wish.
- **Product gap.** A harness is not a consumer product. The Foundation and core team ship the head and the runtime work; Tlon ships the surface and migrates Tlonbot onto it over time; the community ships tools and skills as desks.

## 9. Phasing (sketch)

- **Phase 0: protocol and prototype.** Specify the session log and the effect protocol as nouns. A Gall agent, LLM calls through Iris, a terminal to talk to it. Long-lived sessions, forking and compaction are the things to prove. Runs on today's 32-bit Vere with short sessions.
- **Phase 1: durable memory.** vere64 and the blob store land; payloads arrive as blobs; threshold or hint tuned for this workload. Sessions that live for months. Tlon Messenger as a channel; an inspector that reads the scry namespace.
- **Phase 2: hands and society.** An external executor (sandboxes, MCP), sub-agents, schedules and webhooks, agent-to-agent over Ames, skills desks, and the governed self-modification loop.
- **Phase 3: native.** A dedicated Vere driver if warranted, hosting economics, a marketplace of tools and skills desks, Tlonbot fully on-ship.

## References

- Lightspeed: [github.com/smartcomputer-ai/lightspeed](https://github.com/smartcomputer-ai/lightspeed)
- AgentOS: [github.com/smartcomputer-ai/agent-os](https://github.com/smartcomputer-ai/agent-os)
- vere64: [urbit/vere#970](https://github.com/urbit/vere/pull/970)
- Blob storage for large atoms: [urbit/vere#985](https://github.com/urbit/vere/pull/985)
- OpenAI on separating harness from compute: [The next evolution of the Agents SDK](https://openai.com/index/the-next-evolution-of-the-agents-sdk/)
- Anthropic on decoupling the brain from the hands: [Scaling Managed Agents](https://www.anthropic.com/engineering/managed-agents)
- Tlonbot: [tlon.io/posts/tlonbot](https://tlon.io/posts/tlonbot)
- LLMs on Urbit: [urbit.org/blog/llms-on-urbit](https://urbit.org/blog/llms-on-urbit)
- Building White Marble: [urbit.org/blog/building-white-marble](https://urbit.org/blog/building-white-marble)
