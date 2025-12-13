---
layout: post
title:  "Building an Intelligent PDF Invoice Processor with Vector Search and LLM Verification"
date:   2025-11-27 12:00:00 +0800
categories: llm mongodb streamlit vector-search python
---

I built a [Streamlit](https://streamlit.io/) application that processes PDF invoices and receipts, automatically classifies merchants using vector similarity search, and provides natural language querying capabilities. The project demonstrates how to combine modern embedding models, vector databases, and LLMs to create an intelligent document processing system.

**Update (Dec 2025):** The project has been modernized to use Claude's Vision API for direct PDF analysis and Structured Outputs for guaranteed valid JSON responses. This removes the need for intermediate text extraction and significantly improves handling of complex document layouts.

## The Problem

Anyone who's tried to organize their receipts knows the pain: the same merchant appears under dozens of different names. "Grab Singapore Pte Ltd" on one receipt, "Grab SG" on another, and "GRAB*GRABPAY" on yet another. Traditional exact-match systems fail miserably here, and manually normalizing merchant names doesn't scale.

## The Solution

The system uses a three-stage approach to merchant classification:

1. **Exact synonym matching** - Fast lookup for known variations
2. **Vector similarity search** - Find semantically similar merchant names
3. **LLM verification** - Claude Sonnet 4.5 confirms uncertain matches

### PDF Processing with Claude Vision

Instead of extracting text from PDFs and then analyzing it, the system now sends PDFs directly to Claude's Vision API as base64-encoded images. This approach has several advantages:

- **Preserves layout context** - Tables, columns, and spatial relationships are understood
- **Handles complex formats** - Multi-column receipts, handwritten notes, stamps
- **No OCR dependencies** - Removes the need for PyMuPDF or similar libraries
- **Better accuracy** - Claude can see the document exactly as a human would

### Structured Outputs for Reliable Data Extraction

The project uses Claude's [tool use feature](https://docs.anthropic.com/en/docs/build-with-claude/tool-use) to guarantee valid JSON responses. Instead of hoping the LLM returns well-formed JSON, we define a schema and Claude is constrained to return data matching that schema:

```python
tools = [{
    "name": "extract_invoice_data",
    "description": "Extract structured data from invoice/receipt",
    "input_schema": {
        "type": "object",
        "properties": {
            "merchant_name": {"type": "string"},
            "date": {"type": "string", "format": "date"},
            "total_amount": {"type": "number"},
            "currency": {"type": "string"},
            "items": {"type": "array", "items": {...}}
        },
        "required": ["merchant_name", "total_amount"]
    }
}]
```

This eliminates JSON parsing errors and retry logic that plagues many LLM applications.

### Why Vector Similarity Works

The key insight is using [SentenceTransformers](https://www.sbert.net/) with the `paraphrase-multilingual-mpnet-base-v2` model. This model was specifically trained on 50+ languages and optimized for paraphrase detection, making it ideal for recognizing that "Hong Kong Restaurant" and "香港餐厅" refer to the same entity.

The model produces 768-dimensional embeddings that capture semantic meaning. When a new merchant name comes in, we:

1. Generate its embedding
2. Search [MongoDB Atlas Vector Search](https://www.mongodb.com/atlas/search) with a 0.85 cosine similarity threshold
3. If above threshold, automatically link to existing merchant
4. If borderline, ask Claude to verify

### Technical Architecture

```
PDF Upload → Claude Vision API → Structured Outputs → Merchant Classification → Storage
                                                               ↓
                                         Exact Match → Vector Search → LLM Verification
```

The classification flow prioritizes speed for obvious matches while falling back to LLM verification only when necessary. This keeps costs low (roughly $0.02-0.10 per document depending on PDF complexity) while maintaining high accuracy.

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

1. **Vision APIs beat text extraction** - Sending documents directly to Claude Vision preserves layout context that gets lost in OCR pipelines
2. **Structured Outputs eliminate parsing headaches** - Using tool use for guaranteed JSON schema compliance is far more reliable than prompt engineering
3. **Vector similarity thresholds matter** - 0.85 was the sweet spot between false positives and misses
4. **LLM verification is expensive** - Use it as a fallback, not a first pass
5. **Synonyms compound over time** - Each processed document improves future classification
6. **Multilingual models are underrated** - The cross-lingual transfer capabilities are impressive

The combination of Claude Vision for document understanding, structured outputs for reliable extraction, and vector search with LLM verification for classification creates a system that improves with use while keeping operational costs reasonable.
