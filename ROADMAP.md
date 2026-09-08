# An agent harness on Urbit: roadmap

I propose that we first prove that a harness running on Urbit can do serious coding work. Then build out the everyday agent experience. Finally, use Urbit's native programmability and software distribution to let the harness develop itself and share improvements with other ships.

This roadmap turns the [proposal](README.md) into three delivery milestones, building on the [current harness experiment][harness] and lessons from [Lightspeed][lightspeed] (an enterprise version I've built, which has validated many of the ideas here). The [design notes](harness-design-notes.md) provide architectural background; this document sets the implementation order.

## Milestone 1 — A capable coding agent

The first proof is an agent that takes a task, works autonomously in a real repository, and finishes it—even when the work spans several context windows.

Unix tools come first because coding models are already trained to use familiar terminal and editing interfaces. Native Urbit tools need more design and experimentation. [Tool-design guidance][coding-tools].

- **Implement OpenAI Responses first.** Support native coding conversations faithfully: response items, reasoning and continuation state, tool calls and results, usage, and errors. Make this API complete for the coding loop before broadening provider coverage.
- **Provide the core coding tools.** Take Lightspeed's inventory: `list_dir`, `read_file`, `write_file`, `edit_file`, `apply_patch`, `grep`, and `glob`, plus `exec_command` and `write_stdin` for shell commands and scripts. All operate on the same real filesystem. Long commands return process handles, accept input, and yield further output; large results remain retrievable. [Tool reference][ls-tools].
- **Configure capabilities per session.** The user or creating process explicitly enables and configures each tool family. Omitted capabilities are unavailable. Enforce grants at execution; extend the same rule to MCP, Urbit development, and later tool bundles.
- **Bridge to a real Linux environment.** Attach a VM or container through a replaceable executor. Prefer a registered remote environment so the ship and task machine can run separately. The initial transport remains a decision: an HTTP adapter, direct reuse of Lightspeed's JSON-RPC-over-WebSocket protocol, or a local sidecar over Lick. A local first version controls only the ship's host environment. [Environment reference][ls-environments].
- **Keep the session primitive small.** One ship supports multiple independent sessions, progressing concurrently or sequentially, with one active task per session. Each accepts a task, maintains its context, and reports completion or failure. Task completion leaves a session open and idle; explicitly closing it prevents further inputs, after which it can be deleted.
- **Make long contexts work.** Build and maintain the active context, preserve the full history, and compact automatically using native Responses compaction. Retain the resulting continuation items and handle context overflow or compaction failure without losing the task. Exercise repeated compaction during real coding work. [Compaction reference][openai-compaction].

**Complete when:** the Urbit harness runs inside a VM and completes a Harbor evaluation on [Terminal-Bench 2.1][tb21]. An adapter submits tasks to sessions; the agent uses its normal coding tools, and Harbor verifies the task environments. Set a task-success target before the scored campaign; meeting it is the capability gate. Report the score, failures, model/settings, and a matched Codex baseline (~80% score).

**Deferred:** conversational follow-ups, steering, input queues, user cancellation controls, forks, subagents, general jobs/promises, token streaming, MCP, skills, and a virtual filesystem. Basic process continuation is already part of coding.

## Milestone 2 — A full agent experience

Build toward the feature set of a full agent harness, while keeping deep Urbit application integration for milestone three.

- **Complete session control.** Add conversational continuations, steering during active work, queued follow-ups, cancellation, retries, and session forks. Make status, history, tool results, and usage inspectable. Reconnecting clients and ship restarts should preserve sessions and reconcile outstanding work.
- **Broaden native API support.** Add **OpenAI Chat Completions first, then Anthropic Messages**, including compatible endpoints. Preserve each API's reasoning, context, compaction, and caching behavior. Adapt tool names, schemas, and instructions to the model: OpenAI-style patch and execution tools, for example, and Claude-style editing and Bash tools.
- **Add supervised subagents.** Delegate tasks to child sessions with their own context, run them in parallel, and collect their results. Carry cancellation and limits through the delegation tree. Support reusable agent configurations for recurring kinds of work.
- **Make waiting and remote work efficient.** Introduce promises and `await` for any or all results, with timeouts and cancellation. Waiting should suspend work until a result arrives. Extend the environment bridge with background jobs and later result collection, so coordination does not spend model turns polling across the network (see Lightspeed implementation of those primitives).
- **Implement an Urbit-native MCP client.** Discover and call external tools over current Streamable HTTP, supporting unauthenticated servers and configured API keys or static tokens. Defer OAuth flows and an MCP server. The client must handle the HTTP streaming version; the older HTTP+SSE transport can be omitted. [Transport specification][mcp-transport].
- **Support richer inputs and web access.** Accept images and documents, pass their native content to capable models, and retain retrievable artifacts. Add web search and fetch through provider tools or harness tools.
- **Resolve skill discovery and loading.** Decide whether skills initially live on the ship or in the attached Unix environment. Basic support can land here; native Urbit skills are a firm milestone-three commitment. Introduce a virtual filesystem only if a concrete need justifies it. Token streaming is useful polish and can move to milestone three.

**Complete when:** a user can carry out extended work across all three APIs, continue and control sessions, delegate parallel tasks, await background jobs, and use MCP tools and media inputs through one coherent harness. Keep the milestone-one benchmark as a regression check as these capabilities expand.

## Milestone 3 — A harness that develops itself

This is where Urbit becomes the differentiator. Start with a harness that can extend itself on its own ship, then let agents on different ships collaborate and share improvements.

- **Build an Urbit development toolset.** When enabled for a session, let it inspect and edit Clay, compile and test Hoon, and manage Gall agents on its own ship or a permitted target such as a moon. These tools serve the same development needs as Unix tools, but need their own schemas and feedback. Register newly authored tools for use by sessions granted access.
- **Maintain a native toolbox.** Support Urbit-native skills alongside tools, prompts, and session configurations. Give each session a discoverable inventory of its enabled capabilities and the instructions needed to use them. Store and version these capabilities in Clay.
- **Develop the harness with the harness.** Move from external coding agents implementing the system to the harness developing its own next version. Stage changes, test them against representative tasks, rehearse upgrades against copied session state, and demonstrate an upgrade that preserves existing sessions.
- **Share useful improvements.** Package tools, skills, prompts, and configurations as Clay desks that another ship can install and its harness can discover. [Software distribution][urbit-dist].
- **Connect agents across ships.** Develop our own inter-Urbit agent protocol for exchanging messages, requesting work, and returning progress and results. Design it around Urbit identities, sessions, and native capabilities, with room to evolve beyond the constraints of A2A or ACP. A2A can be layered on top as an optional compatibility adapter so external agents can communicate with Urbit agents; it does not define the protocol between ships.
- **Approve incoming bundles.** Show the receiving owner what another ship's bundle adds or changes, and require explicit approval before installation and activation. Keep this initial flow simple. Broader security, isolation, and validation work follows the working end-to-end demonstration.
- **Run from native events.** Add schedules through Behn, webhooks through Eyre, and event triggers through Gall subscriptions. Store rules and filters on the ship, routing matching events into new or existing sessions. This make Urbit agents claw-like (e.g. OpenClaw).

**Complete when:** from a plain-language request, the harness builds, tests, and uses a native tool; develops and tests an improvement to its own implementation; and shares a capability bundle. After its owner approves adoption, a second ship discovers and uses the bundle. The two harnesses exchange a task request and result through the native inter-Urbit protocol. This demonstration makes the vision tangible: agents that improve their own capabilities, share them, and work together across ships.

[harness]: https://github.com/mopfel-winrux/urbit-agent-harness
[lightspeed]: https://github.com/smartcomputer-ai/lightspeed
[ls-tools]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/crates/tools/src/builtin/mod.rs#L318
[coding-tools]: https://developers.openai.com/cookbook/examples/gpt-5/codex_prompting_guide#tools
[ls-environments]: https://github.com/smartcomputer-ai/lightspeed/blob/8d23c80d165fd2912a8be5bcc786074c15c2d706/docs/spec/04-environments.md#ownership-and-extension-boundary
[openai-compaction]: https://developers.openai.com/api/docs/guides/compaction
[tb21]: https://www.tbench.ai/news/terminal-bench-2-1
[mcp-transport]: https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/streamable-http
[urbit-dist]: https://docs.urbit.org/build-on-urbit/userspace/dist
[urbit-arvo]: https://docs.urbit.org/build-on-urbit/app-school/1-arvo
