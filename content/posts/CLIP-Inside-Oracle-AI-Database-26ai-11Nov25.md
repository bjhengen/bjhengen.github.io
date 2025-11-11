---
title: "CLIP Inside Oracle AI Database 26ai: Fast, Multimodal RAG"
date: 2025-10-28
draft: false
tags: ["AI", "Oracle Database 26ai", "CLIP", "Embeddings", "Cost Optimization", "ONNX", "Vector Search", "Multimodal AI"]
categories: ["AI Infrastructure"]
description: "How we got 10x faster by loading CLIP models directly into Oracle AI Database 26ai. Spoiler: In-database AI is production-ready."
---

After the [3-way LLM toggle](https://brianhengen.us/posts/three-llms-one-app/) went live, I turned my attention to **embeddings** - the invisible glue that powers search and RAG.

Oracle OCI GenAI’s Cohere endpoint had been rock-solid in my testing: fast, reliable, and gave me 80 K token context. But every chunk still meant a network round-trip, and **images were stuck behind OCR**, so text-only embeddings meant photos, diagrams, and whiteboards were blind spots in my knowledge base.

Then Oracle AI Database 26ai dropped. One of the announcements was improved multimodal support - `DBMS_VECTOR.LOAD_ONNX_MODEL` let me **load CLIP (text + image encoders) directly into the database** and generate embeddings **in SQL**.

No more API calls. No more OCR. Just **one transaction** that turns a PDF, PowerPoint, or snapshot into searchable, multimodal vectors.

This is the story of how **bringing AI into the database** unlocked true cross-modal search and made my app feel instant.

## The Problem: Death by a Thousand API Calls

My app stores everything in Oracle AI Database 26ai - from notes, PDFs, Word docs, to PowerPoint presentations. Every piece of content gets chunked and embedded for vector search. Originally, I used OCI's Cohere embedding API:

```javascript
// The old way: External API for every chunk
async function embedChunk(text) {
  const response = await fetch('https://api.cohere.ai/embed', {
    method: 'POST',
    headers: { 'Authorization': `Bearer ${COHERE_API_KEY}` },
    body: JSON.stringify({ texts: [text], model: 'embed-english-v3.0' })
  });
  const embeddings = await response.json();
  return embeddings.embeddings[0]; // 1024-dimensional vector
}

// Upload a 20-page PDF?
// 40-60 chunks × 200-500ms per API call = 8-30 seconds
```

It worked, but I knew I could make some improvements, based on my use case

## The Breakthrough: CLIP Inside Oracle AI Database 26ai

While digging deeper into Oracle AI Database 26ai for my [LLM toggle post](https://brianhengen.us/posts/three-llms-one-app/), I hit a **game-changer**: `DBMS_VECTOR.LOAD_ONNX_MODEL`.

Yes — **load a multimodal ONNX model directly into the database** and run inference **with pure SQL**.

My first thought: *Could I run CLIP — the multimodal brain — entirely in-database?*

I found pre-converted ONNX versions of **CLIP’s dual encoders**:

- **Text encoder** → 768-dim vector (120 MB)  
- **Image encoder** → 768-dim vector (293 MB)  

Both land in the **same semantic space**, meaning:

> A text query like *"Q3 strategy session"* can now **find actual photos** from that meeting.  
> An uploaded whiteboard snapshot can **surface related notes** — no OCR, no captions.

This wasn’t just faster.  
It was **true multimodal intelligence**, unlocked by running the model **where the data lives**.

A few hours later, I had both encoders loaded — and embeddings flowing **in SQL**.

The experiment was on.

## The Migration: From API to In-Database

### Step 1: Load the CLIP Model (20 Minutes)

I downloaded the ONNX model files and loaded them into Oracle using `DBMS_VECTOR.LOAD_ONNX_MODEL`:

```sql
-- Load CLIP text encoder (120 MB)
EXEC DBMS_VECTOR.LOAD_ONNX_MODEL(
  'CLIP_TEXT',
  'clip_txt.onnx',
  JSON('{"function": "embedding", "embeddingOutput": "embedding"}')
);

-- Load CLIP image encoder (293 MB)
EXEC DBMS_VECTOR.LOAD_ONNX_MODEL(
  'CLIP_IMAGE',
  'clip_img.onnx',
  JSON('{"function": "embedding", "embeddingOutput": "embedding"}')
);
```

That's it. 20 minutes for me to download and load. The models are now stored in the database, ready to use.

### Step 2: Update the Schema

Added a new column for CLIP embeddings (keeping the old Cohere embeddings during migration):

```sql
-- Add CLIP embedding column
ALTER TABLE document_chunks
ADD (chunk_vector_clip VECTOR(768, FLOAT32));

-- Both columns coexist during migration:
-- chunk_vector       : 1024-dim Cohere (old)
-- chunk_vector_clip  : 768-dim CLIP (new)
```

### Step 3: Generate Embeddings in SQL

Here's where it gets exciting. Instead of calling an external API, embeddings are generated directly in SQL:

```sql
-- Old way: Generate embeddings via API, then insert
-- (requires external call, network latency, API costs)

-- New way: Generate embeddings in SQL
INSERT INTO document_chunks (
  document_id,
  chunk_text,
  chunk_vector_clip
)
VALUES (
  :documentId,
  :chunkText,
  VECTOR_EMBEDDING(CLIP_TEXT USING :chunkText as INPUT)
);

-- For images:
INSERT INTO document_chunks (
  document_id,
  chunk_vector_clip
)
SELECT
  :documentId,
  VECTOR_EMBEDDING(CLIP_IMAGE USING d.file_content as DATA)
FROM documents d
WHERE d.document_id = :documentId;
```

**That's it.** No external API calls. No network latency. No API keys. Just SQL.

### Step 4: Migrate Existing Data

I had 1,723 existing chunks with my original embeddings. Re-embedding them took me about 2 minutes:

```sql
-- Batch 1: Chunks ≤4000 chars (direct embedding)
UPDATE document_chunks
SET chunk_vector_clip = VECTOR_EMBEDDING(CLIP_TEXT USING chunk_text as INPUT)
WHERE chunk_vector_clip IS NULL
  AND DBMS_LOB.GETLENGTH(chunk_text) <= 4000;
-- Result: 1,716 rows updated in 90 seconds

-- Batch 2: Large chunks >4000 chars (truncate to 3,800)
UPDATE document_chunks
SET chunk_vector_clip = VECTOR_EMBEDDING(
  CLIP_TEXT USING DBMS_LOB.SUBSTR(chunk_text, 3800, 1) as INPUT
)
WHERE chunk_vector_clip IS NULL;
-- Result: 7 rows updated in 8 seconds

-- Total: 1,723 embeddings in ~2 minutes
```

Compare that to what it would have taken me to re-embed via API: 1,723 chunks × 300ms average = **8.6 minutes** minimum, already a big improvement.

### Step 5: Update Search Queries

Search queries barely changed—just swap the column name:

```sql
-- Old Cohere search
SELECT chunk_text, document_name,
       VECTOR_DISTANCE(chunk_vector, :queryVector, COSINE) as similarity
FROM document_chunks dc
JOIN documents d ON dc.document_id = d.document_id
WHERE VECTOR_DISTANCE(chunk_vector, :queryVector, COSINE) < 0.7
ORDER BY similarity
FETCH FIRST 20 ROWS ONLY;

-- New CLIP search (in-database embedding generation)
SELECT chunk_text, document_name,
       VECTOR_DISTANCE(
         chunk_vector_clip,
         VECTOR_EMBEDDING(CLIP_TEXT USING :query as INPUT),
         COSINE
       ) as similarity
FROM document_chunks dc
JOIN documents d ON dc.document_id = d.document_id
WHERE VECTOR_DISTANCE(
  chunk_vector_clip,
  VECTOR_EMBEDDING(CLIP_TEXT USING :query as INPUT),
  COSINE
) < 0.7
ORDER BY similarity
FETCH FIRST 20 ROWS ONLY;
```

Notice the difference? The query embedding is generated **in the WHERE clause**. No preprocessing, no API calls - The Oracle AI Database generates the embedding on the fly.

## The Results: 10x Faster, Work Done in the Oracle AI Database

### Performance Comparison

| Metric | Cohere API | In-Database CLIP | Improvement |
|--------|------------|------------------|-------------|
| **Embedding Time** | 200-500ms | 10-50ms | ⚡ **10x faster** |
| **Network Latency** | 50-150ms | 0ms | ✅ Eliminated |
| **Vector Dimensions** | 1024 | 768 | 📊 25% smaller |
| **Storage per Chunk** | 4 KB | 3 KB | 💾 25% reduction |


### Search Query Comparison

```
Example: "Find meeting notes about customer engagements"

Old Architecture:
├─ Generate query embedding: 280ms (API call)
├─ Vector search: 45ms (database)
└─ Total: 325ms

New Architecture:
├─ Generate query embedding: 25ms (in SQL)
├─ Vector search: 45ms (database)
└─ Total: 70ms

Result: 4.6x faster, instantaneous user experience
```

The speed improvement comes from three factors:
1. **No network calls**: Eliminates 50-150ms latency per request
2. **Parallel processing**: Oracle's ONNX runtime is optimized for batch operations
3. **Locality**: Data and model live in the same database - no serialization overhead

## The Bonus: Multimodal Search

Here's where it gets really interesting. CLIP's text and image encoders output vectors in the same semantic space. That means I could now do cross-modal search:

### Text Query → Find Images

```sql
-- User asks: "Show me photos from customer meetings"
SELECT d.document_name, d.file_content,
       VECTOR_DISTANCE(
         dc.chunk_vector_clip,
         VECTOR_EMBEDDING(CLIP_TEXT USING 'customer meeting photos' as INPUT),
         COSINE
       ) as similarity
FROM document_chunks dc
JOIN documents d ON dc.document_id = d.document_id
WHERE d.document_type = 'IMAGE'
ORDER BY similarity
FETCH FIRST 10 ROWS ONLY;
```

Without any OCR or image captioning, the Oracle AI Database finds relevant images based on semantic meaning. A query like "person at beach" returns vacation photos; "datacenter equipment" returns server rack photos.

### Automatic Image Extraction from Documents

I extended the document processing pipeline to extract images from PDFs, Word docs, and PowerPoint presentations, then automatically generate CLIP embeddings:

```javascript
// Extract images from PowerPoint
async function processPowerPoint(fileBuffer, documentId) {
  const zip = await JSZip.loadAsync(fileBuffer);
  const images = [];

  // Extract all images from pptx
  const imageFiles = Object.keys(zip.files)
    .filter(name => name.startsWith('ppt/media/'));

  for (const filename of imageFiles) {
    const imageData = await zip.files[filename].async('nodebuffer');

    // Store image and generate CLIP embedding in SQL
    await executeQuery(`
      INSERT INTO documents (document_type, file_content, parent_document_id)
      VALUES ('IMAGE', :imageData, :documentId)
      RETURNING document_id INTO :newDocId
    `, { imageData, documentId });

    await executeQuery(`
      INSERT INTO document_chunks (document_id, chunk_vector_clip)
      SELECT :newDocId, VECTOR_EMBEDDING(CLIP_IMAGE USING d.file_content as DATA)
      FROM documents d WHERE d.document_id = :newDocId
    `, { newDocId });
  }
}
```

Now when I upload a PowerPoint with 26 photos, each image gets:
1. Extracted and stored as a child document
2. Automatically embedded with CLIP (in SQL)
3. Searchable via text queries
4. Linked to parent document

**Processing time**: 4.8 MB PowerPoint with 26 images = 3 minutes total (6-7 seconds per image). All embeddings generated in-database.

## The Architecture: Before and After

### Before (API-Dependent)
```
User uploads document
    ↓
Backend extracts text
    ↓
Backend chunks text → 60 chunks
    ↓
For each chunk:
    Backend → Cohere API (200-500ms)
    API → Returns 1024-dim vector
    Backend → Stores in Oracle
    ↓
Total: 12-30 seconds
```

### After (In-Database)
```
User uploads document
    ↓
Backend extracts text
    ↓
Backend sends chunks to Oracle
    ↓
Oracle (single SQL statement):
    - Chunks text
    - Generates CLIP embeddings (ONNX model)
    - Stores vectors
    - Extracts images (if any)
    - Generates image embeddings
    ↓
Total: 2-4 seconds
```

All the intelligence moved into the database. My backend became dramatically simpler.

## Code Changes: Surprisingly Minimal

The entire migration touched just a few files:

### Backend Changes
```javascript
// OLD: routes/documents.js
const embeddings = await cohereService.generateEmbeddings(chunks);
for (let i = 0; i < chunks.length; i++) {
  await db.query(`
    INSERT INTO document_chunks (chunk_text, chunk_vector)
    VALUES (:text, TO_VECTOR(:vector))
  `, { text: chunks[i], vector: JSON.stringify(embeddings[i]) });
}

// NEW: routes/documents.js
await db.query(`
  INSERT INTO document_chunks (chunk_text, chunk_vector_clip)
  SELECT :text, VECTOR_EMBEDDING(CLIP_TEXT USING :text as INPUT)
  FROM DUAL
`, { text: chunks });
```

That's it. Delete the external API call, use SQL instead. Applied this pattern across 6 route files:
- `routes/documents.js` - PDF uploads
- `routes/notes.js` - Note creation
- `routes/emails.js` - Email imports
- `routes/onenote.js` - OneNote processing
- `routes/notes_enhanced.js` - Rich text notes
- `routes/images.js` - Image uploads (new)

**Total lines changed**: ~150 lines removed (API calls), ~80 lines added (SQL embeddings). Net: **-70 lines of code**.

## Production Considerations

### Memory and Storage
- **CLIP models**: 413 MB total (120 MB text + 293 MB image)
- **Storage impact**: 25% reduction per chunk (768-dim vs 1024-dim)
- **Memory overhead**: Minimal—Oracle loads models once, reuses across queries

### Error Handling
With external APIs, every call can fail:
- Network timeouts
- Rate limits
- API downtime
- Invalid API keys
- Service deprecation

With in-database inference:
- Model is always available
- No network dependencies
- Deterministic performance
- No credential management
- Oracle's transaction guarantees

My error handling code dropped from 80 lines (retry logic, exponential backoff, circuit breakers) to 10 lines (standard SQL error handling).

## Lessons Learned

| Insight | Takeaway |
|---------|----------|
| **APIs Aren’t the Only Path** | Running inference **inside Oracle AI Database 26ai** removed network latency, credential churn, and external dependencies—no more waiting on API uptime. |
| **In-Database AI Is Ready for Prime Time** | Oracle’s ONNX runtime was **fast and stable in my testing, and SQL-native**. Load a model once (20 minutes), then every embedding is just a function call. |
| **Smaller, Specialized Models Win for My Use Case** | CLIP (3B params) worked great for embeddings of my data: **10–50 ms inference**, **120 MB footprint**, **same or better relevance**, and **native multimodal support**. Task-specific > general-purpose. |
| **Schema Evolution = Safety Net** | I kept both vector columns during migration (just to be safe):  

```sql
chunk_vector       VECTOR(1024, FLOAT32)  -- Old Cohere
chunk_vector_clip  VECTOR(768, FLOAT32)   -- New CLIP
```

This enabled:

- A/B quality testing
- Side-by-side result comparison
- Instant rollback
- Gradual cutover (no downtime)

After two weeks of validation, I'm ready to drop the old column. Zero regrets.

## What’s Next (Teaser for Part 3)

The CLIP migration unlocked **three immediate wins**—already live in my app:

1. **Image-Enriched Q&A**  
   Ask *"What risks did we discuss in the Q3 offsite?"* → the system pulls **meeting notes + actual photos** of the whiteboard, then feeds both to the LLM (with OCI Vision AI for extra visual analysis).

2. **Visual Document Search**  
   PowerPoints, Word diagrams, and PDFs are now searchable **by visual content**—no more “find the slide with the architecture drawing” manual hunt.

3. **True Cross-Modal Retrieval**  
   Text finds images. Images find text. No OCR. No captions. Just CLIP doing its magic.

I’ve delivered #1–3.  

**Part 3 of this blog series** will dive into the next frontier: **vision-aware inference at query time**—blending in-database CLIP with LLMs and local SLMs to *reason over images* during RAG, not just retrieve them.

Stay tuned.

## The Bottom Line

**Time invested**: One weekend (research → implementation → migration)

**Results**:
- ⚡ **10× faster embeddings** (350 ms → 35 ms average)  
- 🎯 **True multimodal search** (text ↔ images, no OCR)  
- 📦 **Simpler backend** (–70 lines of code, no API orchestration)  
- 🔒 **Rock-solid reliability** (no network, no keys, no downtime)

For a weekend’s work?  
**Absolute no-brainer.**

## Conclusion

Sometimes the best place for AI is **with your data**.

Oracle AI Database 26ai made that possible with **native ONNX support**, **SQL-level inference**, and **zero-compromise performance**.

APIs are powerful.  
**But** when your data lives in the database, **running the model there too** is often the smarter, faster, more secure path.

**Next up in Part 3**:  
How I built **vision-aware RAG** — combining in-database CLIP with LLMs and local SLMs to *reason over images* during inference.  We'll also cover how I overcame a snag in the inference process and why as useful as AI is, nothing beats a helpful colleague! 

---

**Have you moved ML into your database?**  
What trade-offs have you faced? Drop a comment or [reach out](https://brianhengen.us/)

## About the Author
Brian Hengen is a Vice President at Oracle, leading technical sales engineering teams. The views and opinions expressed in this blog are his own and do not necessarily reflect those of Oracle.