---
title: "Fine-Tuning a Personal Executive Assistant: Lessons from My Management Notes"
date: 2025-10-05
draft: false
tags: ["AI", "SLM", "Fine-tuning", "Granite", "Management AI", "Executive Assistant"]
categories: ["AI Training"]
description: "How I fine-tuned IBM Granite 4.0 H Micro on 5+ years of management notes to create a personalized AI assistant—after three failures, the key was data quality."
---

After successfully fine-tuning ChefBot on cooking recipes, I wondered: **Could I fine-tune an AI on my own management experience to create a personalized executive assistant?**

Imagine asking your AI: "Summarize last week's 1:1 with Sarah and suggest coaching points", and getting a response in your voice, drawing from years of team dynamics and decision patterns. All running locally, with complete privacy and no API costs.

This is the story of how I built exactly that. Spoiler: I failed three times before succeeding, and the lesson wasn't about hyperparameters.

## Why Build This?

As a technology executive, I generate mountains of notes—hundreds of pages across:
- Meetings with team members
- Leadership team strategy discussions
- Deal reviews and account planning
- Performance coaching sessions
- Risk assessments and escalations

General-purpose LLMs like ChatGPT can help with generic management advice, but they don't know:
- My specific management philosophy and approach
- My team's unique dynamics and challenges
- My company's internal processes and terminology
- The context from years of accumulated decisions

**The goal:** Train a Specialized Language Model (SLM) that acts like my personal management consultant, tuned on the last 3 years of my personal notes.

**Bonus benefits:**
- ✅ Privacy: All data stays on-premises (critical for sensitive data)
- ✅ No API costs: One-time training vs. ongoing ChatGPT fees
- ✅ Customization: Full control over behavior and outputs
- ✅ Learning: Hands-on experience with real-world fine-tuning challenges

## The Hardware: RTX 5090 Strikes Again

My RTX 5090 workstation proved perfect for this project:
- **32GB VRAM**: Enough for 3B parameter models with QLoRA
- **AMD Ryzen 9 7900X (64GB RAM)**: Fast data preprocessing
- **Local training**: No cloud dependencies or data transfer

For 3B parameter models, the 5090 is actually overkill—but that headroom means I can experiment with larger models later.

## Choosing the Right Model

After researching options, I chose **[IBM Granite 4.0 H Micro](https://huggingface.co/ibm-granite/granite-4.0-h-micro)** over the more common Llama 3.2 3B:

Here's a quick comparison:

```chartjs
{
  "type": "bar",
  "data": {
    "labels": ["Parameters", "Context Window", "HumanEval"],
    "datasets": [
      {
        "label": "Granite 4.0 H Micro",
        "data": [3, 128, 81],
        "backgroundColor": "#4e79a7"
      },
      {
        "label": "Llama 3.2 3B",
        "data": [3, 8, 75],
        "backgroundColor": "#f28e2b"
      }
    ]
  },
  "options": {
    "scales": {
      "y": {
        "beginAtZero": true
      }
    },
    "plugins": {
      "legend": {
        "position": "top"
      },
      "title": {
        "display": true,
        "text": "Granite 4.0 H Micro vs Llama 3.2 3B"
      }
    }
  }
}
```

| Feature | Granite 4.0 H Micro | Llama 3.2 3B |
|---------|---------------------|--------------|
| Parameters | 3B | 3B |
| Context Window | **128K tokens** | 4K-8K tokens |
| Architecture | Hybrid (4 attention + 36 Mamba2 layers) | Standard transformer |
| License | Apache 2.0 | Llama 3 Community |
| Enterprise Focus | ✅ Built for business | ❌ General purpose |
| Deployment | Optimized for MacBook Pro | Standard |

**Why Granite won:**
- **128K context window** is crucial for long management notes and meeting transcripts
- **Hybrid Mamba2 architecture** is more efficient than pure transformers for long context
- **Enterprise-focused** design aligns with my use case
- **Tool-calling capabilities** for future RAG integration
- Only 3B parameters but matches/beats larger models on benchmarks (MMLU: 67.4, HumanEval: 81)

## The Data Disaster: Three Failed Attempts

### Attempt 1: The Overfit Catastrophe
**Setup:** 3 epochs, 25,516 training examples, LoRA rank 64

**Result:** COMPLETE FAILURE
- Training loss: 2.46 → **0.0006** (99.97% reduction!)
- Eval loss: **nan** (can't generalize at all)
- Duration: ~7 hours

The model memorized everything—regurgitating training data verbatim. Worse than the base model.

### Attempt 2: Too Conservative
**Setup:** 1 epoch, reduced learning rate, higher regularization

**Result:** Better, but too generic
- No overfitting (good!)
- But didn't learn my management style (bad!)
- Too cautious—threw out the baby with the bathwater

### Attempt 3: Still Broken
**Setup:** 2 epochs (compromise between V1 and V2)

**Result:** Still overfitting
- Training loss: 0.02-0.03 (too low)
- Eval loss: nan again
- Model output correct JSON but filled with gibberish

At this point, I realized: **You can't fix bad data with better hyperparameters.**

## The Real Problem: Garbage Data

I examined the training data more closely:

**Bad Example 1:**
```json
{
  "question": "What is the main insight?",
  "answer": "{\"summary\":\"FY21 - Leadership Team.pdf\",\"bullets\":[]}"
}
```

**Bad Example 2:**
```json
{
  "question": "How can this be implemented?",
  "answer": "{\"items\":[{\"text\":\"Bob got engaged yesterday!\"}]}"
}
```

The data was auto-generated by weak AI and never validated:
- Empty summaries (just filenames)
- Random text fragments as "action items"
- Generic template questions with zero variation
- No actual insight extraction

**The model learned perfectly—it's just that it was learning garbage.** Pro tip: Always sample and validate your dataset manually before training.

## The Solution: Regenerate with Quality

I went back to the source: **96 PDF files of real management notes**:
- 201MB of raw notes
- Extracted into 1,623 text chunks
- Covering 1:1 meetings, leadership discussions, deal reviews, coaching sessions

**New strategy:** Use my company's coding assistant to generate quality training examples:
1. Feed each chunk to the assistant
2. Ask it to generate 2-3 natural, conversational questions with real insights
3. Extract summaries, action items, risks, and coaching advice
4. Focus on quality over quantity

**Output:** 2,946 high-quality examples (vs 25,516 garbage ones)

**Quality comparison:**

**OLD (garbage):**
```json
{
  "question": "What is the main insight?",
  "answer": "{\"summary\":\"FY21 - Leadership Team.pdf\",\"bullets\":[]}"
}
```

**NEW (quality):**
```json
{
  "question": "How can a leader help maintain productive tension during organizational change?",
  "answer": "A leader must: 1) Create a 'holding environment' where diverse groups can discuss challenges, 2) Sequence and pace the work so people don't feel overwhelmed, 3) Regulate distress and maintain emotional capacity to tolerate uncertainty."
}
```

The difference is night and day. Pro tip: Use a strong API for data augmentation - it makes a big difference

## Attempt 4: Success at Last

**The Setup:**
- **Model:** IBM Granite 4.0 H Micro (3B params)
- **Data:** 2,651 train / 295 validation (90/10 split)
- **Training:** 2 epochs, QLoRA (4-bit quantization), LoRA rank 32
- **Duration:** 1 hour 25 minutes

**The Results:**
- ✅ Training loss: **2.403**
- ✅ Eval loss: **2.299** (healthy gap—no overfitting!)
- ✅ 664 total steps (vs 5,742 with bloated dataset)
- ✅ **10x faster** than previous attempts
- ✅ Model actually works!

## The Proof: Real-World Testing

I tested the tuned model against the base Granite model with real management queries:

### Email Summarization
- **Tuned:** 40% more concise, captures key points in executive-friendly format
- **Base:** Comprehensive but verbose, too much detail

### Technical Questions
- **Tuned:** 8 actionable bullet points, executive-ready
- **Base:** 11+ paragraphs of academic detail

### Context Awareness
- **Tuned:** Recognizes specific details from my training data
- **Base:** Generic responses without organizational context

### Management Advice
- **Tuned:** Concise next steps with clear priorities
- **Base:** 4-section implementation plan (too detailed for quick decisions)

**The verdict:** The fine-tuned model is **significantly more useful** for daily management tasks, mirroring my voice and priorities.

## Key Lessons Learned

### 1. Data Quality Trumps Everything
**You cannot fix bad data with hyperparameter tuning.** The model will learn perfectly—it's just that it will learn the wrong patterns.

- ❌ 25,516 garbage examples = failure
- ✅ 2,946 quality examples = success

Spend time on data quality upfront. It's worth it to generate quality training data rather than weak models that create garbage.

### 2. The Sweet Spot: 2 Epochs with Quality Data
- **1 epoch:** Underfit (too generic)
- **2 epochs:** Just right ✅
- **3 epochs:** Overfitting risk (even with good data)

With quality data, you need fewer epochs to achieve good results.

### 3. Smaller, Focused Datasets Beat Large Garbage Datasets
- 2,651 quality examples >>> 25,516 garbage examples
- Faster training (1.5 hours vs 7+ hours)
- Better results
- Less risk of overfitting

**Quality over quantity** is not just a slogan—it's the difference between success and failure.

### 4. Model Architecture Matters
Granite 4.0 H Micro's hybrid Mamba2 architecture offers real advantages:
- More efficient than standard transformers
- Better long-context handling (crucial for meeting notes)
- Optimized for edge deployment (runs great on MacBook Pro)
- Only 3B params but excellent performance

Don't just default to Llama models—explore alternatives that might fit your use case better.

### 5. Watch the Eval Loss Gap
- Train: 2.40, Eval: 2.30 → **Healthy** ✅
- Train: 0.0006, Eval: nan → **Disaster** ❌

A healthy gap between training and eval loss indicates proper generalization. If training loss goes near zero while eval loss explodes, you're overfitting.

## Production Deployment

**Model Status:** ✅ READY FOR PRODUCTION

**Deployment specs:**
- LoRA adapters: Only 6.6MB (easy to distribute)
- Memory footprint: ~2GB with 4-bit quantization
- Inference speed: ~7 seconds per response
- Runs on MacBook Pro (no GPU required for inference)

**Planned architecture:**
1. Fine-tuned model for management style and decision-making
2. RAG (Retrieval-Augmented Generation) for specific knowledge
3. This keeps the management voice pure while adding factual product info

Why RAG instead of more fine-tuning?
- Product knowledge changes frequently (RAG is easier to update)
- Prevents diluting the management style learned in V4
- Best of both worlds: personal style + current facts


**ROI:**
- Custom AI assistant tuned to 3+ years of management experience
- Runs locally (privacy guaranteed)
- No ongoing API costs
- Ready to deploy on laptop


## What's Next?

**Immediate:**
- Test against daily management tasks
- Gather feedback on gaps in knowledge
- Build RAG system for ongoing practices

Got questions about fine-tuning your own domain-specific AI? Drop a comment or reach out on GitHub!

## The Big Takeaway

After four attempts and multiple failures, here's what I learned:

**Building specialized AI isn't about having the biggest model or the most complex training setup.**

It's about:
1. **Quality data** (spend money/time here)
2. **Right-sized model** (3B can beat 70B with good data)
3. **Clear evaluation metrics** (watch that eval loss!)
4. **Patience to iterate** (fail fast, learn, improve)

The RTX 5090 made experimentation easy—I could try different approaches in hours rather than days. But the real breakthrough came from recognizing that **garbage data can't be fixed with engineering tricks**.

If you're considering fine-tuning an LLM for your domain, invest in data quality first. Everything else follows from that foundation.

---

**Project Stats:**
- **Started:** October 3, 2025
- **Production Ready:** October 4, 2025 (2 days!)
- **Training time:** 1 hour 25 minutes (V4)
- **Data:** 2,946 examples from 5+ years of notes
- **Cost:** <$20 total
- **Model:** IBM Granite 4.0 H Micro (3B params)

Want to try this yourself? The key components:
- RTX GPU (3090/4090/5090) or cloud compute
- Quality source data (your domain expertise)
- QLoRA fine-tuning (Hugging Face PEFT)
- Patience to iterate on data quality