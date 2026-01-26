---
title: "Building Persistent Memory for Claude Code"
date: 2026-01-26
draft: false
tags: ["ai", "claude", "mcp", "development", "postgres", "vector-search"]
summary: "How I built a cross-machine memory system for Claude Code using MCP, PostgreSQL, and semantic search."
---

If you've spent any time working with AI coding assistants, you've probably experienced this frustration: you have a productive session, solve a tricky problem, learn something important about your codebase—and then the next day, the AI has forgotten everything.

If you work across multiple projects and multiple machines, it can get even more frustrating - sometimes I want to use the desktop app, sometimes the command line, and some work necessarily spans machines. In my case, I work across multiple machines (a Mac Studio, a Linux AI workstation, and a laptop) and multiple projects. Every new Claude Code session felt like starting from scratch, aside from the local markdown files used for local/project level "memory". I'd often re-explain my project structure, re-discover the same gotchas, and occasionally make the same mistakes twice.

Lots of folks are working on addressing this - some elegant file-based solutions are out there, but at heart, I'm a database guy, and this seems like exactly the type of problem databases can help solve. Since I'm in a serious learn & build (or build & learn) phase, I decided to build my own solution - a persistent memory system that lets Claude remember lessons learned, project context, and infrastructure details across sessions and machines.

## The Problem

Claude Code is excellent at understanding code and helping with development tasks. But it has a fundamental limitation: each conversation starts fresh. There's no built-in way to:

- Remember that `debugBypass` must be `false` in production (learned the hard way)
- Know that OpenAI's vision API blocks wine-related content (use Claude instead)
- Recall which SSH command connects to which server
- Track what was accomplished in previous sessions

I'd documented some of this in markdown files, but Claude still needed to be told to read them. And cross-project knowledge (like "always use `sharePositionOrigin` on iOS Share Sheets") didn't transfer automatically.

## The Solution: MCP + Semantic Search

Claude Code supports the [Model Context Protocol (MCP)](https://modelcontextprotocol.io/), which allows external tools to extend its capabilities. ***I built my own custom MCP server*** that provides:

1. **Semantic search** across all stored knowledge using vector embeddings
2. **Structured project context** (tech stack, key files, approaches)
3. **Infrastructure connectivity** (SSH commands, containers, databases)
4. **Guardrails** (warnings before dangerous operations)
5. **Session tracking** (what was done, what's pending)

The architecture is straightforward:

```
┌─────────────────────────────────────────────────────────┐
│  CLOUD COMPUTE                                          │
│  ┌──────────────────┐    ┌───────────────────────────┐  │
│  │  DB - relational │◄───│  claude-memory MCP server │  │
│  │  + vector        │    │  (Python + FastMCP)       │  │
│  └──────────────────┘    └───────────────────────────┘  │
│         Docker Compose                                  │
└─────────────────────────────────────────────────────────┘
         ▲              ▲              ▲
         │ HTTPS        │ HTTPS        │ HTTPS
   ┌─────┴─────┐  ┌─────┴─────┐  ┌─────┴─────┐
   │Mac Studio │  │workstation│  │ Laptop.   │
   │(Claude    │  │(Claude    │  │(Claude    │
   │ Code)     │  │ Code)     │  │ Code)     │
   └───────────┘  └───────────┘  └───────────┘
```

## How It Works

### Storing Knowledge

The system stores several types of information:

**Lessons** - Things learned the hard way:
```
Title: "iOS Share Sheet requires sharePositionOrigin"
Content: "On iOS, the Share Sheet will crash on iPads if
sharePositionOrigin is not provided. This is required for
ALL iOS devices, not just iPads."
Tags: ["flutter", "ios", "share-sheet"]
Severity: critical
```

**Patterns** - Reusable solutions:
```
Name: "GoRouter Dialog Pattern"
Problem: "Dialogs don't work correctly with GoRouter"
Solution: "Use navigatorKey with currentContext pattern..."
Code: [example code]
```

**Project Context** - What Claude needs to know:
```
Project: wine.dine
Tech Stack: Flutter 3.32.5, FastAPI, Provider
Key Files:
  - subscription_gate.dart:70 (debugBypass - MUST be false!)
  - pubspec.yaml:19 (version number)
Guardrails:
  - Verify debugBypass = false before building (critical)
```

### Semantic Search

Each lesson and pattern is embedded and stored as a vector in the database. This enables natural language queries:

```
Claude: search("share sheet crashing on ipad")
→ Returns: "iOS Share Sheet requires sharePositionOrigin" (0.89 similarity)
```

The search considers both exact matches and semantic similarity, so "file picker not working on tablet" might still surface the Share Sheet lesson.

### MCP Tools

Claude gets access to 16 tools:

| Tool | Purpose |
|------|---------|
| `search` | Semantic search across all knowledge |
| `search_lessons` | Find lessons with filters (project, tags, severity) |
| `get_project` | Get full project context |
| `list_projects` | List all projects by status |
| `log_lesson` | Save something learned |
| `log_pattern` | Save a reusable solution |
| `start_session` | Begin tracking a work session |
| `end_session` | Complete session with summary |
| `update_project_state` | Update focus, blockers, next steps |
| `get_connectivity` | SSH commands, containers, databases |
| `list_machines` | Show all registered machines |
| `add_machine` | Register a new machine |
| `add_container` | Register a Docker container |
| `add_project` | Register a new project |
| `check_guardrails` | Verify safety before actions |
| `get_permissions` | Check permissions for a scope |

### Cross-Machine Access

The MCP server runs on cloud compute and is accessible via HTTPS. Each machine has Claude Code configured to connect:

```bash
claude mcp add -s user -t http claude-memory \
  https://your-server.com/mcp \
  -H "Authorization: Bearer <api-key>"
```

*(The actual endpoint URL is not shown for security reasons.)*

Now whether I'm on my Mac Studio, Linux workstation, or laptop, Claude has access to the same knowledge base.

## What I've Learned So Far

### Things That Work Well

1. **Cross-project knowledge transfer** - A lesson learned in one Flutter project now surfaces in all Flutter projects
2. **Infrastructure recall** - "How do I SSH into the project server?" is answered instantly
3. **Guardrails** - Claude can check "should I verify anything before building?" and get project-specific warnings

### Things Still Being Evaluated

1. **When to log vs. when to skip** - Not every session produces insights worth persisting
2. **Search relevance tuning** - Sometimes semantic search returns tangentially related results
3. **Session boundaries** - Defining what constitutes a "session" worth tracking

### Unexpected Benefits

- The CLAUDE.md files I maintain on each machine are now simpler—they point to the memory system instead of duplicating information
- I can search across all my projects: "Have I solved X before?"
- The system serves as documentation that Claude actually uses

## Technical Implementation

The core components:

**Database Schema** (relational + vector):
- `machines` - Registered machines with SSH details
- `projects` - Projects with tech stack, status, phase
- `lessons` - Lessons with vector embeddings
- `patterns` - Reusable solutions with embeddings
- `guardrails` - Safety checks per project
- `sessions` - Session tracking with items

**MCP Server** (Python + FastMCP):
- HTTP transport for cross-machine access
- API key authentication
- Starlette + uvicorn for serving
- asyncpg for database access

**Deployment**:
- Docker Compose on cloud compute
- nginx reverse proxy for HTTPS
- Separate from other workloads

The code is available at [github.com/bjhengen/claude-memory](https://github.com/bjhengen/claude-memory).

## What's Next

This is very much a v1. Things I'm considering:

1. **Automatic lesson extraction** - Can Claude suggest lessons to log based on the conversation?
2. **Relevance feedback** - Track which search results were actually useful
3. **Project relationships** - "project a" and "project b" share a backend
4. **Time decay** - Should old lessons fade in relevance?

## Try It Yourself

If you're interested in building something similar:

1. Clone the repo: `git clone https://github.com/bjhengen/claude-memory`
2. Configure `.env` with your credentials
3. Deploy with Docker Compose
4. Add the MCP server to Claude Code: `claude mcp add -s user -t http ...`

The system is designed to be self-hosted. Your knowledge stays on your infrastructure.

---

*This post was written with Claude's help—using the very memory system it describes. It's one piece of a larger knowledge management rebuild I've been working on, unifying years of scattered notes into something AI can actually use. Meta, I know.*
