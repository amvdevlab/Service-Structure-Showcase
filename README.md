# Service Structure Showcase
> **Architecture:** RAG + Template-Mode Assistant

An internal knowledge assistant that turns PDF uploads into a grounded, conversational workspace — and can output **strictly-validated structured documents** (templates) from the same conversation.

Built as a single Next.js 16 application that ships to the edge on Cloudflare Workers, with a full retrieval-augmented generation (RAG) pipeline behind it.

> This repository is a **showcase**. It contains no application source, secrets, or customer data — only the architecture, decisions, and representative snippets that illustrate how it was built.

---

## Demo media

**Video (walkthrough)** — *Coming Soon*

[![Demo walkthrough — click to play](./media/PLACEHOLDER-video-thumbnail.png)](https://REPLACE_ME_WITH_VIDEO_URL)


**Screenshots** — *Coming Soon*

| 1 · Chat & citations | 2 · Documents / upload | 3 · Template or export |
|:---:|:---:|:---:|
| ![Chat and grounded answer with sources](./media/PLACEHOLDER-screenshot-01.png) | ![Document library or upload flow](./media/PLACEHOLDER-screenshot-02.png) | ![Template draft, structured fields, or PDF export](./media/PLACEHOLDER-screenshot-03.png) |



---

## Table of contents

- [What it does](#what-it-does)
- [System overview](#system-overview)
- [Stack & why](#stack--why)
- [Key flows](#key-flows)
  - [1. PDF ingestion](#1-pdf-ingestion)
  - [2. Retrieval](#2-retrieval)
  - [3. Grounded chat](#3-grounded-chat)
  - [4. Template-mode chat](#4-template-mode-chat)
- [Auth & access control](#auth--access-control)
- [Edge runtime decisions](#edge-runtime-decisions)
- [Project shape](#project-shape)
- [Notable engineering decisions](#notable-engineering-decisions)
- [What I'd improve next](#what-id-improve-next)

---

## What it does

- **Ingest PDFs** uploaded by signed-in users, extract text, chunk it, embed it, and index it in a vector store.
- **Answer questions** grounded in that corpus, with chunk-level citations back to the source document.
- **Fill structured templates** from conversation + retrieved context. The model returns strict JSON that is validated against a schema derived from a template definition — so "fill this form from the transcript" is a type-safe operation, not a prompt-only hope.
- **Iterate on drafts**: users can ask to change fields and the model emits an updated instance that re-validates before hitting the UI.

Everything runs behind Google OAuth with a server-side allowlist, and is deployed to Cloudflare Workers.

---

## System overview

```text
                        ┌──────────────────────────────┐
                        │   Next.js 16 (App Router)    │
                        │   React 19 + Tailwind v4     │
                        │   shadcn/ui + Base UI        │
                        └──────────────┬───────────────┘
                                       │  Server Actions / Route Handlers
                                       ▼
        ┌──────────────────────────────────────────────────────────┐
        │              @opennextjs/cloudflare Worker               │
        │  (nodejs_compat, global_fetch_strictly_public)           │
        └───┬─────────────┬─────────────┬──────────────┬───────────┘
            │             │             │              │
     ┌──────▼─────┐ ┌─────▼──────┐ ┌────▼──────┐ ┌─────▼────────┐
     │ Cloudflare │ │  Pinecone  │ │  OpenAI   │ │  Anthropic   │
     │    R2      │ │  (vectors) │ │(embeddings│ │  (Claude)    │
     │  (PDFs)    │ │            │ │  1536-d)  │ │              │
     └────────────┘ └────────────┘ └───────────┘ └──────────────┘
```

---

## Stack & why

| Layer | Choice | Why |
|---|---|---|
| Framework | **Next.js 16 (App Router)** | Server Components for data-heavy dashboards, Route Handlers for the API surface, first-class streaming. |
| Hosting | **Cloudflare Workers** via `@opennextjs/cloudflare` | Global edge, cheap cold-starts, R2 binding lives on the same runtime as the app — no cross-cloud egress for file reads. |
| Storage | **Cloudflare R2** | Zero-egress PDF storage bound directly to the Worker. |
| Vectors | **Pinecone** | Managed, serverless, index-dimension awareness to auto-align embedding size. |
| Embeddings | **OpenAI `text-embedding-3-small`** | 1536-d default, variable-dim support, strong cost/quality ratio for doc chunks. |
| Generation | **Anthropic Claude Sonnet** | Better instruction following for strict-JSON template output; tool-call discipline. |
| Auth | **NextAuth (JWT) + Google OAuth** | No session DB needed, email-verified allowlist is a single callback. |
| UI | **React 19, Tailwind v4, shadcn/ui, Base UI** | Fully owned components, no heavy UI library lock-in. |
| Validation | **Zod 4** | Schema-driven template instances, runtime-safe env parsing. |
| Env | **`@t3-oss/env-nextjs`** | Typed `env` object, fails at build time if required keys are missing. |
| PDF parsing | **`unpdf`** | Works inside Workers (no native deps). |
| PDF generation | **`@react-pdf/renderer`** | Server-rendered PDF export of filled templates. |

---

## Key flows

### 1. PDF ingestion

Upload → size/mime guard → extract → chunk (approx-token window) → embed → upsert → persist metadata.

```ts
export async function ingestPdfDocument(params: {
  buffer: Uint8Array;
  originalName: string;
  mimetype: string;
  uploadedBy: string;
}): Promise<IngestPdfResult> {
  const documentId = crypto.randomUUID();
  const { displayName, safeKeySegment } = normalizeOriginalFilename(params.originalName);
  const r2Key = `documents/${documentId}-${safeKeySegment}`;

  validateClientPdfUpload({ buffer: params.buffer, mimetype: params.mimetype, size: params.buffer.length });

  const { text, numPages } = await extractTextFromPdf(params.buffer);
  const quality = assessExtractedTextQuality(text, {
    minChars: await getMinExtractableChars(),
    warnBelowChars: await getWarnBelowExtractableChars(),
  });
  assertMeetsMinimumTextForIndexing(quality, minChars);

  await putObject({ key: r2Key, body: params.buffer, contentType: "application/pdf" });

  const chunks = chunkTextByTokens(quality.normalizedText);
  const vectors = await embedTexts(chunks);

  const records = chunks.map((_, i) => ({
    id: `${documentId}_chunk_${i}`,
    values: vectors[i],
    metadata: {
      document_id: documentId,
      document_name: displayName,
      chunk_index: i,
      chunk_count: chunks.length,
      r2_key: r2Key,
      ingestion_pipeline: "pdf_v1",
    },
  }));

  await (await getPineconeVectorIndex()).upsert({ records });
  await putDocumentMeta({ id: documentId, name: displayName, r2Key, chunkCount: chunks.length, ... });
  return { documentId, r2Key, chunkCount: chunks.length, numPages, pineconeUpserted: records.length };
}
```

**Chunking decision** — overlapping character window instead of a real BPE tokenizer to avoid bundling ~1 MB of tokenizer tables into the Worker:

```ts
// ~4 chars per token for typical English prose;
// avoids bundling the 1 MB gpt-tokenizer BPE table into the Worker.
const CHARS_PER_TOKEN = 4;

export function chunkTextByTokens(text: string, maxTokens = 800, overlapTokens = 100): string[] {
  const maxChars = maxTokens * CHARS_PER_TOKEN;
  const step = Math.max(1, maxChars - overlapTokens * CHARS_PER_TOKEN);
  /* ... slide a window, trim, return chunks ... */
}
```

**Embedding dimension alignment** — the embedding layer auto-negotiates with Pinecone so a new index with a different dimension "just works":

```ts
async function getDesiredEmbeddingDimensions(): Promise<number | undefined> {
  const fromEnv = await parseEnvEmbeddingDimensions();
  if (fromEnv !== undefined) return fromEnv;
  return getPineconeIndexDimension(); // cached after first describe
}
```

### 2. Retrieval

Vector search plus a lightweight metadata-graph ranking, returned as a single `RetrievalResult` the chat layer consumes verbatim:

```ts
export async function retrieveContext(params: {
  message: string;
  topK?: number;
  graphLimit?: number;
}): Promise<RetrievalResult> {
  const topK = clamp(params.topK ?? 8, 1, 20);
  const graphLimit = clamp(params.graphLimit ?? 5, 1, 20);

  const [vector] = await embedTexts([params.message]);
  const index = await getPineconeVectorIndex();

  const [pineconeResult, graphContext] = await Promise.all([
    index.query({ vector, topK, includeMetadata: true }),
    queryMetaGraphContext({ query: params.message, limit: graphLimit }),
  ]);

  return {
    query: params.message,
    retrieval: { topK, chunkCount: pineconeResult.matches.length, graphCount: graphContext.length },
    chunks: pineconeResult.matches.map(toRetrievalChunk),
    graphContext,
  };
}
```

### 3. Grounded chat

The vanilla RAG path passes a structured context block to Claude with an explicit "don't invent" system prompt:

```ts
function buildSystemPrompt() {
  return [
    "You are an assistant for internal knowledge retrieval.",
    "Ground your response in the provided context snippets.",
    "If context is insufficient, say what is missing instead of inventing facts.",
  ].join(" ");
}
```

### 4. Template-mode chat

Templates are authored once as a `TemplateDefinition` (sections → typed fields → LLM hints). The chat endpoint asks Claude for strict JSON in one of three shapes:

```ts
// The LLM is constrained to one of three output envelopes:
//   { mode: "conversation",   replyMarkdown }
//   { mode: "template_fill",   replyMarkdown, instance }
//   { mode: "template_update", replyMarkdown, instance }
```

A zod schema is **derived from the template definition at request time** and used to validate the `instance` the model returns. If validation fails, one silent retry is attempted before bubbling a `TemplateValidationError`:

```ts
const instanceSchema = buildInstanceSchema(definition);
const result = instanceSchema.safeParse(envelope.instance);
if (!result.success) {
  throw new TemplateValidationError(
    `Instance failed validation: ${result.error.issues
      .slice(0, 5)
      .map((i) => `${i.path.join(".") || "<root>"}: ${i.message}`)
      .join("; ")}`,
  );
}
```

A small regex fallback handles cases where the classifier says `"conversation"` but the user obviously meant to edit the draft ("change the due date", "fix the amount", …):

```ts
const TEMPLATE_UPDATE_VERB_PATTERN =
  /\b(update|change|replace|redo|refill|revise|edit|fix|correct|rewrite)\b/i;
```

---

## Auth & access control

- **Fail-closed** by default: if both the email and domain allowlists are empty, all sign-ins are denied.
- The allowlist is checked server-side in the NextAuth `signIn` callback against Google's `email_verified` flag:

```ts
async signIn({ account, profile, user }) {
  if (account?.provider !== "google") return false;
  if ((profile as any)?.email_verified === false) return false;
  const email = (profile as any)?.email ?? user?.email ?? null;
  return isEmailAllowed(email, loadAccessPolicy());
}
```

- Sessions are JWT, so the Worker never touches a session database.
- Every mutating route goes through a `requireAuthedUser()` helper so every handler has one obvious line of auth plumbing.

---

## Edge runtime decisions

The Worker runtime was the dominating constraint. A few calls made because of it:

- **One env accessor.** The app code never talks to `process.env` or `env` directly. A single helper resolves the Cloudflare binding at runtime and falls back to `process.env` during `next dev`:

  ```ts
  export async function getEnvVar(name: keyof CloudflareEnv): Promise<string | undefined> {
    const env = await getCloudflareEnv();
    return normalize(env[name]) ?? normalize(process.env[name as string]);
  }
  ```

- **No native PDF deps.** `unpdf` was picked specifically because it runs inside Workers.
- **No tokenizer bundle.** The char-based chunker above was a conscious trade — less precise than BPE, but it keeps the Worker tiny and cold-start fast.
- **`nodejs_compat` + `global_fetch_strictly_public`** flags enabled in `wrangler.jsonc` to allow the SDKs (Anthropic, OpenAI, Pinecone) to run unmodified.

---

## Project shape

```text
src/
  app/
    (auth)/login/              # NextAuth sign-in page
    api/
      auth/[...nextauth]/      # Google OAuth handler
      documents/               # upload, list, download, delete
      chat/generate/           # grounded chat + template-mode chat
      generate/                # template -> PDF export
      query/                   # pure retrieval endpoint
      templates/               # template CRUD
    dashboard/                 # signed-in workspace UI
  components/ui/               # shadcn/ui primitives
  lib/
    auth.ts, auth-access.ts    # NextAuth config + allowlist policy
    route-auth.ts              # requireAuthedUser() guard
  server/
    ingest-pdf.ts              # PDF -> chunks -> vectors -> Pinecone
    retrieve-context.ts        # vector + meta-graph retrieval
    chat-local.ts              # grounded-chat composition
    chat-template.ts           # strict-JSON template-mode chat
    generate-answer.ts         # one-shot Q&A with citations
    embeddings.ts              # OpenAI embeddings + dim negotiation
    pinecone.ts                # index + dimension cache
    r2.ts                      # put/get/list/delete
    cloudflare.ts              # single env/binding accessor
    chunk-text.ts              # approx-token overlapping chunker
    pdf.ts                     # unpdf-based text extraction
    templates/                 # TemplateDefinition + schema builder
```

---

## Notable engineering decisions

1. **Strict-JSON template output over tool calling.** Tool calls are flexible but ad-hoc; a single discriminated-union envelope + zod schema is simpler to test, log, and replay.
2. **Schema derived from data.** The zod validator for an instance is generated from the `TemplateDefinition` on every request, so adding a new template is purely data — no code change, no deploy.
3. **One retrieval shape, many consumers.** `RetrievalResult` is consumed identically by grounded chat, template fill, and the one-shot `/api/generate` endpoint. One place to improve ranking benefits all three.
4. **Ingestion policy is a first-class module.** MIME, size, minimum-extractable-text, and warning thresholds all live in `document-policy.ts` with typed accessors — tunable per environment without redeploying app code.
5. **Typed Workers env.** `CloudflareEnv` is generated by `wrangler types` and used everywhere, so missing bindings are a TypeScript error, not a runtime 500.
6. **Fail-closed auth, logged-in everything.** The app has no anonymous surface. Every route handler starts with `requireAuthedUser()`; there is literally no path to reach a model call without a valid Google-verified, allowlisted user.

---

## What I'd improve next

- Replace the metadata-graph heuristic with a small reranker for multi-hop retrieval.
- Move from character-approximate chunking to a real tokenizer once Workers bundle limits allow.
- Stream template-fill output as a *partial instance* so the UI can render field-by-field as the JSON arrives.
- Add an eval harness: a fixtures folder of (definition, transcript, expected-instance) triples replayed nightly against a pinned model version.
- Per-document ACLs (the current model is "shared workspace" with an email allowlist — fine for the original use case, not for multi-tenant).

---

*This repo intentionally contains no runnable code. It's a case study of the system designed and built end-to-end: ingestion pipeline, retrieval, grounded chat, schema-validated template mode, Worker runtime, and auth.*
