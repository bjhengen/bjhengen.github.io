---
title: "The Prompt Is the Program"
date: 2026-03-26
draft: false
tags: ["AI", "Robotics", "LLM", "Raspberry Pi", "AIOS", "RTX 5090", "Qwen"]
categories: ["AI Experiments"]
series: ["AIOS ThinkTank"]
description: "Operating systems were built for humans. What happens when the user is an AI? I built a robot car driven entirely by an LLM to find out."
cover:
  image: "/images/robotcar.png"
  alt: "AIOS:ThinkTank robot car with mecanum wheels, Raspberry Pi, camera, and ultrasonic sensors"
  caption: "AIOS:ThinkTank — an LLM-driven robot car"
---

{{< tldr >}}
- Built a robot car controlled entirely by an LLM (Qwen3.5-27B on an RTX 5090) — no traditional control code
- The AI sees through a camera, reads ultrasonic sensors, and issues direct motor commands every ~1.3 seconds
- It follows me through the house autonomously for 15+ minute sessions — and has crashed into a plant pot twice
- Key insight: qualitative reasoning goes to the AI, quantitative enforcement goes to traditional code
- The operating system is optimized for a user that isn't there anymore
{{< /tldr >}}

*Operating systems were built for humans. What happens when the user is an AI? I built a robot car to find out.*

---

## The Question No One's Asking

Every modern operating system you've ever used was designed for you.

For *you* — a person who needs windows to look at, a mouse to point with, a file system organized into folders because that's how we like to think about documents - electronic filing cabinets. Even Linux, the developer's OS, is generally built around the assumption that a person is sitting at a terminal typing commands and reading output.

This made perfect sense for sixty years. Computers are powerful but don't take action on their own. Humans are smart and we have certain ways we like to work. The operating system is the translator between them — abstracting the hardware and making it feel familiar - taking our intent ("open that file," "run that program," "print that page") and converting it into the precise sequence of hardware operations the machine actually needs.

But what happens when the thing controlling the computer isn't human anymore?

---

## The AI Doesn't Need Your Desktop

An AI doesn't need a graphical interface. It doesn't need a file browser, a taskbar, or a notification system. It doesn't need font rendering, window management, or keyboard input handling. It doesn't read man pages or need help text.

An AI also doesn't need most of what happens under the hood. Process scheduling optimized for interactive responsiveness? That's a human-facing concern. A permission system based on user accounts? Users are a human concept. An audio subsystem? The AI doesn't have ears.

To be clear: I don't think operating systems go away. Device drivers aren't optional. Hardware abstraction is still valuable. Networking stacks still need to move packets. The kernel isn't the problem.

What will drive this change is that everything *above* the kernel assumes the user is a person. The interfaces, the metaphors, the scheduling priorities, the entire interaction model. That's the layer that will be rebuilt — not for humans, but for AI.

And here's what I think that means for the rest of us: AI becomes the new interface. We stop managing machines directly and start managing AI that manages machines - and that is already happening, it seems like every day there is a new announcement about an AI that is tuned for full computer control. You will describe what you want. The AI figures out how to make the hardware do it. **The prompt becomes the program.**

I already work this way when I code. I use Claude Code and OpenAI's Codex as my development team — I look at my role more as the product manager, defining requirements and reviewing output. I don't write implementation code. I describe what the system needs to do, evaluate the result, and iterate. It's product management, not programming.

The question is: does that same pattern work all the way down to the hardware? Can you PM a robot car?

I built something to test it.

---

## AIOS:ThinkTank

AIOS:ThinkTank is a robot car. Four mecanum wheels for omnidirectional movement, a Raspberry Pi 4 for hardware control, a camera for vision, five ultrasonic sensors for spatial awareness. The brain is a large language model — currently Qwen3.5-27B — running on a server that sits under my desk, powered by an NVIDIA RTX 5090.

The architecture is simple: the Pi captures camera frames and sensor data, sends them to the AI, and executes whatever motor commands come back. The AI sees through the camera, feels through the sensors, and moves through the wheels. There is no traditional control program. No state machine. No hand-coded navigation logic. I like to think of it as a remote controlled car, but the LLM is the one doing the driving instead of me.

The car drives because the AI looks at what the camera sees and decides what to do. The prompt is the program.

Here's what the AI receives every ~1.3 seconds:

```
SENSORS: FC=24cm [WARNING <30cm] FL=68cm FR=45cm RL=284cm RR=157cm
KNOWN LOCATIONS: living_room (carpet), hallway (tile)
Current location: hallway

[camera frame attached]

You are driving a robot car. Describe what you see, assess the 
navigation situation, and issue a motor command.
```

And here's what it returns:

```
OBSERVATION: I see a long narrow space with tile flooring and white 
walls. A doorway is visible ahead-left. The human is walking away 
approximately 2 meters ahead.
LOCATION: hallway/tile
COMMAND: FORWARD 200 200 800
REASONING: Human is ahead and moving. Maintaining follow distance. 
Clear path ahead, no obstacles within sensor range.
```

That response gets parsed, the motor command gets executed for 800 milliseconds, and the loop repeats. No PID controller. No SLAM algorithm. No path planner. Just a language model looking at photos, reviewing sensor data, deciding what to do, and then sending the direct motor controls down to the car.

---

## It Actually Works

The car can follow me through the house. Living room to kitchen, around the dining table, past the cat tree, down the hallway. It tracks me through doorways, recovers when it loses sight of me around corners, and navigates furniture. It's done fifteen-minute sessions of continuous autonomous following.

It has also crashed into a plant pot. Twice. And a few walls.

I'm not going to oversell this. The car processes about one frame per second. Its reaction time on corners is nearly four seconds. It gets confused by uniform surfaces and occasionally tries to drive through chair legs. It's slow, cautious, and sometimes just sits there thinking while I walk away.

But it *navigates*. Without a single line of traditional control code. The AI handles the full loop: perception, reasoning, decision, action. And it does it in a physical environment it was never trained on, with hardware it was never designed for, using a general-purpose language model that also knows how to write poetry and explain calculus.

The interesting part isn't that it works *well*. It's that it works *at all*.

---

## What We're Learning

Three months of building and testing have surfaced patterns that I think matter beyond this specific robot car.

### LLMs Think in Vibes, Not Volts

The prompt says "use speed 150 for rotations." The AI uses 230. Every time. Across every model we've tested.

Large language models treat specific numbers in prompts as suggestions, not constraints. They pattern-match to what "seems right" based on training data — which contains zero entries about this specific robot car. If you need exact parameters, you enforce them in code, not in the prompt.

We (me and Claude Code, in this case) built a sanitizer that caps speeds, limits durations, and boosts power on carpet. The AI decides *what* to do. The code decides the *bounds* of how it's done. This division of labor — qualitative reasoning to the AI, quantitative enforcement to traditional software — keeps showing up as the right pattern.

### Reflexes Below, Reasoning Above

The AI thinks at ~1 Hz. The ultrasonic sensors check for obstacles at 20 Hz. If we waited for the AI to notice a wall and decide to stop, the car would hit the wall twenty times before the AI finished its sentence.

So we built a reflex layer. A background thread monitors sensors and triggers emergency stops independently of the AI. The AI handles "go toward the kitchen." The reflex system handles "don't hit the wall." This layered architecture — fast reflexes underneath slow reasoning — wasn't in the original plan. It emerged from necessity.

### The OS Is Optimized for the Wrong User

The Pi runs Raspberry Pi OS Lite. That's a full Linux kernel, systemd, an SSH daemon, a logging framework, package management, a networking stack. The car uses almost none of it. Its actual job is three functions: read sensors, capture frames, execute motor commands. Everything else is infrastructure designed for human system administration.

The OS isn't broken. It's just designed for a user that isn't there. The logging framework writes human-readable text files. Systemd manages services with human-friendly unit files. All useful — for a human operator.

An AI operator would want different things. Faster hardware access with less abstraction overhead. A communication protocol optimized for structured data, not terminal sessions. Resource management tuned for inference cycles, not interactive responsiveness. The bones of the OS — drivers, memory management, networking — stay. The human-facing skin gets replaced with an AI-native interface.

---

## Where This Goes Next

The proof of concept works. An AI can control physical hardware through natural language reasoning without traditional control software. Now the real questions begin.

**Can we rebuild the OS layer for AI?** The Pi runs a full Linux distribution designed for human administrators. What happens if we replace the human-facing layers with AI-native interfaces — faster hardware access, structured communication protocols, resource management tuned for inference rather than interactivity? The kernel and drivers stay. Everything above them gets rethought. Is it a new OS altogether, or do we strip down Linux to the very bare bones and write the control programs in C or Assembly instead of Python?

**Can we specialize the AI?** The car is driven by a general-purpose 27-billion-parameter model that also knows about Shakespeare and organic chemistry. Every driving session generates training data — images, decisions, outcomes. What happens when we fine-tune a smaller model specifically for this car, in this house, on this hardware? How small can the model get before it stops driving well? How fast can we get to process tokens on my hardware?

**Can the AI build its own abstractions?** The car recently started building a topological map — a graph of rooms and doorways, with recorded motor commands for each transition. Two nodes and one edge so far. But if the map grows, the car could navigate to named rooms without following a human. The AI would be creating its own abstraction layer over the physical environment — which is, when you think about it, exactly what an operating system does.

---

## Following Along

This is the first in a series documenting the build. The next posts get into specifics — hardware assembly, the driving sessions, the sensor integration, the moment Claude drove the car and met Ollie the bernedoodle.

The project is public:

- **Code:** [github.com/bjhengen/aios-thinktank](https://github.com/bjhengen/aios-thinktank)
- **Blog:** More posts coming on [brianhengen.us](https://brianhengen.us)
- **Updates:** [LinkedIn](https://linkedin.com/in/brian-hengen) for video clips and shorter-form writeups

The question we're testing: when AI becomes the primary user of a computer, what does the operating system become? Three months in, I think the answer is clear — the OS doesn't disappear, but it gets fundamentally re-optimized for a user that doesn't have eyes, fingers, or need for human metaphors. The next few months will show what that re-optimization looks like in practice.

---

*This is the AIOS:ThinkTank build series — a robot car that asks: what if the AI isn't just assisting the operating system, but IS the operating system?*

## About the Author
Brian Hengen is a Vice President at Oracle, leading technical sales engineering teams. The views and opinions expressed in this blog are his own and do not necessarily reflect those of Oracle.

{{< author >}}
