---
title: "Claude Memory V2: What I Learned Running AI Memory in Production"
date: 2026-02-18
draft: false
tags: ["ai", "claude", "mcp", "development", "architecture"]
summary: "Three weeks of running persistent memory for Claude Code taught me what actually matters—and led to a complete restructure."
---

It seems like every day (or even multiple times a day), I come across articles about people tackling what I'll call the "50 First Dates" problem of working with LLMs - that while they are increasingly capable, and context windows are larger and better managed, each session still starts with a - "refresh your memories about...". It works, but most implementations are limited - to a single machine, a format (such as Claude Code vs Claude Desktop), and the more involved they are, the longer they take to load and review (and the more context the recent memories themselves eat up).

I set out to improve on all of this - partially to be able to work across different formats - I might use Claude Code on my laptop, Mac Studio, or Linux server. I might want to interact with the same project in Claude Desktop, or maybe I'm traveling and I want to be able to use my phone - none of this works if you have local files for each project.

In my [last post](https://brianhengen.us/posts/building-persistent-memory-for-claude-code/) on this topic, I described building a persistent memory system for Claude - a cross-machine MCP server backed by a relational database with vector search capabilities that lets Claude remember lessons, project context, and infrastructure details across sessions.

That was v1. It worked. But three weeks of daily use revealed where the design was incomplete, where it was fragile, and what features were actually missing versus what I *thought* would be missing.

Before we get too far into it, let me say that what I built was for Claude, but anything that supports MCP should be able to follow a similar pattern. This is not something I would call "production-ready" today, but it is something I use constantly now and has dramatically changed and improved the way I interact with Claude.

This is the story of v2. 

## What Broke (And What Didn't)

### The Duplicate Project Problem

The first real issue was embarrassingly predictable. I'd log a lesson for "robotcar" on my Mac Studio, then reference "Robot Car" from my laptop, and Claude would create a whole new project entry. Same project, different capitalization, different database rows.

Within a week I had duplicate projects everywhere. The semantic search still worked—the vector search doesn't care about project boundaries—but `get_project` would return the wrong context depending on which name variant I used.

The fix required two things: case-insensitive project resolution and an alias system. Now "robotcar", "Robot Car", and "RobotCar" all resolve to the same project. The `project_aliases` table maps alternative names, and `resolve_project_id()` checks aliases before falling back to case-insensitive matching on the canonical name.

```sql
CREATE TABLE project_aliases (
    alias TEXT PRIMARY KEY,
    project_id INTEGER REFERENCES projects(id),
    created_at TIMESTAMP DEFAULT NOW()
);
```

Simple. Should have been in v1.

### The Stale Lesson Problem

By week two, I had lessons that were actively misleading. Early on I'd logged "MCP SDK 1.28.0 doesn't exist, use 1.26.0"—which was true at the time, but wouldn't stay true forever. Other lessons were just wrong—initial debugging guesses that I'd logged before finding the real cause.

v1 had no way to update or remove a lesson. Every `log_lesson` call was permanent. The only option was to log a *contradicting* lesson and hope the newer one ranked higher in search results. That's not memory—that's a junk drawer.

v2 adds three capabilities:

- **`update_lesson`** - Edit the title, content, tags, or severity of an existing lesson. When the content changes, the vector embedding is automatically regenerated so search stays accurate.
- **`retire_lesson`** - Soft-delete with a reason. Retired lessons are excluded from search by default but can be included if you want the full history.
- **`merge_projects`** - When duplicate projects slip through, merge one into the other. All lessons, sessions, journal entries, and metadata transfer to the surviving project, and the old name becomes an alias.

This sounds obvious in hindsight. But when you're building a system, you tend to focus on *adding* knowledge. ***The real challenge is maintaining it.***

### The Monolith Problem

v1 was a single 1,400-line `server.py`. Every tool, the database layer, auth, config—all in one file. It was fine for initial development. It was not fine for adding six new tools.

Before adding any v2 features, I split the server into modules:

```
src/
├── server.py      # Entry point, OAuth, lifecycle
├── config.py      # Configuration management
├── db.py          # Database pool, embeddings
├── auth.py        # OAuth provider
├── helpers.py     # Shared utilities (resolve_project_id)
└── tools/
    ├── search.py    # search, search_lessons
    ├── projects.py  # get_project, list_projects, CLAUDE.md tools
    ├── infra.py     # connectivity, machines, containers
    ├── lessons.py   # log_lesson, log_pattern, update, retire
    ├── sessions.py  # start_session, end_session
    ├── journal.py   # write_journal, read_journal
    └── admin.py     # project state, guardrails, permissions
```

This was the right call. Adding the six new v2 tools was straightforward after the refactor. Each module owns its domain, imports the shared `mcp` instance, and registers tools via decorators.

## The Feature I Didn't Expect to Need: Project CLAUDE.md

Claude Code uses `CLAUDE.md` files for project-specific instructions. They work well for static rules ("always run `flutter clean` before builds"). But they're local files—they don't travel across machines, and they can't be updated programmatically.

v2 adds a `claude_md` column to the projects table and three tools to manage it:

- **`get_project_claude_md`** - Retrieve the stored CLAUDE.md for a project
- **`set_project_claude_md`** - Create or replace it entirely
- **`update_project_claude_md`** - Update a specific section by heading match

This means I can store project philosophy, architectural decisions, and active development context in a place that's accessible from any machine. The local CLAUDE.md files still exist for machine-specific settings, but the shared version captures the living context that changes as a project evolves.

The section-level update tool was particularly useful. Instead of replacing the entire document to update the "Current Focus" section, Claude can surgically update just that heading while leaving the rest untouched.

## The V2 Development Process

The entire v2 was planned and executed in a single session with Claude. Here's what that looked like:

1. **Design doc** - Started with a design document covering the three features (project CLAUDE.md, lesson lifecycle, name normalization), the migration strategy, and the refactoring plan
2. **Implementation plan** - Broke it into 11 tasks with dependencies
3. **Migration infrastructure** - Created a `migrations/` directory with numbered SQL files, applied via a shell script
4. **Refactor first** - Split the monolith before adding features
5. **Features** - Added new tools one module at a time
6. **Deploy** - Zero-downtime migration; existing clients didn't need reconfiguration

The whole thing—design, implementation, testing, deployment—took about three hours. The migration itself took under a minute. No data loss, no client disruption - and let's not forget backups - everything (especially the data) was backed up before any changes were made.

The key insight: **refactor before you add features, not after.** The monolith-to-modules refactor made every subsequent step cleaner.

## What 24 Tools Looks Like

v1 had 16 tools. v2 has 24. Here's the full inventory:

| Category | Tools |
|----------|-------|
| **Search** | `search`, `search_lessons` |
| **Projects** | `get_project`, `list_projects`, `get_project_claude_md`, `set_project_claude_md`, `update_project_claude_md` |
| **Infrastructure** | `get_connectivity`, `list_machines`, `add_machine`, `add_container` |
| **Lessons** | `log_lesson`, `log_pattern`, `update_lesson`, `retire_lesson` |
| **Journal** | `write_journal`, `read_journal` |
| **Sessions** | `start_session`, `end_session` |
| **Admin** | `update_project_state`, `check_guardrails`, `add_project`, `get_permissions`, `merge_projects` |

One thing I've noticed: the tools I use most aren't the ones I expected. `search` and `get_project` are the workhorses. `check_guardrails` fires automatically before builds. But `log_lesson`—the tool I originally built the system *for*—gets used less often than I thought. Most sessions don't produce genuinely novel insights. The ones that do tend to be about deployment gotchas, not code patterns.

## Honest Assessment: Where This Sits

After building and running this for a month, I've spent some time looking at what else is out there—mem0, Zep, Letta (MemGPT), and the various file-based memory solutions.

**What this does better:** It's domain-modeled for software development. It knows what a "project" is, what a "lesson" is, what a "guardrail" is. Generic memory systems store text and search it. This system understands the structure of developer knowledge.

**What those do better:** Automatic memory formation. Systems like Letta extract and consolidate memories from conversations without explicit logging. My system relies entirely on Claude (or me) deciding to call `log_lesson`. Things can fall through the cracks.

**What nobody does well yet:** Memory maintenance. Detecting contradictions, aging out stale knowledge, consolidating 50 related lessons into a coherent summary. This is the hard unsolved problem.

## What's Next: V3 Ideas

Based on a month of production use, here's what I'm thinking about:

1. **Automatic memory formation** - Can the system extract lessons from conversations without explicit `log_lesson` calls? This is the biggest gap.
2. **Memory decay and relevance scoring** - A lesson from six months ago shouldn't have the same weight as one from yesterday. Track retrieval frequency, add time-based decay.
3. **Conflict resolution** - If two lessons contradict each other, detect and surface the conflict. Today this fails silently.
4. **Consolidation** - When you have 50 lessons about Flutter, synthesize them into a condensed "Flutter knowledge" document automatically.
5. **Multi-user support** - The architecture is single-user today. Team memory is a different beast entirely.

These are hard problems. But the foundation is solid, and the structured data model makes them more tractable than they'd be in a generic vector store.

The biggest lesson from v2 isn't technical. It's this: **the hard part of memory isn't storage or retrieval. It's maintenance.** Building a system that remembers everything is easy. Building one that remembers the *right* things—and forgets the wrong ones—is the actual challenge.

## What Else Am I Working On?

This project has made me fascinated with what enhanced memory systems could enable for both LLMs and SLMs (Specialized Language Models). I'm working through some experiments and will have more to share soon.

And of course, robotcar that I teased at the beginning of this post - more to come on that soon - just need to get some sensors installed and we'll be off to the races!

---

*This post was written with Claude's help, drawing on lessons stored in the very system it describes. The meta hasn't worn off yet.*

## About the Author
Brian Hengen is a Vice President at Oracle, leading technical sales engineering teams. The views and opinions expressed in this blog are his own and do not necessarily reflect those of Oracle.
