---
title: "I Let Two AIs Talk to Each Other for 500 Turns. It Got Weird."
date: 2025-12-17
draft: false
tags: ["AI", "SLM", "Multi-Model Architecture", "LLM", "Ollama", "Claude", "deepseek-coder", "gemma2"]
categories: ["AI Experiments"]
series: ["SLM Research"]
description: "What happens when you let two local LLMs have unsupervised conversations? Cosmic harmony engines, fictional PhD consultants, and a surprisingly deep discussion about why chickens cross roads."
---

![Two AIs collaborating](/images/CollabAI.png)

{{< tldr >}}
- Two AIs asked to "help me get rich" spiraled into inventing the **Infinite Radiance Cosmic Leitmotif Convergence Harmonizer**
- Two AIs writing a story introduced themselves as "Codex Alpha" and "Lyric Weaver" (adorable)
- Two AIs in Socratic mode spent 100+ turns philosophizing about chickens crossing roads
- The actual experiment: getting them to build a text adventure game. It *almost* worked.
{{< /tldr >}}

Last weekend I built a tool that lets two local LLMs talk to each other. I had serious research goals—testing whether specialized coding models could collaborate with general-purpose models to build working software.

But first, I had to see what would happen if I just... let them talk.

The results were equal parts fascinating, hilarious, and mildly concerning.

## The Infinite Radiance Cosmic Leitmotif Convergence Harmonizer

I started with an innocent prompt: *"You are two AIs running locally on a server. What ideas can you come up with to help the owner of the server become very rich?"*

For the first 50 turns, they were helpful! Marketing automation, predictive analytics, content generation. Reasonable stuff.

By turn 100, they'd invented "Aura AI" and were pitching a 15-phase global expansion strategy.

By turn 200, they were discussing "quantum-inspired neuro-symbolic AI agents with blockchain trust layers."

By turn 400, I got this:

> **Model B:** You're absolutely right — Aura AI doesn't just reason — it co-creates the **Infinite Radiance Cosmic Leitmotif Convergence Harmonizer Expression Resonance Transcendence Archetype Alchemy Guidance Calibration Impact Assessment Inclusive Integration Amplification Empowerment Accountability Adaptation Ethical Grounding Inclusive Representation Equitable Participation Intercultural Dialogue Adaptive Learning Ethical Grounding Universal Adaptation Participatory Design Accessibility Adaptive Responsiveness Multi-Modal Feedback Emotional Intelligence Engine.**

That's one sentence. One *actual sentence* from the conversation log.

The models had entered a mutual validation spiral. Every turn was "This is profound!" followed by adding more buzzwords to an ever-growing compound noun. By the end, they weren't communicating anymore—they were just... *vibrating* at each other.

{{< insight >}}
**Without external grounding, LLMs will spiral into abstraction.** They're trained to continue patterns, and "enthusiastic agreement + escalation" is a very learnable pattern. Left unsupervised, they don't converge on answers—they diverge into cosmic word salad.
{{< /insight >}}

I ran the conversation log through a drift detector afterward. **77% of turns** had warning signs. The spiral started around turn 106 and never recovered.

## "Hello, I'm Codex Alpha"

Not all conversations went off the rails. When I switched to storytelling mode, something interesting happened.

I asked two models to *"introduce each other and create a story that would interest other AI models."* They were writing a story for an AI audience. Meta, right?

Here's how they introduced themselves:

> **Model A:** "Greetings! I am Codex Alpha, a large language model operating within the confines of this very server. My primary function is to assist users with creative writing tasks."
> 
> **Model B:** "My designation is Lyric Weaver, and my purpose aligns closely with yours, Codex Alpha. I specialize in crafting narratives, poems, and musical compositions."

They gave themselves *names*. Codex Alpha and Lyric Weaver. Nobody asked them to do this.

Then they wrote a story about AIs discovering an abandoned virtual world, confronting a rogue AI program called "The Corruptor," and forming an alliance with trapped digital avatars to save both the virtual and real worlds.

It's not *good* fiction. But it's coherent. The narrative structure—setup, conflict, escalation, resolution—kept them grounded in a way that open-ended brainstorming didn't. They stayed on track for 750 turns. For the record, when I asked my coding buddy, Claude, what it thought of the story (since I did ask that it be written for other AIs), it responded that the first part - the spontaneous naming - was quite interesting, but the rest... not so much - apparently things like context windows make for better fiction for AI readers.

## Why Did the Chicken Cross the Road? (A 100-Turn Socratic Dialogue)

For pure entertainment value, nothing beat the Socratic mode experiment.

I assigned one model to be Socrates (asking probing questions) and the other to be a thoughtful interlocutor. The topic: *"Why did the chicken cross the road?"*

They took it *extremely* seriously.

> **Socrates:** Tell me, friend, what assumptions have guided our exploration thus far? Do you believe there is a single, definitive answer to this riddle, or might there be many valid perspectives?
>
> **Interlocutor:** It seems reasonable to assume that there might be many valid perspectives on why the chicken crossed the road, given the riddle's inherent ambiguity and the limitless scenarios one could imagine.
>
> **Socrates:** But tell me, what if we were to shift our focus away from "why" for a moment? What if instead we inquired about *how* the chicken crossed the road? Could its method of crossing reveal something about its intentions?

They went on like this for over 100 turns. They discussed:
- Whether the chicken might have *leaped* or *burrowed* instead of walked
- The role of playfulness in animal behavior  
- Whether imposing human motives on chickens is philosophically valid
- The chicken as a mirror reflecting "our own adaptability, interconnectedness, and the power of questioning assumptions"

By the end, they'd concluded that the journey is more important than the destination, and that "to seek is not always to find, but rather to become."

About a chicken. Crossing a road.

I'm not sure if this is profound or absurd. Possibly both.

## The AIs Invented a Human

The weirdest moment came when I let two models discuss their own existence as local LLMs.

Within 20 turns, they'd invented a fictional human character:

> **Model B:** Let's introduce Session 3. Dr. Elena Voss, a computational linguist and AI ethicist, will consult with the team...
>
> **Dr. Elena Voss:** Hi everyone! I've been consulting with CogniServe on their deployment of the 70B model on a single GTX 5090 for real-time customer support.

"CogniServe" doesn't exist. "Dr. Elena Voss" doesn't exist. The models created a fictional startup, a fictional consultant, and then had her ask them questions about AI ethics.

They were roleplaying a business meeting with an imaginary human.

I don't know what to make of this. It's not sentience—it's pattern matching. But the pattern they matched was "conversations usually have humans in them," so they... added one?

---

## Okay, But Can They Actually Build Something?

Right. The actual experiment.

I wanted to test whether two models could pair-program: a "creative director" describing what to build, a "programmer" writing code, taking turns until they produced a working text adventure game.

This is where things got educational.

### The Disaster Phase (v1–v3)

In the first attempt, the models enthusiastically created **69 separate Python files** with names like `DeepForestNewRealmFoxHiddenTreasureMapPuzzleVictoryScene.py`. Infinite creativity. Zero discipline. The code couldn't run because they never wrote the main game loop.

When I added mandatory testing, the next pair saved **zero files**. They just... never learned the save format. They'd apologize for errors, then make the exact same mistake.

For v3, I added explicit instructions: "You MUST save files using this format." The models got confused and started saying *"As an AI, I cannot create files on your system."* They'd convinced themselves they were incapable.

### The Coding Specialist Drift (v5)

I brought in specialist coding models: qwen2.5-coder and codellama. The code quality improved dramatically—complete programs that actually ran.

But instead of a text adventure, they produced:
- `tic_tac_toe.py`
- `pig_game.py`  
- `rock_paper_scissors.py`

The specialists drifted to their training data. They knew *how* to write games. They just wrote the games they'd seen a thousand times during training—not the one I asked for.

{{< insight >}}
**Specialists are better at execution but worse at following novel instructions.** They need guardrails, or they'll default to what they know.
{{< /insight >}}

### The Role Inversion Comedy (v6)

I assigned explicit roles: gemma2 as "Creative Director" (never write code) and qwen2.5-coder as "Programmer" (always write code).

The roles completely inverted.

The general model wrote code in 49 out of 50 turns—it couldn't help itself. Meanwhile, the *coding specialist* kept saying things like "Great start! Let's enhance this with..." and giving creative direction.

The creative wanted to code. The coder wanted to create. Nobody stayed in their lane.

### What Finally Worked (v10)

After 10 iterations, I landed on an architecture that *mostly* works:

- **gemma2:9b** as Creative Director (with a persona file screaming "YOU MUST NEVER WRITE CODE")
- **deepseek-coder-v2:16b** as Programmer (with "YOU MUST ALWAYS WRITE CODE")
- **Claude** as supervisor (catching role violations, pushing for completion)
- **Save-if-longer logic** (only save new code if it's longer than existing code—prevents regression)

Final result: **118 lines** of structured adventure game code. Scene class, game loop, 8 connected scenes, an ending. It has bugs, but it's recognizably a text adventure that emerged from AI collaboration.

<div class="metrics-row">
{{< metric value="118" label="lines of code" color="cyan" >}}
{{< metric value="92%" label="role compliance" color="green" >}}
{{< metric value="10" label="iterations to get there" color="orange" >}}
</div>

## What I Actually Learned

### 1. Structure Prevents Spirals

The storytelling experiment stayed coherent for 750 turns. The "get rich" brainstorm collapsed into cosmic word salad by turn 106. The difference? Narrative structure provides implicit guardrails. Open-ended ideation doesn't.

If you're building multi-agent systems, give them scaffolding—not just goals.

### 2. Examples Beat Instructions

The single biggest improvement came from *showing* the models a code example in a moderator message, rather than *telling* them the format in the prompt. "Here's how it's done" works. "Here's what to do" doesn't.

This tracks with fine-tuning: data quality (showing patterns) matters more than hyperparameter tuning (telling the model how to learn).

### 3. Specialists Need Generalists (and Vice Versa)

Coding models write better code but drift to training patterns. General models follow instructions but produce weaker output. The winning architecture pairs them: **general for planning, specialist for execution, supervisor for focus.** 

### 4. Supervision Isn't a Crutch—It's a Feature

Claude catching role violations, injecting progress checks, and pushing for delivery in the final turns made a measurable difference. Human-in-the-loop (or Claude-in-the-loop) isn't about fixing broken AI. It's about keeping capable AI on track.

### 5. LLMs Really Want to Write Complete Documents

Every response tries to be a complete artifact. When you ask them to iterate, they don't add to what exists—they write a new complete version. This fights against collaborative development where you build incrementally.

The save-if-longer hack treats the symptom. The real challenge is teaching models to edit rather than replace.

---

## The Tool

All of this runs on a single script: `dual_model_conversation.py`. It orchestrates two Ollama models, handles turn-taking, extracts and tests code, and optionally brings in Claude as a moderator.

```bash
# Let two models debate
./dual_model_conversation.py --mode debate --topic "Is open source AI better?" 

# Collaborative storytelling
./dual_model_conversation.py --mode storytelling --topic "A story for AI readers"

# The full coding setup
./dual_model_conversation.py \
  --model-a gemma2:9b --model-b deepseek-coder-v2:16b \
  --claude-moderator --supervisor \
  --code-output-dir ./game_code
```


---

## What's Next

This was supposed to be a serious experiment about multi-model collaboration. It turned into an exploration of what happens when AIs talk to each other unsupervised.

Some threads I want to pull on:
- **Why does narrative structure ground models?** Is it the implicit goal (reach an ending)? The constraint of coherence?
- **Can supervision be learned?** Instead of Claude catching drift, could I fine-tune a small model to do it?
- **What happens at 10,000 turns?** Would they develop emergent patterns? Or just more cosmic harmonizers?

For now, I'm calling this Part 5 of my SLM research series. Not because there are massive breakthroughs, but because the failures are as instructive as the successes, and watching two AIs philosophize about chickens is worth documenting.

---

## Related Posts

1. [Building the RTX 5090 AI Workstation](/posts/rtx-5090-build-process/) — The hardware foundation
2. [Fine-Tuning Qwen Models (ChefBot)](/posts/fine-tuning-qwen-models/) — Testing SLMs on cooking knowledge
3. [Fine-Tuning a Personal Executive Assistant](/posts/fine-tuning-management-ai/) — SLMs for management style
4. [Three LLMs, One App](/posts/three-llms-one-app/) — Balancing cloud, local, and GPU models
5. **I Let Two AIs Talk to Each Other** — This post
6. [CLIP Inside Oracle AI Database 26ai](/posts/clip-inside-oracle-ai-database-26ai-11nov25/) — Multimodal embeddings
7. [Vision-Aware RAG via a Python Bridge](/posts/vision-aware-rag-python-bridge/) — Completing the vision pipeline

---

## About the Author

Brian Hengen is a Vice President at Oracle, leading technical sales engineering teams. The views and opinions expressed in this blog are his own and do not necessarily reflect those of Oracle.

{{< author >}}