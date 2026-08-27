We should to ask ourselves: what is Urbit for? Over the years there have been a myriad of answers. I propose today we can answer it simply and clearly: Urbit is for your personal agent harness. Or more pointedly, the only way for Urbit to matter in the future is for it to host your core agent loop.

Urbit fits that mission perfectly. For once, the idiosyncrasies of the architecture and runtime are uncannily aligned with the design a frontier agent harness calls for: durable state, which is entirely event sourced, all heavy computational actions, such as LLM turns and tools calls are async and can be offloaded to other systems. This is what Urbit's solid state interpreter vision is best at.

Previous attempts at making Urbit relevant have largely failed to materialize. I would argue this was, at least partially, perhaps largely, because these attempts were not able to leverage Urbit's strengths. The goal was usually to make Urbit some sort of socially networked communication tool. But by doing so we started at the wrong end: we implemented trivial messaging and social networking features that could have been achieved with a much a simpler design and protocol. And they _have_ been achieved with simpler design, hence Signal, WhatsApp, Telegram, et al. roundly won over Urbit. It simply makes no sense to run and maintain something as complex as countless Urbit instance simply to send text messages to each other, while almost all the complexity and features of Urbit itself are not utilized or are in the way.

Let's focus on a use-case that leverages Urbit to the very core!

todo: why Urbit and agents are a good match
- harnesses consist of various pieces, but at the core is an agent loop. this loop is entirely an state machine that manages what the goes into the context window of an agent. at the heart, there is no I/O
- the other part is giving the agent access to ressources such as VMs, sandboxes, MCPs, apis, etc
- everyone building these loops is squashing together these two parts. Urbit would be fantastic to manage the first part, not so much for the second. 
- But the the core agent loop itself is quite a complex endeavor and if done right adds a lot of value (see Pi being used in OpenClaw)
- and what we've seen so far from self-programming, self-configuring harnesses leaves me unimpressed, Pi is kinda the state of the art here. but it's just a typscript app with hot reloading. We can do better, we can be more ambitious.
- Urbit is design exactly for this, a dynamic environment that can be self-configured: The programming is the deployment at the same time (homoiconicity, solid state interpreter, etc).


---
Let's assume a token is about 4 bytes and an agent fills and utilizes a context window of 1M tokens about once an hour. We extrapolate this from an average agentic coding session, which  fills such context often much faster (as quickly as 5-10 minutes sometimes), and assume more moderate, yet still quite intense knowledge work tasks.

24M tokens per day * 4 bytes ~= 100 megabytes per day. 

Most of that data does not have to enter and exit Urbit, however. The harness simply needs the information required to do the branching.

