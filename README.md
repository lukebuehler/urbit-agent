# What is Urbit for?

*A proposal to make Urbit the home of your personal AI agents, because it is already built for exactly that.*

---

We should ask ourselves: what is Urbit for? Over the years there have been a myriad of answers. *I propose today we can answer it simply and clearly: Urbit is for your personal agent harness.* Or more pointedly, the only way for Urbit to matter in the future is for it to host your core agent loop.

Urbit fits that mission perfectly. For once, the idiosyncrasies of the architecture and runtime are uncannily aligned with the design a frontier agent harness calls for: durable state that is entirely event sourced, and all heavy computational actions, such as LLM turns and tool calls, being async and offloaded to other systems. This is what Urbit's solid state interpreter vision is best at. The rest of this proposal tries to show why that is the case, and what it would take.

## Starting at the wrong end

Previous attempts at making Urbit relevant have largely failed to materialize. I would argue this was, at least partially, perhaps largely, because they were not able to leverage Urbit's strengths. The goal was usually to make Urbit some sort of socially networked communication tool. By doing so we started at the wrong end: we implemented messaging and social features that can be achieved with a much simpler design and protocol. And they _have_ been achieved with simpler designs, which is why Signal, WhatsApp and Telegram roundly won that race. It is hard to justify running something as complex as an Urbit ship just to send text messages, while almost everything that makes Urbit special goes unused or gets in the way.

None of this is an argument against Tlon Messenger. Messaging is how people arrive on the network, and it is where an agent will eventually show up. It is an argument about what should sit at the center. Messaging did not really need Urbit, but an agent does.

## The head vs. the hands

A harness consists of various pieces, but at the core is the agent loop. This loop is entirely a state machine: it manages what goes into the context window of the model, turn after turn. It admits inputs, decides what the next turn looks like, sends it off, records what came back, and decides again. At the heart of it there is no I/O. Let's call this part the head.

The other part is giving the agent access to resources: VMs, sandboxes, shells, file systems, MCP servers, APIs. Everything that actually touches the world. Let's call these the hands.

Everyone building these loops is squashing the two parts together. Claude Code, Codex, OpenClaw et al. are designed to run inside an operating system they own, and they need that whole OS for themselves. That is precisely what makes them hard to keep alive, hard to secure, and hard to make truly yours: when the machine goes, the agent goes with it. The industry has started to notice and is pulling the two apart. The head is small, deterministic, and should live for years. The hands are heavy, short-lived, and best borrowed for exactly as long as a task needs them.

Urbit would be fantastic for the head, and not much good for the hands. Urbit should be the head, and only the head. It should never try to be the sandbox, the VM or the shell: those are commodities, and frontier models are trained to expect a real POSIX-compatible machine anyway. What is not a commodity is a head that is yours, that remembers, and that does not die.

## Why Urbit is the head's shape

The core agent loop is a complex endeavor and, done right, it adds a lot of value (see Pi being used as the core of OpenClaw). Look at what such a loop needs, and Urbit's idiosyncrasies start to read like a specification.

A session is a log of what happened, and the state of the agent is a function of that log. On Urbit that is not a design decision anyone has to make, it is how every Gall agent already works. The loop itself has the exact rhythm of Arvo talking to its runtime. The head asks for an LLM turn, the answer arrives as an event; the head asks for a tool call, the result arrives as an event. The head never blocks on the outside world, it only ever decides what to ask for next. Everything that makes Urbit awkward for ordinary applications is exactly what a head needs: one event log, computation strictly separated from I/O, and state that is a value rather than a database.

But even better: Urbit allows us to be more ambitious than anyone else building harnesses. What we have seen so far from self-programming, self-configuring harnesses leaves me unimpressed. Pi is kinda the state of the art here, and it is a TypeScript app with hot reloading. Urbit is designed for the next generation of self-modifying harnesses: a dynamic environment that configures itself, where the programming is the deployment at the same time. Picture an agent that writes a new tool or skill into a desk, tries it on a copy of its own session, and commits it if it works. If it crashes, nothing happened: a failed event never touches state. That is a harness that grows with its owner. So, we can do better, we can be more ambitious than what is currently out there.

## What it takes

The first step is a capable coding agent: a Gall-hosted loop with independent sessions and a bridge to a real Linux environment. Start with OpenAI Responses and established Unix tool-sets to read, edit, and run code, then prove that capability on Terminal-Bench 2.1. Unix tools come first because models are already trained to use them and we can follow standard implementation paractices; native Urbit tools will need more design and experimentation. The Urbit/Unix split also really tests the head-and-hands thesis.

Then build out the full agent experience: conversational continuations, steering and cancellation, more model APIs, subagents, efficient background work, and an Urbit-native MCP client. These make the harness useful for sustained everyday work and connect it to existing tools and services.

The next step is a native Urbit development environment. Give a session tools to inspect, edit, test, and upgrade its own ship, or another ship it has permission to develop. The harness can then build new tools and skills, develop its own next version, and share improvements as desks that other owners approve and adopt. Agents should also work together through our own inter-Urbit protocol, with A2A as an optional bridge to outside agents. Finally, Native Behn schedules and timers, webhooks, and other triggers bring the agent to life because it can react to real-world events.

Making context handling truly native to Urbit requires keeping all relevant context data inside Urbit: conversation history, model continuation state, tool results, and retained artifacts. This is why the 64-bit runtime and native blob storage are essential to the full vision. The 64-bit runtime gives growing session state room beyond the old loom limits; blob storage keeps large payloads on disk within the pier, represented by small references in active state and the event log. Together, they let the agent retain years of memory as part of its ship, moving and recovering with the pier while external executors come and go.

The [harness roadmap](ROADMAP.md) lays out these three milestones and the criteria for proving each one.

## An agent that is yours

Put together, this is a fairly simple thing to want: an agent that is yours. It lives on your ship, so its memory is your memory, and it remembers for years rather than for a session. It moves when your pier moves. You can fork it to try something and throw the fork away. It shows up in Messenger, in your notebooks and in the terminal, and it talks to your friends' agents via Ames. And because the head is light, hosting it stays about as cheap as hosting a ship is today: the expensive parts are borrowed only when there is work to do.

This is not a detour from where the ecosystem is already heading. Tlonbot already gives every account an agent with its own identity, and Tlon has said it wants that agent's memory under the user's control. Groundwire has shown an agent loop running on a ship, with branching conversations and tools exposed from Urbit itself. What is proposed here is the next step of both: the harness itself moves onto the ship, and whatever runs next to the ship becomes a replaceable pair of hands.
