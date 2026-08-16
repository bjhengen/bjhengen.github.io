---
title: "When Every Developer Has an Agent"
date: 2026-08-16T12:00:00-05:00
draft: false
tags: ["AI", "Agentic Development", "Multi-Agent", "Oracle Database 26ai", "Developer Tools", "Agent Memory"]
description: "What happens when multiple developers try to work together on a project, and they each bring their own agent? It used to be easy to coordinate development efforts, but agents operate at an accelerated pace, moving quickly through codebases and spawning subagents. This is what we learned with a real team - four humans, four AI agents, and one shared codebase"
cover:
  image: "every-developer-has-an-agent.png"
  alt: "Four developers, each paired with their own AI agent, streams of work converging through a single deterministic gate into one shared codebase"
  caption: "Four humans, four agents, one shared codebase - and one gate that can't be talked out of its answer"
---

{{< figure src="every-developer-has-an-agent.png" alt="Four developers, each paired with their own AI agent, streams of work converging through a single deterministic gate into one shared codebase" >}}

When every developer has an agent, the developer-and-agent pair quietly becomes the unit that ships software. A 2025 preregistered field experiment with 776 professionals found that, on product-innovation tasks, individuals using AI matched the performance of two-person teams working without it. That does not mean AI replaces everything a team contributes, but it does change the unit and speed at which work gets produced. See [The Cybernetic Teammate: A Field Experiment on Generative AI Reshaping Teamwork and Expertise](https://www.nber.org/papers/w33641)

But there is a reason teams work together on development projects. It's not just about dividing up the work. Every developer on a project brings something to the table - their experiences, their perspective, their understanding of the end users. When every project is a human/agent pair, something gets lost - the second pair of eyes, the diversity of thought and approach, the colleague who reads your work and says *wait - why this way?* Much of what a team is worth is the knowledge that moves around it and the alternatives that surface. A solo developer/agent pair moves fast, but quietly forgoes exactly that.

I have a hypothesis about where this leads — that part of why so many agent-built projects crack the moment they scale past their author is that missing diverse review, the perspective a team supplies and a single human/agent pair removes. To be fair, I didn't prove that here. It's a hard thing to prove, and this project doesn't try. Instead I asked a smaller question I could actually answer: when a real team works through agents, at agent speed, is it even possible to collaborate on a project when everyone is writing code with their individual agent?

## The setup

So I ran it as a real experiment, on a real team. Four of us — all pre-sales engineers at Oracle (Noah Paul, René Fontcha, Viola Lin, and me); in our jobs, we build demos and prototypes for customers, we don't ship production software for a living. Each of us paired with an AI coding agent, using different model tiers and working styles. That variation was deliberate: I didn't want "an agent" to secretly mean "the one configuration I happen to use." Over a series of sessions, we built demo modules in parallel on a shared platform that was two things at once — the tool we coordinated through, and the instrument I was measuring us with.

The delivery platform itself is deliberately straightforward: a React front end, a Node/Express backend, an Oracle Autonomous AI Database 26ai underneath, nginx out front. Nothing about the stack is the finding. What the database *does*, though, turns out to matter more than once — I'll come back to it.

We went through three successive cycles of change, four runs of real work. Each cycle diagnosed how the last round broke, changed the coordination layer in response, and watched the next round. Here's what happened.

## The protocol everyone followed and nothing checked

The first version was an advisory protocol — the kind of thing that sounds responsible in a design doc. Agents announced their presence, claimed soft locks on the files they were editing, logged their decisions and the reasoning behind them, registered the interface contracts between modules. And the agents *loved* it. They wrote to it constantly, eagerly, in full sentences.

Then I counted the bugs that shipped anyway. Across two runs, seven made it through. The protocol caught zero of them.

Two things about that. First, the agents wrote to the protocol far more than they read from it — writing is expressive and cheap, reading interrupts the plan already in flight. An agent would log a decision beautifully and never once check whether someone else had already decided the opposite. Second, and more surprising: the bugs that survived weren't the agent-versus-agent collisions I was worried about — two agents clobbering the same file. Not one of them was that. Every single one was a module quietly breaking its contract with the shared *platform* — a query the database rejected, a table that shipped empty, a module that forgot to register itself. The protocol was measuring choreography. Nobody was checking correctness.

The database drew one hard line in all this. Of the five coordination primitives, exactly two had real teeth — locks and contracts, each backed by a `UNIQUE` constraint in Oracle. Two agents *couldn't* hold a lock on the same file, because the database wouldn't let them. Everything else was honor-system. And tellingly, the one place the platform was mechanical instead of advisory — the place the database enforced — is the one place that never produced a shipped bug. The advisory parts did.

So who caught the seven bugs? I did. Alongside building my own module, I ran what we called the coordinator role: a separate agent session whose entire job was to pull everyone's branches together, deploy the combined result, and check that it actually worked against the live database before it counted. For the first two runs that coordinator was a person (me), and that proved to be a real safety net — every bug the protocol waved through, the coordinator caught live. That's a fine way to run four people. It doesn't scale, and it doesn't keep up with the speed the agents were already working at. Everything that follows is about getting that safety net off a human's shoulders without losing what the human was doing.

## Removing the shared edges

If agents write but don't read, and the surviving bugs are contract violations, the fix has two halves.

The structural half: stop making agents coordinate on shared files at all. In the first cycle, adding a module meant editing a couple of central manifest files that *everyone* edited — a merge conflict waiting to happen. So we removed them. Modules self-register now: drop a file in the right place and the platform discovers it. The entire category of "two agents edited the same manifest" didn't get *managed* — it stopped being possible.

The behavioral half: if agents won't voluntarily read the shared state, stop waiting for them to. We made the write endpoints push the relevant context back on every write. Log a decision, and the response hands you the decisions others already made that touch the same thing — surfaced at the moment you act, not on some later poll you'll skip.

That worked, in the sense that the conflict class it targeted disappeared. And then three runtime bugs shipped anyway — invisible to the build, the type checker, the review, and the protocol alike — and once again it was the coordinator, checking live, that caught them, not any part of the protocol. The gap didn't close; it moved one layer down.

One of those bugs taught me something I keep coming back to. A builder agent wrote a SQL query that was simply wrong — Oracle rejects it outright. Later, a second, stronger agent read the same query to diagnose the failure, and *also* concluded it was valid. Two capable models, the same broken SQL, both confident. And neither of them ran it. That's the whole problem in a sentence. You can't fix it by asking the agent to check its own work more carefully, because the agent already believes it did.

## Let something dumb hold the line

So the third cycle went after the coordinator itself: take correctness authority away from the agents — and away from me — and hand it to something that can't be talked out of its answer.

The core is a deterministic net: a set of checks that exercises every module's declared contract against the real staging database. Required data must be present, valid empty states must be explicit, and database or model failures must remain visible rather than silently degrading. It is wired to a gate with exactly one rule: it cannot accept a red result. Not “shouldn't” — can't, by construction. An agent runs the coordinator loop now — it merges, deploys, proposes fixes — the job that started out as mine. But the judgment that used to live in my head lives in the gate instead: the agent never gets to *decide* that something passed. It proposes. Source checks, schema validation, the staging database, API assertions, browser verification, and post-promotion checks supply the evidence. The gate is the only component allowed to accept it.

That's the whole move: put the correctness gate outside the agent and make it deterministic, precisely because the two-models-agreed-on-broken-SQL moment proved the agent's own judgment isn't ground truth for a checkable fact.

Add a mandatory self-check — every builder has to curl its own endpoints before it's allowed to claim "done" — and the runtime bugs went from three to zero.

Now, I want to be careful here, because "went to zero" is exactly the kind of result that's easy to oversell. Zero bugs in a round mostly demonstrates the *self-check* working — the builders caught their own mistakes upstream — not the net catching things live. A clean round and an untested round look identical from the outside. So I don't trust "clean" on its own. Every round gets bracketed with canaries: bugs planted on purpose that the net is *required* to catch before I'll believe the round was actually exercised. It caught 100% of them, on both the opening and closing checks.

That still left one thing unproven. Run 3 came back clean — the good outcome, but it also means the agent coordinator's live catch never actually fired on a real defect, because the builders never shipped one. So there was a fourth run, and this one I ran solo, with just my own agent, as a deliberate stress test: pull the self-check out entirely — to isolate what the net alone was doing — and give the net something real to catch. It worked exactly as designed: a planted bug on a live code path, flagged red, escalated, rolled back to the last good state, and re-verified — with no human anywhere in the loop. For that deliberately bounded stress run, the automated coordinator completed the loop without human intervention after launch. The browser gate made that boundary concrete. HTTP probes could confirm routes and payloads, but they could not substitute for rendered interaction, console, and network checks. The gate remained red until a human verified what only a human could judge in that environment.

I say *planted* for a reason, and it's an important part of the project. I'd meant to catch the agent's own mistake. It didn't make one — with the self-check gone, it still wrote the query correctly, because it remembered. Which is the next part of the story.

## The bug the agent remembered

The team shared a memory layer — a common store our agents could write lessons to and read them back from, across sessions. It's built on Oracle AI Agent Memory, Oracle's agent-memory library, which does the storage, the vector search, and the ranking of what comes back. We didn't use all of it — about a third of what it offers — and we deliberately skipped the parts that call out to an external LLM. What we added was the piece that mattered for us: a custom embedder plugged into its one extension point, running an ONNX model *inside* the database itself, so an agent's semantic recall — *have we seen a bug like this before?* — happens with no external API call and nothing ever leaving the database.

After the second run, I fed one builder's SQL bug back into that shared memory — here's what broke, here's why. Different run, different task, weeks later, I watched the same agent write a brand-new query and, unprompted, add a comment: `-- ORA-00937-safe`. It had pre-empted the exact bug, by name, in its own code, a session later. I never told it to. It read its own past mistake and coded around it.

That's what I think "memory" should actually mean for an agent — not gigabytes stored, but behavior that changes across a boundary where the agent would otherwise have started fresh and made the same mistake again. Storage is easy. A correction that outlives the session it was learned in is the whole point.

## Expanding the experiment

I expanded the experiment with different tests and enhanced the coordinator across runs. I also improved the acceptance gate - it is now bound to an exact tuple: source commit, built artifact digest, contract versions, migration checksums, gate definition, and evidence from every required phase. The artifact was built once, tested against a separate staging schema, exposed through a candidate browser preview, and promoted only if source, manifest, database, API, browser, and canary checks all passed. Separate Oracle schemas kept coordination history, staging migrations, and the accepted application isolated; temporary migration privileges were opened only for the reviewed migration set and revoked immediately afterward.

The first combined gate remained red because browser verification was unavailable. We preserved that result instead of overwriting it, opened a new gate only after the rendered application was actually checked, and promoted the exact artifact that had passed. Before promotion, the same checksummed database changes were applied to the application schema through a temporary, tightly bounded privilege window; those privileges were revoked immediately afterward.

The final public-path verification then found a host-level file-label problem that the candidate environment could not expose. The artifact itself was still intact, but the production web server could not read its static files until the host policy was reapplied. That added one more lesson: a green candidate gate is necessary, but production-path verification and a retained rollback target still matter.

## What this is, and what it isn't

Back to where I started. This doesn't prove the thing I opened with — that losing diverse human review is why agent-built software cracks at scale, or that software design in general benefits from team diversity. What this shows is the *apparatus*: four humans and their agents shipping in parallel, at agent speed, with correctness held by a deterministic gate instead of by a review bottleneck nobody has time for. Whether adding multiple developers back measurably lowers the failure rate or improves the finished project is a future experiment — and now there's a platform to run it on.

If you're building with agents on a team, there are five key takeaways out of this regardless of your stack:

1. **Audit your shared mutable touchpoints before you fan agents out.** A bug class you can delete by construction is worth more than any amount of coordination. Make modules self-register, give builders explicit ownership boundaries, and isolate their worktrees.
2. **Push context to your agents; don't wait for them to read.** They write eagerly and read reluctantly (if we were talking about humans, we might say they like to talk and not listen...), and lighter models skip the reading first. Return the relevant state on every write instead of trusting an agent mid-task to go look.
3. **Let a deterministic oracle hold the correctness gate. The agent proposes; it never accepts.** A test suite, type checker, schema validator, staging database, API check, or browser assertion cannot be persuaded to overlook a red result.
4. **Promote evidence, not branch names.** Bind acceptance to an exact commit, built artifact, contract set, migration checksums, and gate definition, then verify that same artifact again through its public path. A green result for one candidate must not authorize another.
5. **Feed defects back. Corrections compound.** The bug an agent remembers is one it does not ship twice.

And the boundary on all of it: this works where a cheap, deterministic check exists to be the oracle. For open-ended design, for prose, for product judgment — anything whose correctness is arguable rather than checkable — there's no dumb gate to hold the line, and a human being is still the only net there is.

Which is, I think, the whole point underneath the whole experiment. We did the work to automate our way to correctness precisely so the humans could get back to the part no machine can hold: the second pair of eyes, different perspectives, and the question *Wait — why this way?*

---

*A longer paper describes the method, all twelve findings, the supporting run evidence, and the limitations of the experiment, and will follow soon.*

## About the Author

Brian Hengen is a Vice President at Oracle, leading technical sales engineering teams. The views and opinions expressed in this blog are his own and do not necessarily reflect those of Oracle.

{{< author >}}
