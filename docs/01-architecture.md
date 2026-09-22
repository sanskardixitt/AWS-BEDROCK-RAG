# 01 — Architecture

## What problem RAG solves

A language model knows what was in its training data. It does not know your company's
refund policy, last quarter's numbers, or the PDF you uploaded five minutes ago. There
are three ways to fix that:

| Approach | How it works | When it's right |
|---|---|---|
| **Fine-tuning** | Retrain the model's weights on your data | Teaching *style*, *format*, or a *task*. Expensive, slow, and stale the moment your data changes. |
| **Long context** | Paste all your documents into every prompt | Small, stable corpora. Costs scale linearly with corpus size × number of questions. |
| **RAG** | Retrieve the few relevant passages, then ask | Large or changing corpora. Costs scale with *answer* size, not corpus size. |

RAG wins for document Q&A because of one property: **adding a document is a write, not a
retrain.** Upload a PDF and it is answerable in seconds. Delete it and it stops being
answerable. No model training is involved at any point.

The mental model worth keeping: **RAG is search plus summarisation.** The language model
is doing reading comprehension over passages your search layer picked. If your search
layer picks the wrong passages, no model — however capable — can save the answer. Most
RAG failures are retrieval failures wearing a generation costume.

---

## The two pipelines

Every RAG system has exactly two flows, and they run at completely different times and
frequencies. Confusing them is the most common architectural mistake.

### Pipeline A — Ingestion (write path, rare, slow, batch)

```
PDF upload
   │
   ▼
[1] Extract text       PDF bytes → per-page text
   │
   ▼
[2] Chunk              pages → ~1200-char overlapping passages
   │
   ▼
[3] Embed              each chunk → 1024 floats (Titan)
   │
   ▼
[4] Index              vectors + metadata → S3 Vectors
```

Runs once per document. Latency doesn't matter much (it's asynchronous). Correctness
matters enormously — a bad chunk is wrong for every future question.

### Pipeline B — Query (read path, frequent, fast, interactive)

```
Question
   │
   ▼
[1] Embed question     text → 1024 floats (same model as ingestion!)
   │
   ▼
[2] Search             nearest neighbours in vector space → top 5 chunks
   │
   ▼
[3] Gate               drop anything above the distance threshold
   │
   ├── nothing left ──► return "I don't know" (never calls the LLM)
   ▼
[4] Augment            build a prompt containing only those chunks
   │
   ▼
[5] Generate           Claude answers, citing passage numbers
```

Runs on every question. A user is waiting, so latency matters. Typical end-to-end is
1.5–4 seconds, and the generation step dominates.

> **The single most important invariant:** steps A[3] and B[1] must use the *same
> embedding model with the same dimension settings*. Embeddings from two different
> models are not comparable — they're coordinates in different spaces. Change your
> embedding model and you must re-index your entire corpus. This is why
> `EmbeddingModelId` and `EmbeddingDimension` are template parameters, and why the
> index's `Dimension` is immutable in CloudFormation.

---

## Why vector search works (the 60-second version)

An embedding model maps text to a point in 1024-dimensional space, trained so that
*text with similar meaning lands near each other*. "How do I get my money back?" and
"Refunds are available within 30 days" share almost no words, but land close together.
That's the whole trick, and it's why this beats keyword search for questions phrased in
the user's own words.

"Near" is measured by **cosine distance** — the angle between two vectors, ignoring their
length:

```
distance = 1 - cos(θ)

0.0  identical direction   (same meaning)
0.3  closely related       (typical good match)
1.0  orthogonal            (unrelated)
2.0  opposite direction
```

We use cosine rather than Euclidean because Titan returns normalised (unit-length)
vectors, where the two rank identically — and cosine is the convention every embedding
model is evaluated against.

**Lower is better.** This trips people up constantly, because many vector databases
return a *similarity score* where higher is better. S3 Vectors returns *distance*.
Sorting the wrong way silently inverts your entire ranking, and the system still returns
plausible-looking answers — which is why it's such a nasty bug.

---

## Component-by-component

### S3 (documents bucket) — the trigger and the source of truth

Uploading to `documents/*.pdf` emits an `ObjectCreated` event that invokes the ingest
Lambda. No polling, no cron, no queue to manage.

Three configuration choices worth understanding:

- **Prefix + suffix filters on the event.** The ingest Lambda writes manifest files back
  into the *same bucket*. Without the `documents/` prefix filter, those writes would
  re-trigger the function, which would write another manifest, which would trigger it
  again — an infinite recursive loop. This is a genuinely famous AWS bug that has
  generated real five-figure bills. The fix is the filter, or a separate bucket.
- **Versioning on.** Overwriting a source document is recoverable.
- **Lifecycle rules.** Non-current versions expire after 7 days, and incomplete
  multipart uploads are aborted after 1. Abandoned multipart uploads are invisible in the
  console's object listing but are billed indefinitely — a classic silent cost leak.

### S3 Vectors — the vector store

A *vector bucket* is a distinct bucket type. It does not respond to `GetObject`; it has
its own API (`s3vectors`), and its own boto3 client. Inside it, a *vector index* holds
records of `{key, vector, metadata}`.

The index is created with three immutable properties:

| Property | Value | Why it's immutable matters |
|---|---|---|
| `Dimension` | 1024 | Must match the embedding model. Changing it replaces the index and drops every vector. |
| `DistanceMetric` | cosine | Ditto. |
| `NonFilterableMetadataKeys` | `["text"]` | Can't be changed later, so decide up front. |

That last one deserves explanation. Metadata is filterable by default, so you can query
"only chunks from doc X". But filterable metadata is indexed and size-capped. The chunk
body (`text`) is ~1200 bytes and you never filter on it — you just want it returned with
results so you don't need a second round-trip to fetch the passage. Declaring it
non-filterable keeps it out of the filter index while still returning it.

### Lambda — all the compute

Three functions, each with one job:

| Function | Trigger | Job |
|---|---|---|
| `ingest` | S3 ObjectCreated | Extract, gate on quality, then either index directly or dispatch to Textract |
| `textract-callback` | SNS | Collect OCR results and run the same indexing pipeline |
| `query` | Function URL | Retrieve → augment → generate |

Lambda fits because the workload is **spiky and event-shaped**. Documents arrive
unpredictably; questions arrive unpredictably. A server sized for the peak sits idle and
billed the rest of the time. At this scale Lambda's cost is effectively zero and its
operational burden is zero.

The trade-off is **cold starts**: the first invocation after idleness pays ~400–800ms to
initialise the runtime and import boto3. For a 2-second RAG query this is tolerable. If
it weren't, the fix is provisioned concurrency — which costs money whether or not you
use it, exactly the trade-off Lambda otherwise saves you from.

### Bedrock — the models

Two different models for two different jobs:

- **Titan Text Embeddings V2** (`amazon.titan-embed-text-v2:0`) — text → vector. Called
  once per chunk at ingest, once per question at query. $0.02 per million tokens.
- **Claude Haiku 4.5** (`global.anthropic.claude-haiku-4-5-20251001-v1:0`) — reading
  comprehension over retrieved passages. Called at most once per question.

Generation uses the **Converse API** rather than `InvokeModel`. Converse gives one
uniform request shape across every Bedrock model; `InvokeModel` requires you to
hand-build each vendor's bespoke JSON body. Swapping Claude for Llama becomes a
parameter change instead of a rewrite.

The `us.` prefix on the Claude ID marks a **cross-region inference profile**. Bedrock
routes your request across several US regions to find capacity, which improves
availability and throughput. The catch is IAM: you must grant access to *both* the
inference-profile ARN in your account *and* the underlying foundation-model ARNs in
every region the profile can route to. Granting only the profile produces an
`AccessDeniedException` that looks like the model isn't enabled. The template grants
both — see the `InvokeChatModel` statement.

### Textract — conditional OCR

Only reached when pypdf's output is degenerate. Because Textract cannot process
multi-page PDFs synchronously, this path is asynchronous:

```
ingest Lambda ──StartDocumentTextDetection──► Textract
                                                 │  (seconds to minutes)
                                                 ▼
                                          SNS topic
                                                 │
                                                 ▼
                                    textract-callback Lambda
                                                 │
                                          GetDocumentTextDetection
                                                 │
                                                 ▼
                                       shared indexing pipeline
```

Note the IAM subtlety: Textract publishes to SNS using **its own role**, not your
Lambda's. That's why `TextractPublishRole` exists with a trust policy naming
`textract.amazonaws.com`, and why the ingest Lambda needs `iam:PassRole` on it. This
"service assumes a role you hand it" pattern recurs across AWS (CodePipeline, Glue,
SageMaker) and is worth internalising.

### CloudWatch — logs, metrics, alarms

Three distinct things people lump together:

**Logs** are structured JSON, one object per line, so they're queryable:

```
fields @timestamp, key, chunks, embedding_tokens
| filter event = "ingest_complete"
| stats sum(chunks), avg(embedding_tokens) by key
```

**Metrics** are emitted via **EMF (Embedded Metric Format)** — a specially shaped log
line that CloudWatch parses into metrics asynchronously. The alternative,
`PutMetricData`, is a synchronous billed API call sitting on your critical path. EMF is
a `print()` statement. In production you'd use `aws-lambda-powertools`, which wraps
exactly this; it's hand-written in `observability.py` so the mechanism is visible.

Custom metrics chosen deliberately to answer real questions:

| Metric | Question it answers |
|---|---|
| `EmbedLatencyMs` / `RetrieveLatencyMs` / `GenerateLatencyMs` | Which stage is slow? (Almost always generation.) |
| `TopCosineDistance` | Is retrieval quality degrading over time? |
| `RefusedNoContext` vs `AnsweredFromContext` | What fraction of questions can't be answered? A rising refusal rate means a corpus gap. |
| `InputTokens` / `OutputTokens` | Your Bedrock bill, live. |
| `TextractFallbacks` | How many documents hit the paid OCR path. |

**Dimension cardinality is a cost trap.** Every unique combination of dimension values
is a separate metric at ~$0.30/month. Putting a document ID or user ID in a *dimension*
creates thousands of metrics. Those belong in the log *body*, where they're free and
still queryable. The code uses only low-cardinality dimensions (`Service`, `Extractor`).

---

## Request walkthrough: one question, end to end

```
POST https://<url>.lambda-url.ap-south-1.on.aws/
Authorization: AWS4-HMAC-SHA256 ...          ← SigV4, or you get 403
{"question": "How long is the refund window?"}
```

1. **Function URL** validates the SigV4 signature against IAM. Unsigned requests never
   reach your code — no Bedrock spend, no Lambda duration billed for garbage traffic.
2. **Query Lambda** parses the body (handling base64 encoding, which Function URLs apply
   for some content types).
3. **Embed** the question via Titan → 1024 floats. ~80ms.
4. **Search** S3 Vectors for the 5 nearest vectors. ~50–150ms. Returns chunk text,
   source key, page range and distance.
5. **Gate.** Discard anything with distance > 0.75.

   This step is what separates a real system from a demo. Nearest-neighbour search
   *always* returns k results — ask "who is the CEO?" against a refunds handbook and it
   will happily return the 5 least-unrelated chunks. Without the gate, those get fed to
   the LLM, which produces a fluent, well-cited, completely invented answer. With the
   gate, nothing survives and we return "I don't know" **without calling the LLM at
   all** — better answer, and cheaper.
6. **Augment.** Build a numbered context block. The numbering is load-bearing: the model
   cites `[2]`, and the response array must be in the same order for that citation to
   resolve.
7. **Generate.** Claude answers at `temperature=0` under a system prompt that forbids
   outside knowledge and mandates citations.
8. **Respond** with the answer, the sources (with page numbers and distances), token
   usage, and a per-stage latency breakdown.

The response deliberately exposes distances and token counts. When an answer looks
wrong, the first question is always "was it retrieval or generation?" — and the distance
values answer it immediately.

---

## Failure modes and how each is handled

| Failure | Without handling | What this system does |
|---|---|---|
| Corrupt PDF | Lambda throws, event lost | Retries, then DLQ + alarm |
| Scanned PDF | Empty index, no error | Quality gate → Textract |
| One bad page in a good PDF | Whole document fails | Per-page try/except; other pages still index |
| Re-uploading an edited document | Stale chunks linger forever | Manifest-based delete-then-write |
| Question with no answer in corpus | Confident hallucination | Distance gate → explicit refusal |
| Bedrock throttling | Request fails | Standard-mode retries with exponential backoff |
| Runaway invocation loop | Unbounded Bedrock bill | Reserved concurrency + AWS Budget alert |
| S3 event key with spaces | `NoSuchKey` | `unquote_plus` on the event key |
| Textract job fails | Infinite Lambda retries | Logged, not re-raised (retrying can't help) |

The manifest deserves a closer look because it's the one most tutorials skip. Document
v1 produced 40 chunks (`doc:00000`–`doc:00039`). You edit it; v2 produces 30. Writing v2
overwrites keys `00000`–`00029`, but `00030`–`00039` from v1 survive — stale text that
will be retrieved and cited as current. S3 Vectors has no "delete where
metadata.doc_id = X" operation, so we record exactly which keys belong to each document
in `_manifests/{doc_id}.json` and delete them before writing. Every production vector
pipeline needs some version of this bookkeeping.

---

## Scale characteristics

| Dimension | This design | Where it breaks | What you'd change |
|---|---|---|---|
| Corpus size | Comfortable to ~10M vectors | S3 Vectors caps at 2B/index | Shard across indexes |
| Query rate | Reserved concurrency 2 (deliberate) | Raise it | Add caching; watch Bedrock RPM quotas |
| Document size | ~2000 chunks (Lambda 5-min timeout) | Very large PDFs | Step Functions, or fan out per page via SQS |
| Ingestion burst | 5 concurrent | Bedrock embedding quota | SQS between S3 and Lambda for buffered backpressure |
| Query latency | 1.5–4s | Generation dominates | Stream the response; retrieval is already fast |

The first thing to break at scale is **Bedrock's per-account requests-per-minute quota**,
not any storage or compute limit. Retries with backoff handle transient throttling; a
sustained load needs a quota increase request.

---

Continue to [02-design-decisions.md](02-design-decisions.md) for why each choice was made
over the alternatives.
