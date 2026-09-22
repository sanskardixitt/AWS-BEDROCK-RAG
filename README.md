# Serverless RAG on AWS

A complete, deployable Retrieval-Augmented Generation system built from AWS serverless
primitives. Upload a PDF to S3; ask questions about it over HTTPS; get answers with page
citations. Everything is defined in one SAM template and torn down with one command.

```
                 ┌────────────────┐
  you ──upload──►│  S3 (documents)│
                 └───────┬────────┘
                         │ ObjectCreated event
                         ▼
                 ┌────────────────────────┐        poor text
                 │  Ingest Lambda         ├──────────────────► Textract (async)
                 │  pypdf → quality gate  │                          │
                 │  → chunk → embed       │◄── SNS ── Callback ◄─────┘
                 └───────┬────────────────┘        Lambda
                         │ PutVectors
                         ▼
                 ┌────────────────────────┐
                 │  S3 Vectors index      │  ← 1024-dim, cosine
                 └───────┬────────────────┘
                         │ QueryVectors
                         ▼
  you ──ask───►  ┌────────────────────────┐ ──► Bedrock Titan  (embed question)
   (SigV4)       │  Query Lambda          │ ──► Bedrock Claude (generate answer)
                 └───────┬────────────────┘
                         ▼
                 CloudWatch: logs · EMF metrics · alarms · dashboard
```

## Read the docs in this order

> **New here? Start with [GUIDE.md](GUIDE.md)** — a single Hinglish walkthrough covering
> setup, deploying, ingesting your own PDFs, and a line-by-line trace of what actually
> happens inside. Everything below is the deeper reference.

| Doc | What it covers |
|---|---|
| [GUIDE.md](GUIDE.md) | **Hinglish.** Run it, ingest your PDFs, understand the internals |
| [00-aws-setup.md](docs/00-aws-setup.md) | Credentials, IAM permissions, region choice, model access |
| [01-architecture.md](docs/01-architecture.md) | How the system works, component by component, with the data flow |
| [02-design-decisions.md](docs/02-design-decisions.md) | Every significant choice, the alternatives rejected, and why |
| [03-cost-model.md](docs/03-cost-model.md) | What this actually costs, where the traps are, how to tear down |
| [04-production-gaps.md](docs/04-production-gaps.md) | What separates this from a system you'd run for a company |
| [05-interview-questions.md](docs/05-interview-questions.md) | ~50 questions with answers, from fundamentals to system design |
| [06-industrial-rag-system-design.md](docs/06-industrial-rag-system-design.md) |  What real production RAG looks like — all 6 planes, maturity ladder, interview framework |
| [tests/eval/README.md](tests/eval/README.md) |  The eval harness — metrics, what order to read them in, how to tune |

## Before you deploy: three things that will bite you

**1. Bedrock has no free tier.** Nothing in this project is covered by the AWS Free
Tier's Bedrock allowance, because there isn't one. The cost is genuinely tiny (a
100-page PDF costs about $0.0004 to index) but it is not zero. The template creates an
AWS Budget so you find out from an email, not an invoice. Set `AlarmEmail` — the budget
is skipped entirely if you leave it blank.

**2. You must manually enable model access.** A brand-new AWS account cannot call any
Bedrock model until you request access in the console. This is a one-time click, and
its absence produces `AccessDeniedException` that looks like an IAM problem but isn't.

**3. Never point Bedrock Knowledge Bases at OpenSearch Serverless on a learning
account.** That is the console default, and it bills roughly **$700/month** for idle
capacity. This project uses S3 Vectors instead, which bills per GB stored and per query.
See [03-cost-model.md](docs/03-cost-model.md).

## Prerequisites

| Tool | Check | Install (Windows) |
|---|---|---|
| AWS CLI v2 | `aws --version` | `winget install --id Amazon.AWSCLI` |
| AWS SAM CLI | `sam --version` | `winget install --id Amazon.SAM-CLI` |
| Python 3.12+ | `python --version` | [python.org](https://www.python.org/downloads/) |
| Credentials | `aws sts get-caller-identity --profile argus` | already configured |

**Neither the AWS CLI nor SAM is installed on this machine yet** — install both first.
Your AWS credentials *are* already configured: profile `argus` assumes the `ArgusAdmin`
role with MFA. That's the right credential to deploy with, and you don't need a new key
or role. Full explanation in [00-aws-setup.md](docs/00-aws-setup.md).

Deployment is configured for **ap-south-1 (Mumbai)** to match your profile.

## Setup

### Step 0 — Enable Bedrock model access (one time, in the console)

1. Open the [Bedrock console](https://console.aws.amazon.com/bedrock/) — confirm the
   region selector says **Asia Pacific (Mumbai) ap-south-1**.
2. Left nav → **Model access** → **Modify model access**.
3. Enable **Amazon → Titan Text Embeddings V2** and **Anthropic → Claude Haiku 4.5**.
4. Submit. Usually granted instantly.

Verify from your terminal:

```bash
aws bedrock list-foundation-models --region ap-south-1 --profile argus \
  --query "modelSummaries[?contains(modelId,'titan-embed-text-v2')].modelId"

aws bedrock list-inference-profiles --region ap-south-1 --profile argus \
  --query "inferenceProfileSummaries[?contains(inferenceProfileId,'haiku-4-5')].inferenceProfileId"
```

The second command must show `global.anthropic.claude-haiku-4-5-20251001-v1:0`.

> **Region gotcha:** from Mumbai the `us.` prefix does **not** work — the US geo profile
> rejects ap-south-1 as a source region, and no APAC geo profile exists for Haiku 4.5.
> Only the `global.` profile works. That is now the default. If the command above returns
> a different ID, use that one — trust it over any hardcoded string.

### Step 1 — Build and deploy

```bash
export AWS_PROFILE=argus       # PowerShell: $env:AWS_PROFILE = "argus"

sam build
sam deploy --guided            # first time only; writes your answers to samconfig.toml
```

When prompted, set `AlarmEmail` to your address — the budget resource is skipped entirely
if you leave it blank. Confirm the SNS subscription email that arrives, otherwise alarms
fire silently into a topic nobody is listening to.

Answer `y` to "Allow SAM CLI IAM role creation" — that is SAM building the Lambda
execution roles from the `Policies:` blocks in the template.

Later deploys are just `sam build && sam deploy`.

### Step 2 — Ingest a document

```bash
python scripts/make_sample_pdf.py samples/handbook.pdf   # already generated
python scripts/upload.py samples/handbook.pdf
```

Watch ingestion happen:

```bash
sam logs --stack-name serverless-rag --name IngestFunction --tail
```

You should see `pypdf_extracted` → `chunked` → `vectors_written` → `ingest_complete`.
Then confirm what is indexed:

```bash
python scripts/upload.py --list
```

### Step 3 — Ask questions

```bash
python scripts/ask.py "How long is the refund window?"
python scripts/ask.py "What is the uptime target and what credit applies below 95%?"
python scripts/ask.py "Who is the CEO?"        # should refuse — not in the document
```

That last one is the important test. A RAG system that invents a CEO is worse than one
that says "I don't know", and most tutorial implementations invent one.

### Step 4 — Look at the telemetry

```bash
aws cloudformation describe-stacks --stack-name serverless-rag --region ap-south-1 \
  --query "Stacks[0].Outputs[?OutputKey=='DashboardUrl'].OutputValue" --output text
```

The dashboard shows latency split across embed / retrieve / generate, token
consumption, retrieval quality, and how often the system refused to answer.

## Teardown — do this when you stop experimenting

```bash
# Vector buckets must be EMPTY before they can be deleted, and the docs bucket is
# versioned, so both need clearing first.
python scripts/teardown.py --stack serverless-rag

sam delete --stack-name serverless-rag
```

Verify nothing survived:

```bash
aws cloudformation describe-stacks --stack-name serverless-rag --region ap-south-1  # should error
aws s3vectors list-vector-buckets --region ap-south-1
aws s3 ls | grep serverless-rag
```

## Repository layout

```
template.yaml              All infrastructure. Start reading here.
samconfig.toml             Deployment defaults.
src/
  common/
    config.py              Every tunable, read from env vars, fails fast.
    chunking.py            Paragraph-aware chunking with overlap + page attribution.
    bedrock.py             Titan embeddings, Claude generation via Converse API.
    vectors.py             S3 Vectors read/write + the manifest that makes re-ingest safe.
    extraction.py          pypdf, the OCR quality gate, and the Textract async path.
    pipeline.py            Shared chunk→embed→index pipeline both ingest paths use.
    observability.py       Structured JSON logging + EMF custom metrics.
  ingest/app.py            S3-triggered. Digital PDFs end-to-end; scanned ones dispatched.
  textract_callback/app.py SNS-triggered. Finishes scanned PDFs after OCR.
  query/app.py             Function URL. Retrieve → augment → generate.
scripts/
  make_sample_pdf.py       Stdlib-only PDF generator with known facts to test against.
  upload.py                Upload PDFs / list what's indexed.
  ask.py                   SigV4-signed query client.
  teardown.py              Empty the buckets so sam delete succeeds.
tests/test_chunking.py     Offline tests. No AWS account needed.
docs/                      The system design write-ups.
```

## Testing

### Offline — no AWS, no network, free

```bash
python tests/test_chunking.py          # 11 tests: chunking logic
python tests/eval/test_metrics.py      # 21 tests: the eval scoring logic itself
```

Chunking is pure logic, so it should be tested as pure logic. Finding an off-by-one here
is free; finding it after embedding 4,000 chunks is not. The second file tests the
*evaluator* — buggy scoring is worse than no scoring, because it gives you confident
numbers that are wrong.

### Against the deployed stack — the eval harness

```bash
python tests/eval/run_eval.py --dry-run     # cost estimate, calls nothing
python tests/eval/run_eval.py --no-judge    # deterministic metrics only, ~free
python tests/eval/run_eval.py               # full run, ~$0.15
```

30 golden questions (22 answerable, 8 deliberately unanswerable) scored on:

| Layer | Metrics |
|---|---|
| **Retrieval** | Recall@k, MRR — the ceiling on everything downstream |
| **Generation** | keyword hit (free), correctness + faithfulness (LLM judge) |
| **Hallucination guard** | answered-when-answerable vs refused-when-unanswerable |

Tune a knob and compare against the previous run:

```bash
python tests/eval/run_eval.py --top-k 3 --compare tests/eval/results/<previous>.json
```

Use it as a CI gate:

```bash
python tests/eval/run_eval.py --fail-under-recall 0.85 --fail-under-refusal 0.90
```

Full explanation, including why the metrics must be read in a specific order:
[tests/eval/README.md](tests/eval/README.md).
