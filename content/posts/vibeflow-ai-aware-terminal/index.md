---
title: "I built a terminal that knows when your AI is waiting on you"
date: 2026-06-17T09:00:00-04:00
draft: false
tags: ["Rust", "AI", "Developer Tools", "Open Source", "Terminal", "Linux", "Agentic Development"]
description: "vibeflow is an open-source Linux terminal that shows, per tab, whether the AI agent inside is working or waiting on you — so you stop alt-tabbing to find the one that needs a reply."
cover:
  image: "/images/vibeflow_social_preview.png"
  alt: "vibeflow — an AI-aware Linux terminal"
  caption: "vibeflow"
---

{{< figure src="/images/vibeflow_social_preview.png" alt="vibeflow — an AI-aware Linux terminal" >}}

If you've been doing a lot of agentic development, here's a situation that probably sounds
familiar. You've got 5 tabs open, working on 5 different projects, running Claude Code, Codex,
maybe OpenCode.

As the coding agents improve, they work for longer, so naturally you multitask, getting another
project moving forward while the agents crank away on the others. The problem is that it's easy to
lose track of what's going on in the other tabs, and you end up clicking through them one-by-one
to see where they are and whether they're waiting on input. Then you come across one that finished
whatever you asked four minutes ago and has been sitting there waiting for an answer to its next
question. You don't notice, because every tab looks exactly the same. To make it worse, it's hard
to keep straight which project is running in which tab.

I know it sounds like a focus problem, but when I talk to others doing agentic development, the
stories are similar.

There are various ways to solve this problem. Good tools exist, but I do a lot of work directly in
Linux, and I couldn't find anything there that fit my needs, so I decided to build a terminal that
works the way I do and can help keep track of what's going on during working sessions.

## Terminals are blind on purpose

Modern terminal emulators are impressive: GPU-accelerated, beautiful font rendering,
fast. But they render *glyphs*. They don't know anything about what's happening inside them. To the
terminal, an agent thinking hard for two minutes and an agent that stopped and is waiting for your
input look identical: a process attached to a pty that happens not to be printing anything right
now.

That was completely fine for fifty years, because *you* were the one typing every command. You
always knew the state of your own session, because you were driving it. The moment you start
delegating work to agents that run for minutes, even an hour or more, and then quietly wait, the
terminal's blindness becomes the bottleneck.

## What vibeflow does

vibeflow is a Linux terminal emulator with one core idea: every tab carries a small indicator that
tells you what the program inside it is *doing*. It shows the state two ways at once: a thin
colored stripe down the tab's left edge, and the state spelled out in the label.

{{< figure src="vibeflow_claudewaiting_codexworking_ocwaiting.png" alt="vibeflow tab bar showing Codex working in blue between Claude and OpenCode waiting in amber" caption="Codex working (blue) between Claude and OpenCode waiting (amber). The amber tabs are the ones that need you." >}}

The stripe's color is the at-a-glance signal:

- **Blue**: the tool inside is working (generating, running commands).
- **Amber, gently pulsing**: it's waiting on you. This is the one that matters.
- **Gray**: idle at a prompt; a normal running command shows nothing special.

That's it. You glance at the tab bar and you know, at a distance, which conversation needs you
back, without alt-tabbing through all of them to check. The amber tab is the one to click.

## How it knows (the short version)

There are two ways vibeflow figures out a tab's state, and they stack.

The clean way is an open escape sequence, which I call **OSC 1338**, that a tool can emit to
announce its own state: "I'm working," "I'm waiting." Claude Code, for example, can be wired up
with a few hooks so it emits these as it goes. Any tool can adopt it; it's a published, documented
protocol, not a private feature.

The fallback, for tools that don't emit anything, is heuristic: vibeflow looks at which process is
in the foreground of each tab and watches its output. When a known AI tool goes quiet for a few
seconds after working, that's almost always "waiting on you," and the tab goes amber.

The first way is precise; the second works on day one with zero cooperation from the tool. If you
want the actual wire format, the detection timing, and the parts that were genuinely hard to get
right, plus the bugs daily use and a pre-launch audit surfaced, I wrote a
[deep-dive on how it works](https://brianhengen.us/posts/how-vibeflow-detects-ai-state/).

## The Build

vibeflow is free and open source (MIT/Apache-2.0), GPU-rendered in Rust, and runs on Linux under
both X11 and Wayland. It's v0.1.7: early, useful daily for me, and rough in the places early
software is rough.

I should also be upfront: I built it as one person, I've done a lot of work in Python, but I'm new
to Rust, and I leaned heavily on AI agents (Claude Code) to write it. That felt fitting for a tool
whose entire reason to exist is to help streamline agentic development, but it also means I'd
welcome more eyes on it.

## Try it

```
cargo install vibeflow
```

…or grab the single-file AppImage from the
[releases page](https://github.com/bjhengen/vibeflow/releases/latest). Source and docs are on
[GitHub](https://github.com/bjhengen/vibeflow).

If you build terminal tools or AI coding tools, I'd especially love for you to look at the
OSC 1338 protocol. The whole point of making it an open escape sequence instead of a private
feature is that it gets more useful the more tools and more terminals speak it.
