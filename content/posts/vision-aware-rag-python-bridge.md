---
title: "When the SDK Says No: Vision-Aware RAG via a 300-Line Python Bridge"
date: 2025-11-22
draft: false
tags: ["AI", "Oracle Database 26ai", "Grok-4", "Vision AI", "RAG", "Microservices", "Python", "ONNX", "OracleAIDatabase"]
categories: ["AI Infrastructure"]
description: "After CLIP made multimodal search instant, I hit a wall: the LLM could find images but couldn't see them. Here's how an Oracle engineer's tip and 300 lines of Python broke through."
---

*This is Part 3* — the final chapter in turning a text-only knowledge base into a fully vision-aware RAG system

After [moving CLIP embeddings into Oracle AI Database 26ai](https://brianhengen.us/posts/clip-inside-oracle-ai-database-26ai-fast-multimodal-rag/), I had something great: **true multimodal search**. Text queries found images. Images found related text. No OCR, no captions—just CLIP doing its magic in the database. 

But I quickly hit a wall.

When I asked my knowledge management app: *"What color is the car in this photo?"*

The response: *"I don't have information about the color of any vehicles in the available documents."* but, it would still retrieve the picture - it just couldn't comprehend it and integrate it into the response.

Wait. The **CLIP search found the image**. The app showed me the photo. But the LLM couldn't *see* it.

I had **retrieval**. What I needed was **reasoning**.

This is the story of how a great conversation with a Director of Product Management at Oracle AI World, a polyglot microservice, and 300 lines of Python turned multimodal search into true vision-aware RAG.

## The Retrieval vs. Reasoning Gap

Let me show you what was happening under the hood.

### What CLIP Gave Me (The Good Part)

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
  AND VECTOR_DISTANCE(
        dc.chunk_vector_clip,
        VECTOR_EMBEDDING(CLIP_TEXT USING 'customer meeting photos' as INPUT),
        COSINE
      ) < 0.85
ORDER BY similarity
FETCH FIRST 5 ROWS ONLY;

-- Result: 5 relevant images found in 45ms ✅
```

Perfect. CLIP's semantic search worked beautifully. Text queries found visually relevant images, and the app displayed them to the user.

### What the LLM Saw (The Problem)

But when I passed those images to the LLM for Q&A, here's what the context looked like:

```javascript
// What the LLM actually received
const context = `
Based on the following documents, answer the user's question.

RELEVANT DOCUMENTS:
- customer_meeting_photo_1.jpg (image file)
- customer_meeting_photo_2.jpg (image file)
- customer_meeting_photo_3.jpg (image file)

USER QUESTION: What color shirt was the customer wearing in the meeting?
`;

// LLM's response: "I don't have information about clothing colors in the documents."
```

The LLM saw **filenames**. Not pixels. Not visual content. Just text strings.

It was like asking someone to describe a photo while only telling them the filename. Useless.

### The Architecture Gap

```
Before (Retrieval Only):
User: "What color is the car?"
    ↓
CLIP Search → Finds car_photo.jpg (similarity: 0.92) ✅
    ↓
LLM Context: "Found document: car_photo.jpg" ❌
    ↓
LLM: "I don't have information about vehicle colors"
    ↓
User: 😤

What I Needed (Retrieval + Reasoning):
User: "What color is the car?"
    ↓
CLIP Search → Finds car_photo.jpg ✅
    ↓
Load Image Bytes → Convert to format LLM understands ✅
    ↓
LLM Context: [actual image pixels/encoding] ✅
    ↓
LLM Vision Model: "The car in the image is red" ✅
    ↓
User: 🎉
```

I needed to bridge the gap between **retrieval** (which CLIP handled perfectly) and **reasoning** (which required the LLM to actually *see* the images).

## The SDK Wall

Oracle Cloud has excellent vision-capable models:
- **Grok-4** from xAI (128K context, multimodal)
- **Meta Llama 3.2 90B Vision Instruct** (90B parameters, native vision)

Both accessible through OCI GenAI. Both worked perfectly in the **OCI Console Playground**.

But my app used the **TypeScript SDK** (v2.119.0). And that's where things fell apart.

### Attempt 1: The Obvious Approach

I tried passing images directly to the SDK's chat API:

```javascript
// backend/services/ociGenAIwithSDK.js
const oci = require('oci-generativeaiinference');

async function generateTextWithImages(prompt, images) {
  const chatDetails = {
    compartmentId: process.env.OCI_COMPARTMENT_ID,
    servingMode: {
      servingType: "ON_DEMAND",
      modelId: "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya..."
    },
    chatRequest: {
      messages: [
        {
          role: "USER",
          content: [
            { type: "TEXT", text: prompt },
            {
              type: "IMAGE",
              imageUrl: `data:image/jpeg;base64,${images[0].base64}`
            }
          ]
        }
      ]
    }
  };

  return await this.client.chat(chatDetails);
}
```

**Result:**
```
Error: Type 'IMAGE' is not assignable to type 'TextContent'
TypeScript SDK only supports text-only messages
```

The SDK's TypeScript types literally **rejected image content**. Text only.

### Attempt 2: The REST API Route

Fine. If the SDK won't cooperate, I'll use the REST API directly.

I spent an afternoon trying **five different request formats** based on the API documentation:

```javascript
// Format 1: Nested image_url (Python SDK style)
{
  "chatRequest": {
    "messages": [{
      "role": "USER",
      "content": [
        { "type": "TEXT", "text": "Describe this image" },
        {
          "type": "IMAGE",
          "image_url": {
            "url": "data:image/jpeg;base64,{base64_data}",
            "detail": "high"
          }
        }
      ]
    }]
  }
}
// Result: 400 - "Please pass in correct format of request"

// Format 2: Simple imageUrl
{
  "content": [
    { "type": "TEXT", "text": "Describe this image" },
    { "type": "IMAGE", "imageUrl": "data:image/jpeg;base64,{base64_data}" }
  ]
}
// Result: 400 - "Please pass in correct format of request"

// Format 3: Direct imageData
{
  "content": [
    { "type": "TEXT", "text": "Describe this image" },
    { "type": "IMAGE", "imageData": "{base64_data}" }
  ]
}
// Result: 400 - "Please pass in correct format of request"

// Format 4: No apiFormat field
// Result: 400 - "Please pass in correct format of request"

// Format 5: COHERE apiFormat instead of GENERIC
// Result: 400 - "Please pass in correct format of request"
```

**All five failed.** Same cryptic error. No hints about what was wrong.

The REST API documentation showed `ImageContent` as a supported type, but provided **zero examples** of the actual request structure for vision models.

## The Email That Changed Everything

Three days later, I was at a team meeting discussing a different project. During a break, I mentioned my vision API struggles to a colleague.

"Oh, you should talk to David Start (Director of Product Management at Oracle, and a collegue with whom I've always enjoyed working)," she said. "He's done a lot with OCI GenAI."

As it just so happened, Oracle AI World was right around the corner - over at the Oracle Database booth, I found David and explained the challenges I was having:
- Vision models work in Playground
- TypeScript SDK doesn't support vision
- REST API attempts all failing

**And it just so happened, that David had cracked this nut a little while back**

After we got back from AI World, he slacked me a bit of code ***and it was gold***

### The Python SDK Solution

David's slack was short and to the point:

He attached a Python script showing:
- How to structure the multimodal request using the **OCI Python SDK**
- Proper image encoding (base64 in data URI format)
- The exact message format that worked
- How to handle both text-only and vision requests

The key insight: **The Python SDK was ahead of the TypeScript SDK** in supporting vision features.

I had two options:
1. Wait for TypeScript SDK to catch up (could be weeks/months)
2. Build a **Python microservice** to bridge the gap

Given that I had working CLIP search, hundreds of images in the database, and I wanted to get this resolved... option 2 was obvious.

## 300 Lines to Vision-Aware RAG

I decided to build a **Flask microservice in Python** that would:
1. Accept requests from my Node.js backend
2. Use the OCI Python SDK to call Grok-4 Vision
3. Handle both text-only and multimodal requests
4. Return responses in a format my backend expected

**Time invested:** About 2 hours, thanks to David's example.

### The Python Vision Service

Here's the core of what I built:

```python
# backend/vision_service.py
from flask import Flask, request, jsonify
from flask_cors import CORS
import oci
import base64
import os

app = Flask(__name__)
CORS(app)

# Initialize OCI GenAI client
config = oci.config.from_file()
generative_ai_inference_client = oci.generative_ai_inference.GenerativeAiInferenceClient(
    config=config,
    service_endpoint=f"https://inference.generativeai.{config['region']}.oci.oraclecloud.com"
)

# Grok-4 model OCID (Chicago region)
GROK4_MODEL_ID = "ocid1.generativeaimodel.oc1.us-chicago-1.amaaaaaask7dceya3bsfz4ogiuv3yc7gcnlry7gi3zzx6tnikg6jltqszm2q"

@app.route('/analyze', methods=['POST'])
def analyze():
    """
    Generate text with optional images using Grok-4 Vision

    Request body:
    {
      "prompt": "Describe this image",
      "images": [
        {"base64": "...", "filename": "photo.jpg"}
      ],
      "temperature": 0.7,
      "maxTokens": 1500
    }
    """
    data = request.json
    prompt = data.get('prompt', '')
    images = data.get('images', [])
    temperature = data.get('temperature', 0.7)
    max_tokens = data.get('maxTokens', 1500)

    # Build message content
    content = [{"type": "TEXT", "text": prompt}]

    # Add images if present (multimodal mode)
    if images and len(images) > 0:
        for img in images[:5]:  # Limit to 5 images
            # This is the format that works with the Python SDK
            content.append({
                "type": "IMAGE",
                "image_url": {
                    "url": f"data:image/jpeg;base64,{img['base64']}"
                }
            })

    # Create chat request
    chat_request = oci.generative_ai_inference.models.GenericChatRequest(
        messages=[
            oci.generative_ai_inference.models.Message(
                role="USER",
                content=content
            )
        ],
        max_tokens=max_tokens,
        temperature=temperature,
        top_p=0.95
    )

    # Send to Grok-4 Vision
    chat_detail = oci.generative_ai_inference.models.ChatDetails(
        compartment_id=os.environ.get('OCI_COMPARTMENT_ID'),
        serving_mode=oci.generative_ai_inference.models.OnDemandServingMode(
            model_id=GROK4_MODEL_ID
        ),
        chat_request=chat_request
    )

    response = generative_ai_inference_client.chat(chat_detail)

    # Extract response text
    response_text = response.data.chat_response.choices[0].message.content[0].text

    return jsonify({
        'response': response_text,
        'model': 'grok-4',
        'multimodal': len(images) > 0
    })

@app.route('/health', methods=['GET'])
def health():
    return jsonify({'status': 'healthy', 'service': 'vision-llm'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=3002)
```

**That's it.** 150 lines of actual logic (the rest is imports, error handling, and comments).

### The Node.js Integration

Now my Node.js backend could call this Python service:

```javascript
// backend/services/ociGenAIwithSDK.js
const axios = require('axios');

class OCIGenAIService {
  async generateText(prompt, options = {}) {
    const { images = [], temperature = 0.7, maxTokens = 1500 } = options;

    // Call Python vision service
    const response = await axios.post('http://localhost:3002/analyze', {
      prompt,
      images,
      temperature,
      maxTokens
    });

    return response.data.response;
  }
}
```

### The Complete Q&A Flow

Here's what happens when a user asks about an image:

```javascript
// backend/routes/notes.js - Q&A endpoint
async function handleQuestion(question, llmProvider) {
  // 1. CLIP search for relevant images
  const imageResults = await executeQuery(`
    SELECT d.file_content, d.document_name,
           VECTOR_DISTANCE(
             dc.chunk_vector_clip,
             VECTOR_EMBEDDING(CLIP_TEXT USING :question as INPUT),
             COSINE
           ) as similarity
    FROM document_chunks dc
    JOIN documents d ON dc.document_id = d.document_id
    WHERE d.document_type = 'IMAGE'
    ORDER BY similarity
    FETCH FIRST 5 ROWS ONLY
  `, { question });

  // 2. For vision provider: convert images to base64
  let imageData = [];
  if (llmProvider === 'vision' && imageResults.rows.length > 0) {
    imageData = imageResults.rows
      .filter(row => row.SIMILARITY < 0.85)  // Only highly relevant images
      .slice(0, 3)  // Max 3 images
      .map(row => ({
        base64: row.FILE_CONTENT.toString('base64'),
        filename: row.DOCUMENT_NAME
      }));
  }

  // 3. Text search for context (same as before)
  const textContext = await getRelevantTextContext(question);

  // 4. Build prompt with context
  const prompt = `
Based on the following context, answer the question.

CONTEXT:
${textContext}

QUESTION: ${question}
`;

  // 5. Call LLM with images (if vision provider)
  const response = await llmRouter.generateText(prompt, {
    images: imageData,  // Empty array for non-vision providers
    temperature: 0.3,
    maxTokens: 1500
  }, llmProvider);

  return {
    answer: response,
    relatedImages: imageResults.rows.map(r => ({
      filename: r.DOCUMENT_NAME,
      similarity: r.SIMILARITY
    }))
  };
}
```

## The Architecture: Before and After

### Before (Retrieval Only)
```
User: "What color is the car?"
    ↓
Node.js Backend
    ↓
CLIP Vector Search (Oracle AI Database 26ai)
    → Finds: car_photo.jpg (similarity: 0.92)
    ↓
LLM (Cohere Command-R)
    Context: "Found document: car_photo.jpg"
    ↓
Response: "I don't have information about vehicle colors"
```

### After (Vision-Aware RAG)
```
User: "What color is the car?"
    ↓
Node.js Backend
    ↓
CLIP Vector Search (Oracle AI Database 26ai)
    → Finds: car_photo.jpg (similarity: 0.92)
    ↓
Fetch Image BLOB from database
    ↓
Convert to base64
    ↓
Python Vision Service (Flask on port 3002)
    ↓
OCI Python SDK
    ↓
Grok-4 Vision (128K context, multimodal)
    Context: [actual image pixels as base64]
    ↓
Response: "The car in the image is red"
```

## The Results: Vision That Actually Works

### Response Time
```
Text-only query:
- CLIP search: 45ms
- Text context: 120ms
- Grok-4: 2.1s
- Total: ~2.3s

Vision query (with 1 image):
- CLIP search: 45ms
- Image fetch: 85ms
- Base64 encode: 15ms
- Text context: 120ms
- Python service → Grok-4: 4.2s
- Total: ~4.5s

Vision query (with 3 images):
- CLIP search: 45ms
- Image fetch: 210ms
- Base64 encode: 35ms
- Text context: 120ms
- Python service → Grok-4: 5.8s
- Total: ~6.2s
```

Not instant, but **totally acceptable** for the value delivered.

### Accuracy Comparison

| Question Type | Before (Text-Only) | After (Vision-Aware) |
|---------------|-------------------|---------------------|
| **"What color is the car?"** | ❌ "No information" | ✅ "Red sedan" |
| **"What text is on the sign?"** | ❌ "Cannot determine" | ✅ Accurate OCR |
| **"How many people in photo?"** | ❌ Guesses from filename | ✅ Counts accurately |
| **"What brand is the laptop?"** | ❌ No answer | ✅ Identifies logo |
| **"Describe the scene"** | ❌ Generic response | ✅ Detailed description |

### Real Usage Example

Here's an actual exchange from my testing:

**Q:** "Explain this image for me"

**Before (Text-only):**
> "I don't have access to the specific content in the image. The available documents include references to images, but I cannot see the visual content."

**After (Vision-aware):**
> "The image shows:
> - 'Oil and gas pipelines' as the header
> - Two bullet points:
>   • Oil pipelines are in red
>   • Gas pipelines are in green
> - A map of the United States showing the pipelines"

**Accuracy:** I checked the actual image. Grok-4 got every detail correct.

That's the difference between **retrieval** and **reasoning**.

## The Bonus: Hybrid Vision Approach

While implementing the Grok-4 solution, I worked with **OCI Vision AI**—a separate service specialized for image analysis.

Instead of sending raw pixels to an LLM, OCI Vision provides:
- **Object Detection**: "person, vehicle, building"
- **Scene Classification**: "outdoor, daytime, parking lot"
- **Text Extraction (OCR)**: Reads actual text in images
- **Confidence Scores**: How certain is each detection

### Why This Matters

For some queries, **structured analysis beats raw pixels**:

```javascript
// OCI Vision output for a photo
{
  "sceneClassifications": [
    { "label": "Person", "confidence": 0.89 },
    { "label": "Vehicle", "confidence": 0.85 },
    { "label": "Outdoor", "confidence": 0.92 }
  ],
  "objectDetections": [
    { "label": "person", "confidence": 0.91, "boundingBox": {...} },
    { "label": "car", "confidence": 0.87, "boundingBox": {...} },
    { "label": "shirt", "confidence": 0.73, "boundingBox": {...} }
  ],
  "textDetections": [
    { "text": "Daytona 500", "confidence": 0.96 }
  ]
}
```

I can then add this structured data to the context:

```
=== VISUAL ANALYSIS ===
IMAGE: meeting_photo.jpg
SCENE: Outdoor (92%), Person (89%), Vehicle (85%)
OBJECTS DETECTED: person, car, shirt
TEXT IN IMAGE: "Daytona 500"
=== END VISUAL ANALYSIS ===

QUESTION: What event is this photo from?
```

Even a **text-only LLM** can now reason about image content using the structured analysis.

### The Hybrid Architecture

I implemented **both approaches** and let the system choose:

```javascript
// For simple queries → OCI Vision (faster)
if (questionType === 'simple-visual') {
  analysis = await ociVisionAnalyzer.analyzeImage(imageBlob);
  context += formatVisionAnalysis(analysis);
  llmResponse = await grok4.generateText(context);  // Text mode
}

// For complex queries → Grok-4 Vision (deeper reasoning)
if (questionType === 'complex-visual') {
  llmResponse = await grok4Vision.generateText(prompt, {
    images: [imageBase64]  // Vision mode
  });
}
```

**Best of both worlds.**

## Lessons Learned

| Insight | Takeaway |
|---------|----------|
| **SDKs Lag Behind Services** | The OCI Console supported vision weeks before the TypeScript SDK. If you hit SDK limitations, check if other SDKs (Python, Java, Go) are ahead. Don't assume they're all in sync. |
| **Polyglot > Monolith for AI** | Mixing Python (OCI SDK, ML libraries) and Node.js (API, business logic) proved **better** than forcing everything into one language. Use the right tool for each job. |
| **Microservices Unlock Innovation** | The 300-line Python service took 2 hours to build and unblocked weeks of potential waiting. Small, focused services > large refactors. |
| **Community > Documentation** | David Start's email was more valuable than hours of API docs. **Reach out to people.** The Oracle community is helpful. |
| **Specialized > General** | OCI Vision (specialized) sometimes beats Grok-4 (general) for structured tasks like OCR. Don't assume the biggest model is always best. |

## The Code: Open for Adaptation

The Python vision service is **~300 lines total**. Here's the startup configuration I added to `package.json`:

```json
{
  "scripts": {
    "start": "node server-prod.js",
    "start:vision": "python3 backend/vision_service.py",
    "start:all": "concurrently \"npm start\" \"npm run start:vision\""
  },
  "devDependencies": {
    "concurrently": "^9.2.1"
  }
}
```

One command starts both services:
```bash
npm run start:all
```

The architecture is **simple by design**:
- Node.js handles business logic, database, and API
- Python handles OCI Vision SDK calls
- HTTP bridge between them (could be gRPC for production)

No complex orchestration. No containers (yet). Just two processes talking over localhost.

**And it works.**

## What's Next

The vision-aware RAG is now **production-ready** in my knowledge management app:
- Users can ask questions about image content
- Grok-4 Vision sees and reasons about actual pixels
- OCI Vision provides structured analysis when appropriate
- Response times are acceptable (3-6 seconds with images)

### Future Enhancements

1. **Caching:** Vision analysis could be cached per image to avoid re-analyzing
2. **Batch Processing:** Analyze multiple images in parallel
3. **Video Support:** Extract frames and analyze video content
4. **SDK Update:** When TypeScript SDK adds vision, remove the Python bridge (or keep it—it works great)

### Blog Series Wrap-Up

This three-part series covered:

**Part 1:** [Three LLMs, One App](https://brianhengen.us/posts/three-llms-one-app/) - Multi-provider LLM architecture
**Part 2:** [CLIP Inside Oracle AI Database 26ai](https://brianhengen.us/posts/clip-inside-oracle-ai-database-26ai-fast-multimodal-rag/) - In-database embeddings for 10x faster search
**Part 3:** This post - Vision-aware RAG with a Python bridge

Together, they show a complete RAG evolution:
- **Text search** → **Multimodal search** → **Vision-aware reasoning**

Each step unlocked new capabilities. Each required different solutions. Each taught valuable lessons.

## Special Thanks

This breakthrough wouldn't have happened without **David Start**, Oracle Director of Database Product Management (https://www.linkedin.com/in/davidastart/).

David's Python SDK example and willingness to help a colleague solve a tricky problem turned **days of frustration** into **hours of productive work**.

In the AI world, we often focus on models, frameworks, and infrastructure. But sometimes the biggest unlock is **help from someone who's been there before**.

Thank you, David.

## The Bottom Line

**The Problem:** Vision models existed but were inaccessible via TypeScript SDK
**The Blocker:** REST API attempts failed, across multiple attempts
**The Breakthrough:** Helpful Python SDK guidance
**The Solution:** 300-line Flask microservice bridging Node.js and OCI Python SDK
**Time to Production:** 1 evening
**Result:** Vision-aware RAG that actually works

---

**Have you built polyglot microservices for AI workloads?**
What challenges did you face? [Drop a comment or reach out](https://brianhengen.us/)

**What's next** (outside this series): an AI-powered robot car that navigates my house…

## About the Author
Brian Hengen is a Vice President at Oracle, leading technical sales engineering teams. The views and opinions expressed in this blog are his own and do not necessarily reflect those of Oracle.
