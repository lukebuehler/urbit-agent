# First milestone: a complete, durable coding session on Urbit

*Discussion draft — 7 September 2026*

The first implementation goal should be a complete coding-session harness running on Urbit, with a real Linux environment attached. The agent should be able to inspect a repository, edit files, run commands and tests, recover from ordinary failures, manage a long context, and finish a task. Its session should survive client disconnects and ship restarts.

Terminal-Bench 2.1 is the capability target. Recovery, cancellation, compaction, and fork tests are the durability target. Both are necessary: a benchmark score does not establish that a session is durable, and durable state alone does not make an effective coding agent.

> **Proposed milestone: Durable coding-session v1.** One ship, one provider, one attached Linux environment, a terminal interface, measured Terminal-Bench performance, and tested recovery semantics.

This document develops the [original proposal](README.md) and [working design notes](harness-design-notes.md), using [urbit-agent-harness][harness] as the current implementation and [Lightspeed][lightspeed] as the architectural reference. It proposes a narrower implementation sequence: bring the external executor and coding tools forward, finish the session contract, and expand the integration surface afterward.

## 1. Scope and evidence

The assessment below is based on source inspection of:

- `urbit-agent-harness` at `49d19cb7ba1a29b3462b38092ee03f86c316edcc`.
- Lightspeed at `8d23c80d165fd2912a8be5bcc786074c15c2d706`.
- The proposal and design notes in this repository.

Source links are pinned to those revisions. “Present” means the code implements the described path; it does not mean that recovery or benchmark behavior was independently exercised during this review. Prototype demonstrations reported in the harness README were not rerun. No Terminal-Bench results were verified, so Lightspeed/Codex parity is a comparison target rather than an established result of this assessment. The inspected Lightspeed [evaluation plan][ls-evaluation] places its Harbor adapter in a separate repository.

## 2. Finish the head and give it ordinary hands

The head owns session state, input admission, context selection, the next-step decision, and outstanding work. The hands execute model requests and tools. This follows the existing proposal without requiring a new Urbit kernel component.

Keep a thin Gall agent as the host of a pure Hoon session library. Defer tools that interact with Gall applications, Clay, Messenger, and other ships. Hosting the loop in Gall and integrating the agent with the wider Gall ecosystem are separate pieces of work.

For the first version, attach an existing Linux container or VM. It needs a real shell, filesystem, processes, development tools, and whatever network access the task permits. It does not need a custom virtual filesystem, an environment marketplace, or a provisioning fleet.

The executor should own provider JSON, request assembly, credentials, streaming, process execution, and bulk output. Hoon should receive the facts required to make decisions and references to retained payloads. This keeps each ship event small and keeps the executor replaceable.

```text
Terminal / client
       |
       v
Gall host + pure Hoon session core
  history, runs, context, pending effects
       |                         ^
       | intents: IDs + refs     | receipts: status + refs
       v                         |
External executor + retained payload store
       |                         |
       v                         v
Model provider           Attached Linux environment
```

The current [harness roadmap][h-roadmap] places the external executor at item 10, behind streaming, another provider, channels, and self-authored tools. For the coding milestone, the executor belongs in the first implementation slice. HTTP through Iris can carry the protocol initially; choosing Lick or a dedicated runtime driver need not block it.

## 3. What urbit-agent-harness already has

The repo already contains the basic shape of the proposed head and several integrations beyond it. The work should build on that foundation while tightening its semantics.

| Area | Present in the inspected implementation | Still needed for coding-session v1 |
|---|---|---|
| Deterministic core | A closed event vocabulary, `play` to derive a view, and `decide` to select turns, tools, compaction, or halt. | Explicit runs and turns, stronger admission invariants, and bounded work per event. [Types][h-types], [core][h-core]. |
| Persistence | Sessions and related state are saved by Gall; version 0 migrates to version 1. | Recovery tests covering outstanding effects, and failure without data loss for unknown state versions. [State loading][h-load]. |
| Model calls | OpenAI-compatible Chat Completions through Iris; request IDs, completion/failure events, and stale-response checks. | A faithful adapter for the selected coding model, preservation of native continuation data, retry policy, and provider processing outside Hoon. [LLM dispatch][h-llm]. |
| Compaction | A model-generated summary plus a retained recent tail that tries not to split tool flows; manual and automatic triggers. | Native compaction where supported, context revisions, generation headroom, overflow recovery, and repeated-compaction tests. [Context handling][h-core], [request assembly][h-request]. |
| Tool loop | Multiple calls per assistant response; synchronous results and asynchronous pending-call tracking; errors returned to the model. | Typed outcomes, execution-time grant checks, explicit scheduling policy, and general effect reconciliation. [Tool dispatch][h-dispatch]. |
| Coding environment | `run_js` executes through the on-ship WASM/Spider runtime; Clay tools read desk files. | A real external Linux filesystem and shell, editable repository files, process handles, PTY support, and process control. [Tool inventory][h-readme]. |
| Run control | Send, cancel, retry, and configuration actions. | Distinct steer/queue/cancel operations, input-consumption tracking, recorded terminal states, and cancellation delivered to external work. [Actions][h-actions]. |
| Forking | Copies a session into another session ID, filtering request markers out of the copied history. | An explicit fork boundary and lineage, preserved historical facts, and a defined policy for the attached environment. [Fork handler][h-actions]. |
| Limits | Four consecutive tool errors or 24 assistant turns trigger a halt; rough context estimate; JS watchdog. | Configurable run budgets, structured failure classes, bounded provider retries, and counters independent of compactable context. [Loop guards][h-guards]. |
| Inspection and clients | Web chat, scries, event subscriptions, and an ACP bridge with session loading and item-level updates. | Durable event cursors, reconnect reconciliation, authoritative run status, and retained large-output retrieval. Token streaming follows. [README][h-readme], [ACP][h-acp]. |
| Configuration and credentials | Config is data; tool families select advertised schemas; agent-level default key with per-session override. | Revision pinning, enforced capabilities, and credentials held outside Arvo. The current default key still resides in ship state. [Types][h-types], [dispatch][h-dispatch], [LLM dispatch][h-llm]. |
| Subagents | `run_subagent` creates a child session and returns its answer to the parent; ordinary children are kept. | General supervision, budgets, deadlines, joins, and cancellation propagation. Expand after the single-agent contract is reliable. [Child settlement][h-subagents]. |
| Wider integrations | Skills in state, proposal/rehearsal/commit actions, peer asks over Ames, Behn timers, and webhooks. | These are existing extensions to maintain, but further expansion is outside this milestone. [README][h-readme]. |

### Important semantic gaps

**A run is currently inferred from the transcript.** The session contains a log and request counter; its view contains messages, a pending LLM request, pending tool IDs, usage, and an error. There is no explicit run ID, turn ID, input-consumption marker, or completed-run record.

This matters when new input arrives during generation. `%send` immediately appends a user item. If the already-running request then returns a final assistant message without tool calls, `decide` sees an assistant item last and goes idle. The newly admitted input can remain unconsumed by any model request. A regression test should exercise precisely this sequence. Lightspeed has an explicit [test and rule for unconsumed steering][ls-steering].

**Cancellation rewrites the application session history.** The current handler removes all `%llm-requested` and `%tool-requested` events. It does not append a cancellation event or send cancellation cards to the outstanding work. Stale Iris results are ignored after their pending markers disappear, which is useful, but this is not a complete cancellation lifecycle. The distinction concerns the harness's session history, not whether Arvo retains its own event log.

**Compaction is a useful prototype, with incomplete context semantics.** It requests a plain-text summary through the ordinary chat endpoint and keeps recent items. Token estimation serializes the whole request and divides its byte length by four. Pending compaction does not have a frozen context revision. The next version needs explicit handling of input arriving during compaction, insufficient remaining capacity, and failed or stale compaction results.

**The fixed loop guards are unsuitable as the general coding budget.** Four failed commands can be a normal part of debugging. A productive task can also require more than 24 turns. The counters are derived from active conversation items, so compaction can change their meaning. Budget accounting belongs in run state.

**Tool grants need enforcement at dispatch.** The selected tool families determine which definitions are sent to the model, but the inspected dispatcher selects implementations by returned tool name without checking the session's grant list. Withholding a schema is not sufficient enforcement. The executor must only accept calls admitted against the originating turn's granted toolset.

**State persistence needs a stronger upgrade contract.** Known versions migrate, but an unrecognized state shape currently falls back to empty state. For a durable personal session, an unsupported version should fail visibly while preserving the saved state.

## 4. The core session contract

### 4.1 Sessions, runs, turns, and tool calls

A **session** persists across user requests. A **run** is an attempt to satisfy one admitted request. A **turn** is one model invocation. A **tool call** is one operation requested by that turn. An **effect** identifies an external operation whose receipt may arrive later.

Give these stable IDs and explicit lifecycle states. Distinguish queued, active, waiting, cancelling, completed, failed, and cancelled work. A final answer, a context-limit failure, and an unavailable executor must not all appear as “idle.” Preserve partial output with an accurate terminal reason. Lightspeed's [run][ls-run] and [turn][ls-turn] components provide useful examples.

Initially allow one active run per session, with multiple queued runs and multiple sessions progressing independently.

### 4.2 Immutable history and a small reducer

The loop is: admit input, record facts, apply them to state, choose the next step, and emit effect intents. Record cancellation and recovery as new facts. Replaying history reconstructs state without reissuing effects.

Use Arvo's existing durability for the returned state and cards; do not build another workflow engine inside Urbit. Maintain derived session state incrementally instead of folding the complete history on every normal transition. Keep a replay path to verify that the incremental state is correct. Version state and protocol types deliberately.

### 4.3 Durable effects and recovery

Every external operation needs an identity, immutable inputs or input references, status, and a recorded terminal receipt. Receipts must be correlated with the correct session, run, turn, and operation. Duplicate and stale delivery should not alter settled work.

After reconnect, ask the executor what happened to outstanding operations. A command may have completed even when its reply was lost. Prefer rediscovering its result to starting it again. Retry automatically only where the operation's contract makes that safe. Represent uncertain outcomes explicitly when reconciliation is impossible.

An event log does not guarantee exactly-once shell execution. The protocol and executor need result retention and operation deduplication. Executor restart may destroy an ordinary process; the harness must retain the session and report that loss honestly rather than pretend the process survived.

### 4.4 Input admission and active-run control

Support three distinct operations:

- **Steer:** admit input to the current run for the next model-turn boundary.
- **Queue:** accept a request for a subsequent run and expose it as queued immediately.
- **Cancel:** stop new dispatches, request cancellation of outstanding work, and record the terminal outcome.

Track which inputs a planned turn consumes. Unconsumed steering must still receive a turn even if the in-flight response would otherwise finish the run. Client retries need submission IDs so reconnecting does not duplicate requests.

Config, toolset, and environment changes must not silently change the inputs of an already-planned turn. Lightspeed's [active-run control semantics][ls-control] are worth adapting directly.

### 4.5 Context planning and compaction

Keep the full transcript separate from the active context. The context planner selects instructions, tools, user input, native model output, tool results, and any activated instructions. Freeze their revisions for each planned turn.

Implement one provider faithfully first. Preserve its native continuation data, including reasoning-related items or signatures where required. Keep the provider API kind fixed within a session initially. Avoid reducing all provider output to a universal text-message structure.

Compaction needs an explicit request, frozen source context, pending state, and committed result. It should preserve current instructions, task constraints, unresolved work, recent useful detail, and complete tool-call/result groups. Retain the original history. On failure, retain the previous active context and apply an explicit recovery policy.

Reserve capacity for model generation and tool results. Handle an actual context-limit error as well as the estimated high-water mark. Test repeated compactions during one task. Prefer provider-native compaction; where unavailable, make the summarization fallback explicit.

Keep request prefixes stable between deliberate context rewrites. Instruction and catalog updates should be versioned so ordinary turns do not unnecessarily rebuild the prefix. See Lightspeed's [context component][ls-context].

### 4.6 Tool orchestration, waiting, and budgets

Validate each call against its granted definition. Record each terminal result independently, so a failing call does not rerun completed siblings. Return useful errors to the model. Allow safe concurrency and serialize operations whose effects depend on ordering; a sequential first implementation is acceptable if it is explicit and correct.

Long work must return a handle or leave a pending effect. Waiting should not consume model turns merely to keep the harness alive. Support cancellation and deadlines across the full path. Separate “return control after a short wait” from “kill this process at a deadline.”

Make run limits configurable: elapsed time, token/cost budget where measurable, maximum turns where desired, and provider retry limits. Store accounting outside compactable context. Treat ordinary command/test failures as observations the model can act on.

### 4.7 Inspect, resume, and fork

Expose session identity, authoritative run state, ordered events, usage, tool results, and terminal reasons. Support bounded history reads and event cursors so a client can reconnect without gaps or duplicate display. Full large outputs should remain retrievable through references.

Start with forks at settled boundaries. Preserve history and record lineage. A fork must not inherit live external work as though it owns that work. A copied head does not snapshot Linux files or processes; specify whether the branch attaches to a fresh environment, an explicit snapshot, or a deliberately shared environment.

Token streaming improves the interactive experience after these semantics work. Partial frames should travel from executor to client; completed results become durable events.

## 5. Minimum coding tools

| Tool capability | Required behavior |
|---|---|
| Execute | Real shell commands, working directory, environment variables, stdout/stderr, exit status, optional PTY, wait/yield limit, and separate kill deadline. |
| Continue process | Read subsequent output, send stdin, close stdin, interrupt, terminate, and discover terminal status. |
| Read files | Bounded reads with line ranges and useful missing-file/encoding errors. |
| Write and edit files | Create files, make precise replacements, or apply patches; report failed matches clearly. |
| Find files and content | Directory listing, globbing, and text search. Shell utilities can provide these initially. |
| Retrieve output | Bounded model-visible previews plus access to retained full logs and other artifacts. |

These operations must share the same attached Linux environment. Shell edits must be visible to file reads and vice versa. Basic Git use, package installation, compilation, and test execution can use the shell rather than requiring separate tool families.

Lightspeed's [process start][ls-process] and [process continuation][ls-continue] contracts are good starting points. Familiar tool schemas and clear output formatting matter alongside the underlying ability to execute commands.

An on-ship JS runtime does not supply this environment. Further investment in WASM execution is unnecessary for this milestone.

## 6. Defer VFS, retain payload references

A workspace virtual filesystem and a store of immutable model/tool payloads solve different problems. Defer the VFS, but define the payload-reference boundary now.

The executor should assemble provider requests from retained payloads, rather than requiring the head to serialize and resend the entire transcript each turn. Results should return references plus the few fields used for decisions: status, stop reason, tool identities, usage, and relevant handles.

A simple backing store is sufficient for the prototype. Native Urbit blob integration can follow without changing the effect protocol. This is a pragmatic sequencing choice, not proof of multi-year on-ship storage: measure ship event size, loom growth, and replay/inspection cost with representative workloads.

If retained payloads initially live outside the pier, backup and restore must include that store. An executor can be replaceable only if replacing it does not discard the session's history or required receipts. Keep provider credentials outside Arvo from the first external-executor version.

## 7. Implementation sequence and acceptance gates

### Gate A: a functioning coding loop

Implement the session/run/turn skeleton and a versioned executor protocol. Attach one existing Linux environment, implement one provider adapter and the core file/process tools, and use a thin terminal or ACP client.

The smoke set should cover repository exploration, file editing, a failed test followed by a correction, a long command, interactive input, and output too large for one context entry. Start wiring Harbor here so the real environment path is exercised early.

Gate A is reached when these tasks run end to end with inspectable trajectories and accurate completion/failure reporting. It is a development gate, not the completed durability milestone.

### Gate B: a durable session contract

Use deterministic fake-provider/executor tests for state-machine behavior and integration tests for the actual recovery boundary. Required cases include:

| Scenario | Acceptance condition |
|---|---|
| Replay a recorded session | Same state and next intended action; replay itself performs no external work. |
| Disconnect the client during a run | Work continues; reconnect reconstructs current state and missing events. |
| Restart with a model/tool effect pending | Resume or reconcile the operation, or expose a specific recoverable failure. |
| Lose a reply after a command executed | Retained receipt is recovered, or outcome is marked uncertain; no blind duplicate execution. |
| Deliver duplicate or stale receipts | Settled state does not change and later runs remain unaffected. |
| Send input during a final answer | The input is consumed by a subsequent turn or explicitly queued. |
| Cancel during generation or tools | No new work starts; cancellation reaches the executor; late results follow a defined policy. |
| Complete only part of a tool batch | Completed siblings remain completed through retry/restart. |
| Compact repeatedly during a task | Instructions, constraints, unresolved work, and valid tool history remain usable. |
| Fail compaction or admit input during it | No input is silently lost and no stale result overwrites newer context. |
| Reach budget or context limits | Accurate terminal/recovery reason with retained partial output. |
| Fork and continue both sessions | Independent head state and an explicit environment policy. |
| Load an unsupported state version | Saved state is preserved and failure is visible. |

No dedicated automated harness test files were found in the inspected checkout. The reported manual demonstrations are useful starting cases; the above tests establish the stronger contract.

### Gate C: a reproducible Terminal-Bench comparison

Integrate through Harbor's [custom-agent interface](https://www.harborframework.com/docs/agents), using the same executor and tools as normal sessions. Harbor owns task environments and verification; the agent solves the task through its ordinary tools.

Pin the Terminal-Bench 2.1 dataset revision, task images, model snapshot, reasoning settings, resource limits, and timeouts. Version pinning matters: the [2.1 release](https://www.tbench.ai/news/terminal-bench-2-1) revised 28 of the 89 tasks from 2.0.

Begin with the smoke set, then run a declared full-suite campaign with repeated attempts. Retain per-task verifier rewards, trajectories, terminal reasons, infrastructure failures, elapsed time, and token usage. Do not silently exclude agent or executor failures from the denominator.

Compare with Lightspeed under matched conditions. A Codex comparison can use the same model/settings and environment envelope while each system retains its own prompt and tool implementation. This measures complete systems; it does not isolate the value of one harness feature. Lightspeed's [evaluation design][ls-evaluation] provides a useful methodology.

Declare the acceptable performance margin and evaluation procedure before interpreting scores. Report paired task differences and uncertainty. “Pass Terminal-Bench” should mean a measured task-success target, not merely that the adapter runs, and not an assumed requirement to solve every task.

The first milestone is complete only when both the session-contract gate and the declared capability target are met.

## 8. What follows, and what waits

Basic supervised subagents belong in a complete Lightspeed-like session system. The prototype already has child sessions; strengthen those after the single-agent contract works. Children need isolated context, explicit lineage, result delivery, joins, deadlines, budgets, and cancellation propagation. Reuse the same session and pending-work primitives rather than building another loop.

Defer further expansion of:

- VFS and automatic workspace synchronization.
- Gall/Clay application tools and Messenger integration.
- Agent-to-agent protocols and peer-compute markets.
- Skill authoring, self-authored tools, and governed harness modification.
- MCP discovery and broad provider coverage.
- Bot/channel frameworks and specialized scheduling products.
- Environment provisioning fleets, migration, and multi-environment orchestration.
- A dedicated Vere driver, unless measurements show the prototype transport is inadequate.

Existing implementations can remain available without becoming acceptance dependencies. Basic deadlines and durable waiting remain core requirements even though a general scheduling product is deferred.

The immediate work is therefore concrete: move ordinary computation into the executor, make the session lifecycle and context semantics explicit, and prove that the same session can both solve coding tasks and survive interruptions.

## Source references

Source links throughout this document refer to the snapshots assessed here. Useful entry points are:

- [Harness implementation][harness], [feature inventory][h-readme], and [roadmap][h-roadmap].
- [Lightspeed implementation][lightspeed], [session lifecycle][ls-run], and [context management][ls-context].
- [Lightspeed's end-to-end evaluation design][ls-evaluation].

The local checkouts used were `../urbit-agent-harness` and `../lightspeed` relative to this repository.

[harness]: https://github.com/mopfel-winrux/urbit-agent-harness/tree/49d19cb7ba1a29b3462b38092ee03f86c316edcc
[lightspeed]: https://github.com/smartcomputer-ai/lightspeed/tree/8d23c80d165fd2912a8be5bcc786074c15c2d706
[h-readme]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/README.md
[h-roadmap]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/docs-refs/roadmap.md
[h-types]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/sur/harness.hoon#L57
[h-core]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/lib/harness.hoon#L11
[h-load]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/app/harness.hoon#L78
[h-llm]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/app/harness.hoon#L841
[h-request]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/lib/harness.hoon#L247
[h-dispatch]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/app/harness.hoon#L699
[h-actions]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/app/harness.hoon#L313
[h-guards]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/lib/harness.hoon#L85
[h-acp]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/acp/README.md
[h-subagents]: https://github.com/mopfel-winrux/urbit-agent-harness/blob/49d19cb7ba1a29b3462b38092ee03f86c316edcc/desk/app/harness.hoon#L1089
[ls-run]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/crates/engine/src/core/components/run.rs
[ls-turn]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/crates/engine/src/core/components/turn.rs
[ls-context]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/crates/engine/src/core/components/context.rs
[ls-steering]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/crates/engine/src/core/drive.rs#L4693
[ls-control]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/docs/roadmap/p129-active-run-control.md#L134
[ls-process]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/crates/tools/src/environment/tools/run_process.rs#L17
[ls-continue]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/crates/tools/src/environment/tools/continue_process.rs#L18
[ls-evaluation]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/docs/roadmap/p149-harbor-end-to-end-agent-evaluation.md
