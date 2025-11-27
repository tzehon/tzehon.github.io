---
layout: post
title:  "Building an Intelligent PDF Invoice Processor with Vector Search and LLM Verification"
date:   2025-11-27 12:00:00 +0800
categories: llm mongodb streamlit vector-search python
---

I built a [Streamlit](https://streamlit.io/) application that processes PDF invoices and receipts, automatically classifies merchants using vector similarity search, and provides natural language querying capabilities. The project demonstrates how to combine modern embedding models, vector databases, and LLMs to create an intelligent document processing system.

## The Problem

Anyone who's tried to organize their receipts knows the pain: the same merchant appears under dozens of different names. "Grab Singapore Pte Ltd" on one receipt, "Grab SG" on another, and "GRAB*GRABPAY" on yet another. Traditional exact-match systems fail miserably here, and manually normalizing merchant names doesn't scale.

## The Solution

The system uses a three-stage approach to merchant classification:

1. **Exact synonym matching** - Fast lookup for known variations
2. **Vector similarity search** - Find semantically similar merchant names
3. **LLM verification** - Claude Sonnet 4.5 confirms uncertain matches

### Why Vector Similarity Works

The key insight is using [SentenceTransformers](https://www.sbert.net/) with the `paraphrase-multilingual-mpnet-base-v2` model. This model was specifically trained on 50+ languages and optimized for paraphrase detection, making it ideal for recognizing that "Hong Kong Restaurant" and "香港餐厅" refer to the same entity.

The model produces 768-dimensional embeddings that capture semantic meaning. When a new merchant name comes in, we:

1. Generate its embedding
2. Search [MongoDB Atlas Vector Search](https://www.mongodb.com/atlas/search) with a 0.85 cosine similarity threshold
3. If above threshold, automatically link to existing merchant
4. If borderline, ask Claude to verify

### Technical Architecture

```
PDF Upload → Text Extraction → Metadata Extraction (Claude) → Merchant Classification → Storage
                                                                      ↓
                                              Exact Match → Vector Search → LLM Verification
```

The classification flow prioritizes speed for obvious matches while falling back to LLM verification only when necessary. This keeps costs low (roughly $0.01-0.05 per document) while maintaining high accuracy.

### Natural Language Querying

Instead of requiring users to write MongoDB queries, the system converts natural language to aggregation pipelines:

- "Show me all Grab receipts from last month" → `$lookup` + `$match` on date range
- "Calculate total spending by merchant" → `$group` + `$sort`
- "Find transactions over $100" → `$match` on amount

Claude generates the appropriate pipeline structure, handling the translation between human intent and database operations.

## Multilingual Support

This is where the project gets interesting. The `paraphrase-multilingual-mpnet-base-v2` model enables cross-lingual semantic matching. It can recognize that merchant names in different languages or scripts might refer to the same business, even without explicit training on those specific pairs.

In practice, this means receipts from different countries or in different languages can be automatically grouped under the correct merchant.

## Running It Yourself

The entire system runs on:
- MongoDB Atlas free tier (M0)
- Anthropic API (pay-per-use)
- Local Python environment

Setup involves configuring MongoDB Atlas with vector search enabled and providing your Anthropic API key. The model downloads automatically on first run (~420MB).

The code is available at [research/invoice_processor](https://github.com/tzehon/research/tree/main/invoice_processor).

## Key Learnings

1. **Vector similarity thresholds matter** - 0.85 was the sweet spot between false positives and misses
2. **LLM verification is expensive** - Use it as a fallback, not a first pass
3. **Synonyms compound over time** - Each processed document improves future classification
4. **Multilingual models are underrated** - The cross-lingual transfer capabilities are impressive

The combination of fast vector search with slower but more accurate LLM verification creates a system that improves with use while keeping operational costs reasonable.
