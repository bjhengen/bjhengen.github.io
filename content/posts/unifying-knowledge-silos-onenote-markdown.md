---
title: "Unifying 12 Years of Knowledge: From OneNote to Local Markdown to Oracle AI Database"
date: 2026-01-27
draft: false
tags: ["AI", "Oracle Database 26ai", "OneNote", "Markdown", "Knowledge Management", "MCP", "Vector Search", "Integration"]
categories: ["AI Infrastructure"]
series: ["Building a Knowledge Management App with Oracle AI Database 26ai"]
description: "How I migrated 5,178 OneNote documents spanning 12 years and built auto-sync for local markdown notes—turning scattered notes into a unified, AI-searchable knowledge base powered by Oracle AI Database 26ai."
---

{{< series-nav series="Building a Knowledge Management App with Oracle AI Database 26ai" part="4" total="4" >}}

{{< tldr >}}
Unified 12 years of scattered notes - 5,178 OneNote pages plus daily markdown journals - into a single AI-searchable knowledge base powered by Oracle AI Database 26ai. The result: a unified knowledge base where I can ask "What did I write about AI in 2021?" and get instant, accurate answers—regardless of where the note originated.
{{< /tldr >}}

After building [multimodal search with CLIP](https://brianhengen.us/posts/clip-inside-oracle-ai-database-26ai-11nov25/) and [vision-aware RAG](https://brianhengen.us/posts/vision-aware-rag-python-bridge/), I had a powerful system. But most of my knowledge was trapped in two silos:

1. **Microsoft OneNote**: 12 years of meeting notes, project docs, and random ideas (5,000+ pages)
2. **Local Markdown Editor**: My current daily journal and PKM system (growing daily)

Both tools are excellent for *capturing* thoughts. Neither is great for *finding* them six months later.

This is the story of how I unified these knowledge silos into Oracle AI Database 26ai, and why the database becoming the knowledge hub changes everything.

{{< learn >}}
- How to migrate massive OneNote repositories via Microsoft Graph API
- Building a real-time file watcher for local markdown files with Chokidar
- Memory management for vector embedding generation at scale
- Why Oracle AI Database 26ai is the perfect knowledge unification layer
{{< /learn >}}

## The Knowledge Silo Problem

### My Reality Before Integration

**Knowledge Scattered Across:**
- **OneNote** — 5000+ Pages
- **Local Markdown Editor** — Daily journals, linked references
- **PDFs, Word docs, PowerPoints...**

**The frustrating part**: I *knew* I'd written about a topic before. Finding it meant searching three different apps, hoping I'd remember which one I used.

Worse, none of these tools understood *context*. Searching for "AI" in OneNote returns every word with "AI" in it. My objective of creating the knowledge management app in the first place was to be able to intelligently manage my knowledge base, but I had a wealth of information I wanted to include, not just my **new** notes.

### The Vision: One Query, All Knowledge

What if I could ask:

> "What were the key projects I was involved in last year?"

And get results from:
- A markdown journal entry from September
- A OneNote meeting note from February
- A PowerPoint presentation I built

All ranked by semantic relevance, not keyword matching. All in one search.

**That's what I built.**

## Part 1: The OneNote Migration (5,178 Documents)

### The Challenge: 12 Years of Notes

Microsoft OneNote stores notes in a proprietary format. The traditional approach—exporting to PDF—loses metadata, breaks formatting, and requires manual intervention for each notebook.

I needed:
- **Direct API access** to OneNote's cloud storage
- **Metadata preservation** (creation dates, modification times)
- **Bulk processing** capability (thousands of pages)
- **Rate limiting handling** for Microsoft Graph API

### Choosing the Integration Path

I evaluated three approaches:

| Approach | Pros | Cons |
|----------|------|------|
| **PDF Export** | Already working | Manual, loses metadata |
| **MCP Server** | Clean abstraction | Additional process to manage |
| **Direct Graph API** | Full control | More code, but simpler deployment |

**Winner**: Direct Microsoft Graph API integration. The MCP pattern inspired the architecture, but for this use case, a direct service was simpler to deploy and maintain.

### The Authentication Dance

Microsoft's auth flow is very developer-friendly:

```javascript
// backend/services/onenoteDirect.js
async function authenticateWithDeviceCode() {
  // Request device code from Microsoft
  const response = await axios.post(
    'https://login.microsoftonline.com/consumers/oauth2/v2.0/devicecode',
    new URLSearchParams({
      client_id: CLIENT_ID,
      scope: 'Notes.Read User.Read offline_access'
    })
  );

  console.log(`
    ╔════════════════════════════════════════╗
    ║     OneNote Authentication Required     ║
    ╠════════════════════════════════════════╣
    ║  1. Go to: microsoft.com/devicelogin   ║
    ║  2. Enter code: ${response.data.user_code}              ║
    ║  3. Sign in with your Microsoft account ║
    ╚════════════════════════════════════════╝
  `);

  // Poll for completion
  return pollForToken(response.data.device_code);
}
```

No Azure app registration. No complex OAuth redirect flows. Just a code, a browser, and done.

### Handling Microsoft's Rate Limits

The Graph API has strict rate limits. My first attempt hit 429 errors within 30 seconds—requests 1-46 succeeded, then request 47 got throttled with "429 Too Many Requests."

**Solution**: Exponential backoff with `Retry-After` header support.

```javascript
async function fetchWithRetry(url, options, maxRetries = 5) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      const response = await axios.get(url, options);
      return response;
    } catch (error) {
      if (error.response?.status === 429) {
        // Microsoft tells us exactly how long to wait
        const retryAfter = error.response.headers['retry-after'] ||
                          Math.pow(2, attempt) * 1000;

        console.log(`Rate limited. Waiting ${retryAfter}ms...`);
        await sleep(retryAfter);
        continue;
      }
      throw error;
    }
  }
}
```

### The Recursive Section Group Problem

OneNote has a nested structure: Notebooks → Section Groups → Sections → Pages. Section groups can contain other section groups, creating arbitrary depth.

My first implementation only fetched top-level sections. Missing: hundreds of pages buried in nested groups.

```javascript
// Recursive section group fetching
async function fetchAllSections(notebookId, parentGroupId = null) {
  const sections = [];

  // Get sections at this level
  const directSections = await fetchSections(notebookId, parentGroupId);
  sections.push(...directSections);

  // Get section groups at this level
  const sectionGroups = await fetchSectionGroups(notebookId, parentGroupId);

  // Recursively fetch from each section group
  for (const group of sectionGroups) {
    const nestedSections = await fetchAllSections(notebookId, group.id);
    sections.push(...nestedSections);

    // Rate limiting delay between groups
    await sleep(3000);
  }

  return sections;
}
```

**Critical insight**: The 3-second delay between section groups wasn't optional—it was the difference between completing the migration and getting bounced from the API.

### Processing Pipeline: HTML → Text → Vectors

OneNote pages come as HTML. The embeddings I'm using in Oracle AI Database 26ai look for text chunks with CLIP embeddings.

```javascript
async function processOneNotePage(page, connection) {
  // 1. Fetch HTML content
  const htmlContent = await fetchPageContent(page.id);

  // 2. Convert HTML to clean text (preserve structure)
  const textContent = htmlToText(htmlContent, {
    wordwrap: false,
    preserveNewlines: true,
    tables: true  // Keep table structure
  });

  // 3. Store document with metadata
  const documentId = await insertDocument(connection, {
    documentName: page.title,
    documentType: 'NOTE',
    contentHtml: htmlContent,
    textContent: textContent,
    createdDate: page.createdDateTime,  // Preserve original date!
    modifiedDate: page.lastModifiedDateTime,
    sourceSystem: 'onenote',
    sourceId: page.id
  });

  // 4. Chunk and generate embeddings (in-database!)
  await generateChunksWithEmbeddings(connection, documentId, textContent);
}
```

The key insight: storing `createdDateTime` from OneNote means my 2012 notes show up with their *original* dates, not the import date. When I search "What was I working on in 2015?", I get accurate results.

### The Results: 5,178 Documents

{{< metric value="5,178" label="Documents Migrated" color="cyan" >}}

**Migration Statistics:**
- **OneNote**: 5,178 pages
- **Processing time**: ~4 hours
- **Vector embeddings**: 13,279 chunks
- **Storage**: 2.4 GB in Oracle AI Database 26ai
- **Searchable**: Immediately

**First query after migration**:
> "What were my top 3 projects in 2023?"

**Response**: Three relevant summaries, properly dated, with source attribution. *Magic.*

## Part 2: Local Markdown Real-Time Sync

### Why Local Markdown?

After OneNote, I switched to a local markdown-based PKM (Personal Knowledge Management) approach that stores everything on my laptop. Key differences:

- **Local-first**: Markdown files on disk
- **Bidirectional links**: `[[Page]]` references create connections
- **Daily journals**: Automatic date-based organization
- **Open format**: Plain text, no lock-in

But a local markdown editor is a *capture* tool. For search and AI-powered retrieval, I wanted everything in Oracle AI Database 26ai. Could I take my notes in markdown and have them automatically upload into the 26ai database and be available in my knowledge management system? ***YES!***

### The Architecture: File Watcher → Parser → Database

**Data Flow:**
1. **Markdown Directory** — `/journals/2025_12_18.md`, `/pages/Project Ideas.md`
2. **Chokidar File Watcher** — Monitors for changes
3. **Debounce (3 minutes)** — Batches rapid edits
4. **Sequential Processing Queue** — One file at a time
5. **Parse Markdown + Extract Tags/Links** — Structure extraction
6. **Generate CLIP Embeddings (in-database)** — Vector creation
7. **Oracle AI Database 26ai** — Final storage

### Challenge 1: Memory Management

My first implementation processed files in parallel. It crashed almost immediately:

```
FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed -
JavaScript heap out of memory
```

**Root cause**: Each file requires embedding generation. Parallel processing exhausted the 4GB default Node.js heap.

**Solution**: Sequential processing with explicit memory management.

```javascript
// backend/services/markdownWatcher.js
class MarkdownWatcher {
  constructor() {
    this.processingQueue = [];
    this.isProcessing = false;
  }

  async processQueue() {
    if (this.isProcessing || this.processingQueue.length === 0) return;

    this.isProcessing = true;

    while (this.processingQueue.length > 0) {
      const filePath = this.processingQueue.shift();

      try {
        await this.processFile(filePath);

        // Give garbage collector time to clean up
        await sleep(1000);

      } catch (error) {
        console.error(`Failed to process ${filePath}:`, error);
      }
    }

    this.isProcessing = false;
  }
}
```

**Critical server start command**:
```bash
node --max-old-space-size=8192 server-prod.js
```

The 8GB heap isn't optional—it's the difference between stable operation and random crashes.

### Challenge 2: Timezone-Safe Date Parsing

Journal files are named `2025_12_18.md`. Parsing this as a date seems trivial:

```javascript
// WRONG - Creates UTC midnight, displays as previous day in EST
const date = new Date("2025-12-18");
// Shows: December 17, 2025 (wrong!)
```

**The fix**: Use the Date constructor with explicit local components.

```javascript
// backend/services/markdownParser.js
function parseJournalDate(filename) {
  // filename: "2025_12_18.md"
  const match = filename.match(/(\d{4})_(\d{2})_(\d{2})\.md$/);
  if (!match) return null;

  const [, year, month, day] = match;

  // Create date in LOCAL timezone
  return new Date(
    parseInt(year),
    parseInt(month) - 1,  // Months are 0-indexed
    parseInt(day)
  );
}
```

This pattern applies everywhere you parse date strings into Date objects.

### Challenge 3: The Infinite Loop Bug

Small files (< 1000 characters) caused an infinite loop in the chunking logic:

```javascript
// BROKEN: Infinite loop when text.length < chunkSize
while (start < text.length) {
  const end = Math.min(start + chunkSize, text.length);
  chunks.push(text.slice(start, end));
  start = end - overlap;  // Never advances when end == text.length
}
```

**The fix**: Explicit termination condition.

```javascript
// FIXED: Handles small files correctly
while (start < text.length) {
  const end = Math.min(start + chunkSize, text.length);
  chunks.push(text.slice(start, end));

  // Break when we've processed the entire text
  if (end >= text.length) break;

  start = end - overlap;
}
```

### Extracting Markdown Graph Structure

Many markdown PKM tools support `[[wiki links]]` and `#tags`. I extract these during parsing:

```javascript
function parseMarkdownContent(content, filename) {
  // Extract wiki links: [[Page Name]]
  const wikiLinks = [...content.matchAll(/\[\[([^\]]+)\]\]/g)]
    .map(match => match[1]);

  // Extract tags: #tag or #[[multi word tag]]
  const tags = [...content.matchAll(/#(?:\[\[([^\]]+)\]\]|(\w+))/g)]
    .map(match => match[1] || match[2]);

  return {
    title: filename.replace('.md', ''),
    content: content,
    wikiLinks: [...new Set(wikiLinks)],
    tags: [...new Set(tags)],
    isJournal: filename.match(/^\d{4}_\d{2}_\d{2}\.md$/) !== null
  };
}
```

This metadata is stored in the database, enabling future features like "show all notes linking to this concept."

### The 3-Minute Debounce

Many markdown editors auto-save constantly. Without debouncing, every keystroke triggers a sync:

```javascript
// Too aggressive - causes constant reprocessing
watcher.on('change', async (path) => {
  await processFile(path);  // Runs on EVERY save
});
```

**Solution**: 3-minute debounce that resets on each change.

```javascript
const DEBOUNCE_MS = 180000; // 3 minutes

watcher.on('change', (path) => {
  // Reset the timer on each change
  if (this.debounceTimer) {
    clearTimeout(this.debounceTimer);
  }

  this.pendingChanges.add(path);

  this.debounceTimer = setTimeout(() => {
    this.processPendingChanges();
  }, DEBOUNCE_MS);
});
```

**Why 3 minutes?** It's long enough to batch multiple edits during active writing, but short enough that I don't forget I made changes.

{{< insight >}}
The debounce timer significantly impacts user experience. Too short (30s) and you're constantly processing partial thoughts. Too long (10min) and users wonder why their notes aren't syncing. 3 minutes is the sweet spot for my workflow.
{{< /insight >}}

## The Unified Result

### One Query, Multiple Sources

After both integrations, here's what happens when I ask:

> "What patterns have I noticed in customer architecture discussions?"

```sql
-- Behind the scenes: Oracle AI Database 26ai
SELECT
  d.document_name,
  d.source_system,
  d.created_date,
  dc.chunk_text,
  VECTOR_DISTANCE(
    dc.chunk_vector_clip,
    VECTOR_EMBEDDING(CLIP_TEXT USING :query as INPUT),
    COSINE
  ) as similarity
FROM document_chunks dc
JOIN documents d ON dc.document_id = d.document_id
WHERE d.source_system IN ('onenote', 'markdown', 'pdf', 'powerpoint')
ORDER BY similarity
FETCH FIRST 10 ROWS ONLY;
```

**Results come from**:
- A 2023 OneNote meeting note
- Yesterday's markdown journal entry
- A 2024 PowerPoint presentation
- A PDF whitepaper I uploaded last month

All ranked by semantic relevance. All with source attribution. All in < 200ms.

### The Numbers

{{< metric value="5,925" label="Total Documents" color="cyan" >}}
{{< metric value="13,279" label="Vector Chunks" color="pink" >}}
{{< metric value="<200ms" label="Search Latency" color="green" >}}

| Source | Documents | Span |
|--------|-----------|------|
| OneNote | 5,178 | 2012-2025 |
| Markdown | 47 | 2025-present |
| PDFs | 412 | Various |
| PowerPoints | 156 | Various |
| Images | 132 | Various |

### Why Oracle AI Database 26ai Is Optimal for This

I could have built this with a dedicated vector database (Pinecone, Weaviate, etc.). Here's why Oracle was the better choice:

1. **Single Source of Truth**: Documents, embeddings, metadata—all in one place
2. **ACID Guarantees**: No sync issues between vector store and metadata
3. **In-Database Embeddings**: CLIP runs in SQL, not external APIs
4. **SQL + Vectors**: Complex queries combining filters and similarity
5. **Existing Infrastructure**: Already using Oracle for everything else

The database isn't just *storing* my knowledge—it's the *integration layer* that makes diverse sources act as one.

## Lessons Learned

| Insight | Takeaway |
|---------|----------|
| **Rate Limits Are Real** | Microsoft Graph API will absolutely throttle you. Build retry logic from day one, not after you've been bounced. |
| **Memory Management Matters** | Vector embedding generation is memory-intensive. Sequential processing with explicit GC time prevents crashes. |
| **Preserve Original Dates** | Storing `createdDateTime` from source systems enables time-based queries. Don't use import timestamps. |
| **Debounce Aggressively** | File watchers that fire on every save create chaos. 3-minute debounce is the sweet spot for PKM tools. |
| **Test Timezones** | Date string parsing defaults to UTC. Always use `new Date(year, month-1, day)` for local dates. |

## What's Next

The unified knowledge base enables features that were impossible before:

1. **GraphRAG**: Entity extraction across all sources, building a knowledge graph of people, projects, and concepts (already implemented!)

2. **Daily Summaries**: Auto-generated executive briefs from markdown journals and recent documents

3. **Cross-Source References**: "Show me everything I've written about [topic] since [date]"

4. **Temporal Queries**: "What changed in my thinking about [concept] between 2020 and 2024?"

The foundation is complete. Now the fun begins.

---

**Have you unified knowledge silos?**
What sources did you integrate? What challenges did you face?
[Drop a comment or reach out](https://brianhengen.us/)

## About the Author

Brian Hengen is a Vice President at Oracle, leading technical sales engineering teams. The views and opinions expressed in this blog are his own and do not necessarily reflect those of Oracle.

{{< author >}}
