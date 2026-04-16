# RAG with Cloudflare Vectorize (Future)

> **Status**: Deferred — not in MVP. Document here for future implementation.

## Problem

A generic LLM knows nothing about internal company procedures, deployment workflows, or proprietary tools. The system prompt alone can only carry so much context. For high-quality intern guidance, the LLM needs access to internal documentation.

## Solution: Retrieval-Augmented Generation at the Edge

Expand the Cloudflare Worker proxy into a RAG pipeline using Workers AI and Vectorize.

## Architecture

### 1. Document Ingestion Pipeline (One-Time Setup)

```
Internal Wiki / Deployment Guides / Runbooks
    ↓ Split into chunks (~500 tokens each)
    ↓ Embed via Workers AI (@cf/baai/bge-base-en-v1.5, 768 dimensions)
    ↓ Upsert into Vectorize index with metadata tags
```

### 2. Query-Time Retrieval (Per Request)

When the intern's hotkey or idle trigger fires:

```typescript
// 1. Embed the intern's question
const queryVector = await env.AI.run("@cf/baai/bge-base-en-v1.5", {
  text: internTranscript
});

// 2. Search with RBAC metadata filtering
const results = await env.VECTORIZE.query(queryVector.data, {
  topK: 5,
  filter: {
    "audience": { "$eq": "intern" },
    "department": { "$eq": "engineering" }
  },
  returnMetadata: "all"
});

// 3. Inject retrieved context into Claude's system prompt
const augmentedPrompt = `${baseSystemPrompt}\n\nRelevant internal docs:\n${results.matches.map(m => m.metadata.text).join("\n---\n")}`;
```

### 3. Send Augmented Request to Claude

The retrieved internal documentation is prepended to the system prompt, giving Claude specific knowledge about company workflows without fine-tuning.

## Multi-Tenancy and RBAC

Tag every document with access metadata during ingestion:

```json
{
  "audience": "intern",
  "department": "engineering",
  "clearance": 1,
  "tool": "deploy-dashboard"
}
```

Filter at query time so interns never retrieve documents above their clearance level. This prevents prompt injection attacks from tricking the LLM into leaking compartmentalized data.

## Alternative: Cloudflare AI Search

Cloudflare offers AI Search as a higher-level product that handles chunking, embedding, indexing, and filtering out of the box. Simpler to deploy but less granular control for custom scoring algorithms. Evaluate based on compliance requirements.

## Worker Route Change

The existing `POST /chat` route would be modified to:
1. Accept the transcript + screenshot
2. Run vector similarity search
3. Inject retrieved context into the system prompt
4. Forward the augmented request to Anthropic
5. Stream the SSE response back to the macOS client

The macOS app itself needs no changes — it still sends the same payload to `/chat`.
