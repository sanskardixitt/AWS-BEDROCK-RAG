# 03 — Cost model

Every number here is arithmetic you can redo. Prices are US East (N. Virginia) as of
September 2026 and **change** — verify against the live pricing pages before trusting
them for anything that matters.

> **You are deploying to ap-south-1 (Mumbai).** Per-unit prices differ modestly by region
> (Lambda and S3 in Mumbai are within a few percent of N. Virginia; Bedrock token prices
> for models served through a `global.` inference profile are billed at the source
> region's rate). The *structure* of this analysis — where the money goes and which traps
> exist — is identical. Treat the absolute numbers as the right order of magnitude rather
> than an invoice.

---

## The headline

> **Bedrock has no free tier.** There is no perpetual monthly allowance of free tokens.
> Everything you invoke is billed from the first call.

The good news is the amounts are genuinely small. The bad news is that two things in this
architecture can produce disproportionate bills, and neither is the AI:

1. **OpenSearch Serverless** — the Bedrock Knowledge Base console default, ~$700/month
   idle. Avoided entirely by using S3 Vectors.
2. **CloudWatch custom metrics** — $0.30 per metric per month beyond the free 10. This is
   plausibly the largest *recurring* line item in this project. Details below, because it
   surprised me too.

---

## Unit prices

| Service | Unit | Price |
|---|---|---|
| Titan Text Embeddings V2 | per 1M input tokens | $0.02 |
| Claude Haiku 4.5 (Bedrock) | per 1M input tokens | ~$1.00 |
| Claude Haiku 4.5 (Bedrock) | per 1M output tokens | ~$5.00 |
| S3 Vectors storage | per GB-month | $0.06 |
| S3 Vectors PUT | per GB | $0.20 |
| S3 Vectors query | per request, scaled by index size | fractions of a cent |
| S3 standard storage | per GB-month | $0.023 |
| Lambda | per GB-second | $0.0000166667 |
| Lambda requests | per 1M | $0.20 |
| Textract DetectDocumentText | per 1,000 pages | $1.50 |
| CloudWatch custom metric | per metric-month | $0.30 |
| CloudWatch Logs ingestion | per GB | $0.50 |

Claude pricing on Bedrock is set by AWS, not Anthropic, so confirm it on the
[Bedrock pricing page](https://aws.amazon.com/bedrock/pricing/).

---

## Ingesting one 100-page PDF

Assume ~2,500 characters per page → 250,000 characters ≈ **62,500 tokens**
(a useful rule of thumb: **1 token ≈ 4 characters** of English prose).

| Step | Arithmetic | Cost |
|---|---|---|
| Text extraction (pypdf) | runs inside Lambda you already pay for | **$0** |
| Chunking | pure CPU | **$0** |
| Embedding | 62,500 tokens × 1.17 overlap = 73,125 tokens<br>73,125 ÷ 1,000,000 × $0.02 | **$0.0015** |
| Vector PUT | ~210 chunks × (1024 dims × 4 bytes + ~1.2 KB text) ≈ 1.1 MB<br>0.0011 GB × $0.20 | **$0.0002** |
| Lambda | ~20s at 1024 MB = 20 GB-s (free tier: 400,000 GB-s/mo) | **$0** |
| **Total** | | **≈ $0.002** |

**Indexing 100 such documents costs about 20 cents.** Ingestion is not where your money
goes.

If that same PDF were scanned and routed to Textract: 100 pages × $1.50/1,000 =
**$0.15** — roughly **75× the cost** of the digital path. That ratio is the entire
justification for ADR-2's quality gate.

---

## Answering one question

| Step | Arithmetic | Cost |
|---|---|---|
| Embed the question | ~20 tokens ÷ 1M × $0.02 | $0.0000004 |
| Vector search | 1 query against a small index | ~$0.00001 |
| Generation — input | 5 chunks × ~300 tokens + ~200 prompt = ~1,700 tokens<br>1,700 ÷ 1M × $1.00 | $0.0017 |
| Generation — output | ~150 tokens ÷ 1M × $5.00 | $0.00075 |
| Lambda | 512 MB × 3s = 1.5 GB-s (free tier) | $0 |
| **Total** | | **≈ $0.0025** |

**About 400 questions per dollar.** Roughly 70% of that is generation *input* tokens —
which is precisely the retrieved context. This is why chunk size and `top_k` are cost
levers, not just quality levers: doubling `top_k` roughly doubles your per-question cost.

A **refused** question (nothing below the distance threshold) costs only the embedding
and the search — about **$0.00001**, or 250× less, because the LLM is never invoked.

---

## Monthly total for realistic learning usage

50 documents ingested, 500 questions asked:

| Line item | Cost |
|---|---|
| Ingestion (50 × $0.002) | $0.10 |
| Queries (500 × $0.0025) | $1.25 |
| S3 Vectors storage (~55 MB) | $0.003 |
| S3 document storage (~50 MB) | $0.001 |
| Lambda | $0 (free tier) |
| CloudWatch Logs (~200 MB) | $0 (5 GB free) |
| **CloudWatch custom metrics** | **$1.50 – $2.50** |
| **Total** | **≈ $3 – $4/month** |

### The custom-metrics surprise

CloudWatch gives you **10 custom metrics free**. A "metric" is one metric *name* per
unique *dimension combination*. This project emits roughly 15–18:

- `Service=query`: `EmbedLatencyMs`, `RetrieveLatencyMs`, `GenerateLatencyMs`,
  `EmbeddingTokens`, `InputTokens`, `OutputTokens`, `ChunksRetrieved`,
  `TopCosineDistance`, `AnsweredFromContext`, `RefusedNoContext` — **10**
- `Service=ingest, Extractor=pypdf`: `EmbedLatencyMs`, `ChunksIndexed`,
  `PagesExtracted`, `EmbeddingTokens` — **4**
- `Service=ingest, Extractor=textract`: the same four again — **4 more**
- `Service=ingest`: `TextractFallbacks` — **1**

So 5–13 metrics beyond the free tier, at $0.30 each: **$1.50–$3.90/month**.

**Observability plausibly costs more than the AI.** That is not a bug in the arithmetic —
it is a real and frequently-missed property of AWS pricing, and worth internalising. If
you want to cut it:

- Drop the `Extractor` dimension (it halves the ingest metric count).
- Emit latency metrics only from the query path.
- Or accept it: ~$2/month buys you a dashboard that makes the system debuggable.

Alarms are free up to 10 (we use 4). Dashboards are free up to 3 (we use 1).

---

## What the AWS Free Tier actually covers

Two different things wear the same name:

**Always free (no expiry):**

| Service | Allowance |
|---|---|
| Lambda | 1M requests + 400,000 GB-seconds/month |
| CloudWatch | 10 custom metrics, 10 alarms, 5 GB log ingestion, 3 dashboards |
| SNS | 1M publishes |
| SQS | 1M requests |

**12 months from account creation, then billed:**

| Service | Allowance |
|---|---|
| S3 | 5 GB standard storage, 20k GET, 2k PUT |

**3 months only:**

| Service | Allowance |
|---|---|
| Textract | 1,000 pages/month of DetectDocumentText |

**Never free:**

| Service |
|---|
| **Bedrock** — all model invocations |
| **S3 Vectors** — storage, PUT and query |

Lambda's allowance is the one that genuinely matters here: at 1.5 GB-seconds per query,
400,000 GB-seconds covers roughly **266,000 questions per month** at zero compute cost.
You will not exhaust it.

---

## How you could actually get a large bill

Ranked by how often it happens to real people:

1. **OpenSearch Serverless, ~$700/month.** The Bedrock Knowledge Base console wizard
   selects it by default. It bills a 4-OCU minimum *continuously, whether or not you
   query it*. This project never creates it — but if you experiment with Knowledge Bases
   in the console, read the vector-store step carefully.

2. **A recursive S3 → Lambda loop.** A function triggered by a bucket that writes back to
   the same bucket invokes itself forever. Each iteration calls Bedrock. This template
   prevents it with a prefix filter (`documents/`) plus a suffix filter (`.pdf`), while
   manifests are written to `_manifests/`. AWS now auto-detects and halts some recursive
   loops, but do not rely on that.

3. **Log groups that never expire.** CloudWatch Logs defaults to infinite retention.
   Logs accumulate silently for years at $0.03/GB-month storage on top of $0.50/GB
   ingestion. Every log group here sets `RetentionInDays` explicitly.

4. **High-cardinality metric dimensions.** Putting a document ID or user ID into an EMF
   *dimension* creates one billed metric per unique value. A thousand documents becomes a
   thousand metrics: $300/month. Identifiers belong in the log *body*, where they are
   free and still queryable. See `observability.py`.

5. **A load test with no concurrency cap.** 1,000 concurrent Lambdas each calling Bedrock
   is legitimate traffic as far as AWS is concerned. `ReservedConcurrentExecutions: 2` on
   the query function is the circuit breaker.

6. **Abandoned multipart uploads.** Invisible in the console's object list, billed
   indefinitely. The lifecycle rule aborts them after one day.

---

## Guardrails already in the template

| Guardrail | Where | Protects against |
|---|---|---|
| AWS Budget at $5, alerts at 80% actual / 100% forecast | `Budget` | Everything — the backstop |
| `ReservedConcurrentExecutions: 2` (query) | `QueryFunction` | Runaway invocation |
| `ReservedConcurrentExecutions: 5` (ingest) | `IngestFunction` | Bulk-upload spikes |
| `RetentionInDays: 7` | all log groups | Infinite log retention |
| Prefix + suffix S3 event filters | `IngestFunction` | Recursive trigger loops |
| `AbortIncompleteMultipartUpload` | `DocsBucket` | Invisible storage charges |
| Distance gate short-circuits the LLM | `query/app.py` | Paying to hallucinate |
| `MAX_CHUNKS_PER_DOCUMENT = 2000` | `config.py` | One pathological upload |
| `AuthType: AWS_IAM` | `QueryFunction` | Strangers spending your Bedrock budget |

**The budget only exists if you set `AlarmEmail`.** A budget with no subscriber is not a
guardrail, so the template skips creating it entirely rather than giving you false
comfort.

---

## Teardown

The only reliable way to stop spending is to delete the stack.

```bash
python scripts/teardown.py --stack serverless-rag   # empties both buckets
sam delete --stack-name serverless-rag
```

Order matters. CloudFormation cannot delete a non-empty S3 bucket, and S3 Vectors cannot
delete a vector bucket containing vectors. A stack stuck in `DELETE_FAILED` keeps
billing.

Verify:

```bash
aws cloudformation describe-stacks --stack-name serverless-rag   # should error
aws s3vectors list-vector-buckets --region ap-south-1
aws s3 ls | grep serverless-rag
aws logs describe-log-groups --log-group-name-prefix /aws/lambda/serverless-rag
aws budgets describe-budgets --account-id $(aws sts get-caller-identity --query Account --output text)
```

Log groups declared in the template are deleted with the stack. But note: if a Lambda is
ever invoked *before* CloudFormation creates its log group, Lambda creates one itself —
and that one is not stack-managed and survives `sam delete`. The last command above is
how you catch it.

---

## The one habit worth building

Check **Billing → Cost Explorer → group by Service** once a week while you're
experimenting. Not because this project will surprise you, but because the instinct to
look is what separates people who learn AWS cheaply from people who learn it expensively.
