# 02 — Design decisions

Written as lightweight ADRs (Architecture Decision Records): the decision, the
alternatives actually considered, why they lost, and what would make us revisit. This
format is standard in engineering orgs, and "walk me through a decision you made and its
trade-offs" is the most common senior-level interview question there is.

---

## ADR-1 — Vector store: S3 Vectors

**Decision:** Amazon S3 Vectors (GA December 2025).

**Alternatives:**

| Option | Idle cost | Verdict |
|---|---|---|
| **OpenSearch Serverless** | **~$700/mo** | Bedrock Knowledge Bases' console default. Bills a 4-OCU minimum continuously, whether you query it or not. Catastrophic on a learning account. |
| **S3 Vectors** | ~$0 | $0.06/GB-month storage, per-request query charge. Purpose-built, sub-second search, 2B vectors/index. **Chosen.** |
| **Flat file in S3 + NumPy in Lambda** | $0 | Genuinely viable to ~50k chunks. Teaches the math. But you rebuild filtering, batching and pagination yourself. |
| **DynamoDB** | ~$0 (free tier) | No vector index — you'd scan and score every item in Lambda. Wrong tool. |
| **pgvector on RDS** | ~$15/mo | Real ANN indexing and SQL joins. But it's a server you patch and back up — not serverless. |
| **Pinecone / Weaviate** | varies | Excellent products, outside AWS, and outside the point of this exercise. |

**Why S3 Vectors won:** it is the only option that is simultaneously managed, genuinely
pay-per-use, and integrated with Bedrock. The zero-idle-cost property is what makes it
correct for an account you'll leave deployed for weeks and query occasionally.

**Revisit when:** you need sub-10ms p99, hybrid keyword+vector search, or heavy filtered
queries. Then OpenSearch earns its cost — and at that point you'd have the traffic to
justify it.

**The trap worth stating plainly:** if you follow the Bedrock Knowledge Base console
wizard and accept defaults, you will provision OpenSearch Serverless. Many people have
discovered this on a monthly invoice.

---

## ADR-2 — PDF extraction: pypdf first, Textract on failure

**Decision:** Extract with `pypdf`; measure characters-per-page; escalate to Textract
only below 100 chars/page.

**The evaluation you asked for:**

A "PDF" is two unrelated file types sharing an extension:

- **Digital-native** (exported from Word, LaTeX, a browser): text is stored as text
  objects. `pypdf` reads it in milliseconds, free, inside the Lambda you already pay for.
- **Scanned** (a photo of paper): contains only images. `pypdf` returns empty strings no
  matter how good your code is. OCR is genuinely required.

| Option | Cost | Handles scanned? | Verdict |
|---|---|---|---|
| pypdf only | $0 | No | Silently indexes nothing for scanned PDFs — the worst failure mode |
| Textract always | $1.50/1,000 pages | Yes | Pays OCR prices for text that was already machine-readable |
| **pypdf → Textract fallback** | $0 for ~90% of docs | Yes | **Chosen** |

Textract's free tier is 1,000 pages/month of `DetectDocumentText` for your **first three
months only** — it is not perpetual. After that every page is billed.

**Why the tiered approach is also the better *learning* outcome:** the fallback path is
asynchronous (`StartDocumentTextDetection` → SNS → second Lambda), which is the canonical
AWS long-running-job pattern. You get the cheap answer *and* the more valuable
architecture lesson.

**The threshold is a heuristic, and heuristics need escape hatches.** 100 chars/page is
tuned for prose. A slide deck or a diagram-heavy datasheet may legitimately sit below it
and get sent to Textract unnecessarily. It's exposed as `MinCharsPerPage`, and
`EnableTextractFallback=false` guarantees zero Textract spend.

**Not chosen: Textract `AnalyzeDocument`** (tables/forms, $15–$50 per 1,000 pages).
10–30× the price, and it returns structure we'd discard during chunking anyway. It's the
right call only if your documents are genuinely tabular and you preserve that structure
downstream.

---

## ADR-3 — Chunking: paragraph-aware, ~1200 chars, ~200 overlap

**Decision:** Pack whole paragraphs until a ~1200-character budget is reached; overlap
consecutive chunks by ~200 characters; carry page numbers through.

**Why chunk at all:** an embedding compresses whatever you give it into *one* fixed
vector. Embed a 40-page document and you get a vector meaning "vaguely about finance" —
useless for a specific question. Smaller units stay specific.

**Size trade-off — the highest-leverage knob in the system:**

| Chunk size | Retrieval precision | Context completeness | Failure mode |
|---|---|---|---|
| ~300 chars | High | Poor | Retrieves a sentence lacking the context to interpret it |
| **~1200 chars** | Good | Good | **Chosen** — roughly a paragraph or two |
| ~4000 chars | Poor | High | The vector averages several topics; retrieval gets vague |

**Why overlap:** a hard cut at character 1200 can split "The refund window is" from
"30 days". Neither half answers the question. ~200 characters of overlap means any given
sentence appears intact in at least one chunk. Cost: ~17% more vectors — i.e. fractions
of a cent.

**Why paragraph-aware over fixed-size:** fixed-size splitting is one line of code and
noticeably worse, because it severs sentences mid-thought and the resulting embeddings
are muddier. Packing whole paragraphs keeps semantic units intact. We only fall back to
hard character splitting for a paragraph that alone exceeds the budget (tables, OCR
blobs).

**Why page attribution:** it costs almost nothing to carry the page number through, and
it upgrades citations from "handbook.pdf" to "handbook.pdf, page 7". Citations are the
difference between a demo and something a person will act on.

**Not chosen: semantic chunking** (embed every sentence, cut where similarity drops).
Better quality, but it multiplies embedding calls at ingest and adds real complexity. The
right upgrade *after* you've measured that chunking is your bottleneck.

**The honest caveat:** these numbers are reasonable defaults, not tuned optima. The
correct values depend on your documents, and the only way to find them is an evaluation
set — see [04-production-gaps.md](04-production-gaps.md).

---

## ADR-4 — A relevance threshold, and refusing to answer

**Decision:** Discard retrieved chunks with cosine distance > 0.75. If nothing survives,
return "I don't know" **without invoking the LLM**.

**Why this is not optional:** nearest-neighbour search always returns *k* results. It has
no concept of "nothing relevant here" — it returns the *k* least-bad matches from
whatever you have. Ask "who is the CEO?" against a refunds handbook and you get 5
unrelated chunks, each with a distance around 1.0.

Hand those to a language model and it will write a fluent, confidently-cited, entirely
invented answer. The citations make it *worse*, because they make it look verified.

The threshold converts a wrong answer into an honest one, and it is simultaneously the
cheapest optimisation in the system: an unanswerable question costs one embedding call
(~$0.000001) instead of a full generation.

**Tuning:** 0.75 is deliberately permissive. Too strict and you refuse answerable
questions; too loose and hallucinations return. The `RefusedNoContext` /
`AnsweredFromContext` metrics on the dashboard exist precisely so you can tune this
against real traffic rather than by guessing.

**Defence in depth:** the threshold is the first layer; the system prompt's explicit
"reply exactly 'I don't know based on the provided documents'" instruction is the second.
Neither alone is sufficient.

---

## ADR-5 — Generation: Converse API, temperature 0, mandatory citations

**Decision:** `bedrock-runtime.converse()`, `temperature=0`, system prompt forbidding
outside knowledge and requiring `[n]` citations.

- **Converse over InvokeModel:** one uniform request/response shape across every Bedrock
  model. `InvokeModel` requires hand-building each vendor's bespoke JSON. Swapping models
  becomes a parameter change.
- **Temperature 0:** this is extractive QA, not creative writing. The same question
  should give the same answer; "creativity" here manifests as invented facts.
- **Citations:** they make answers auditable. The passage numbering in the prompt matches
  the order of the `sources` array in the response, so `[2]` resolves to `sources[1]`.
  Get that mapping wrong and citations point at the wrong document — arguably worse than
  none.
- **Claude Haiku 4.5:** the cheapest capable model on Bedrock. RAG generation is reading
  comprehension over supplied text, which is a task small models do well. It's a
  parameter — swap to Sonnet if answer quality justifies the cost.

---

## ADR-6 — API surface: Lambda Function URL with IAM auth

**Decision:** Function URL, `AuthType: AWS_IAM`.

| Option | Cost | Auth | Verdict |
|---|---|---|---|
| **Function URL + IAM** | Free | SigV4 | **Chosen** — no key to leak, IAM-controlled |
| Function URL + NONE | Free | None | Anyone who finds the URL spends your Bedrock budget |
| HTTP API Gateway | Free 1M req/mo for 12 months, then $1/M | Many | Adds throttling, custom domains, JWT auth — needed later, not now |
| REST API Gateway | $3.50/M | Many | More features, 3.5× the price |

`AuthType: NONE` deserves emphasis as an anti-pattern: your endpoint invokes a paid AI
model on every request. A public URL with no auth is an open invitation to spend your
money. The cost of `AWS_IAM` is that clients must sign requests — which is why
`scripts/ask.py` exists, and writing it teaches you SigV4.

**Revisit when:** you need rate limiting per client, a custom domain, WAF, or
non-AWS-credentialed callers. That's API Gateway's job.

---

## ADR-7 — IaC: SAM, with native S3 Vectors CloudFormation resources

**Decision:** AWS SAM. `AWS::S3Vectors::VectorBucket` and `AWS::S3Vectors::Index` are
supported natively, so no custom resources or bootstrap scripts.

| Option | Verdict |
|---|---|
| **SAM** | **Chosen.** Purpose-built for serverless; `sam build/deploy/logs/delete` covers the whole lifecycle. |
| Raw CloudFormation | SAM *is* CloudFormation plus a transform. All the verbosity, none of the shortcuts. |
| Terraform | Better multi-cloud story and stronger job-market signal, but more boilerplate for serverless and no `sam local`. Worth learning next. |
| CDK | Real programming language, good for complex logic. The abstraction hides what's being created, which is the opposite of what you want while learning. |
| Console clicking | Not reproducible, not reviewable, and easy to leave billable orphans behind. |

**The teardown argument is the decisive one for a learning account.** `sam delete`
removes every resource in the stack. Resources created by clicking must be found and
deleted by hand, and the ones you forget keep billing.

**Deliberate detail:** the ingest function references the documents bucket by a
`!Sub`-constructed *name string*, not `!Ref DocsBucket`. SAM's S3 event source attaches a
notification to the bucket, so a `!Ref` back from the function creates a circular
dependency CloudFormation rejects. Constructing the name from `${ProjectName}` and
`${AWS::AccountId}` breaks the cycle.

---

## ADR-8 — Observability: hand-rolled EMF over Powertools

**Decision:** Write structured JSON logs and EMF metric lines directly, rather than
adding `aws-lambda-powertools`.

**Reasoning:** Powertools is genuinely the right production choice and you should use it
at work. But it would hide exactly the mechanism worth learning here. ~60 lines in
`observability.py` make it concrete that:

- A "structured log" is just `json.dumps` to stdout.
- A "custom metric" is just a log line with an `_aws` key that CloudWatch parses out of
  band.
- `PutMetricData` — the obvious alternative — is a synchronous, billed API call on your
  critical path. EMF is a `print()`.

**Revisit:** on any real project, use Powertools. It adds tracing, idempotency, batch
processing and parameter fetching, all well-tested.

---

## ADR-9 — Cost guardrails as infrastructure

**Decision:** Reserved concurrency, explicit log retention, and an AWS Budget are all
*in the template*, not in a runbook.

| Guardrail | Without it |
|---|---|
| `ReservedConcurrentExecutions: 2` on query | A retry loop or load test invokes Bedrock thousands of times in seconds |
| `RetentionInDays: 7` on log groups | CloudWatch Logs defaults to **never expire** — the most common silent AWS cost leak |
| `AWS::Budgets::Budget` at $5 | You learn your spend from the invoice |
| Lifecycle rule aborting multipart uploads | Failed uploads bill forever and are invisible in the object list |
| Distance gate short-circuiting the LLM | Every unanswerable question costs a full generation |

The principle: **a cost control that lives in a document is not a cost control.** If it
isn't enforced by the platform, it doesn't exist.

---

## ADR-10 — Idempotent re-ingestion via manifests

**Decision:** Record every vector key written for a document in
`_manifests/{doc_id}.json`; delete those keys before writing a new version.

**The problem:** v1 produces 40 chunks (`doc:00000`–`doc:00039`); the edited v2 produces
30. Writing v2 overwrites `00000`–`00029`, leaving `00030`–`00039` as stale text that
will be retrieved and cited as current. This is a correctness bug, not untidiness.

**Alternatives:**

- *Delete by metadata filter* — S3 Vectors has no such operation.
- *`ListVectors` and filter client-side* — scans the entire index per document. O(index)
  for an O(document) task.
- *Content-hash keys* — deduplicates nicely but makes finding a document's vectors for
  deletion harder, not easier.
- **Manifest** — one small S3 object per document, O(1) lookup. **Chosen.**

**Ordering matters:** delete-then-write. A crash mid-way leaves the index *missing* data
— detectable, and fixed by re-uploading. Write-then-delete would leave a *mixture* of old
and new chunks: silent and corrupting.

---

## Summary

| # | Decision | Primary driver |
|---|---|---|
| 1 | S3 Vectors | Zero idle cost vs. OpenSearch's ~$700/mo floor |
| 2 | pypdf → Textract fallback | Pay for OCR only when OCR is genuinely needed |
| 3 | Paragraph chunks, 1200/200 | Retrieval precision vs. context completeness |
| 4 | Distance threshold + refusal | Hallucination prevention, and cost |
| 5 | Converse, temp 0, citations | Portability, determinism, auditability |
| 6 | Function URL + IAM | Free, and no unauthenticated access to a paid model |
| 7 | SAM | Reliable teardown; native S3 Vectors support |
| 8 | Hand-rolled EMF | Learning value; Powertools in production |
| 9 | Guardrails in the template | Enforced controls beat documented ones |
| 10 | Manifest-based re-ingest | Correctness under document updates |
