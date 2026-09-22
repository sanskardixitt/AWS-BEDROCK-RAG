# 00 — AWS credentials and permissions setup

## The short answer

**You do not need a new access key, and you do not need to create a new IAM role.**
You already have both, and your setup is better than most people's.

What you actually need to do:

1. Install the AWS CLI and SAM CLI (neither is installed yet)
2. Confirm your `ArgusAdmin` role can create the resources this stack needs
3. Enable Bedrock model access in the console (one-time click)
4. Deploy with `--profile argus`

---

## Why the question is confusing: there are three separate credential layers

This is the part worth internalising, because conflating these is the single most common
source of confusion with serverless IAM. **You only have to configure one of them.**

```
┌──────────────────────────────────────────────────────────────────────┐
│ LAYER 1 — DEPLOY-TIME (you, on your laptop)          ← YOU CONFIGURE │
│                                                                      │
│   Your IAM user's access key  →  assumes ArgusAdmin role  →  MFA     │
│   Used by: sam deploy, aws cli                                       │
│   Needs: permission to CREATE S3 buckets, Lambdas, IAM roles, etc.   │
└──────────────────────────────────────────────────────────────────────┘
                                   │  creates
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│ LAYER 2 — RUNTIME (the Lambda functions)              ← AUTOMATIC    │
│                                                                      │
│   Each Lambda gets its OWN execution role, created by SAM from the   │
│   Policies: blocks in template.yaml. Lambdas never use your keys.    │
│   Needs: permission to CALL Bedrock, read S3, write S3 Vectors       │
│                                                                      │
│   YOU DO NOTHING HERE. It is already written in the template.        │
└──────────────────────────────────────────────────────────────────────┘
                                   │  protects
                                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│ LAYER 3 — CALLING THE API (scripts/ask.py)      ← ALREADY COVERED    │
│                                                                      │
│   The Function URL uses AuthType: AWS_IAM, so requests must be       │
│   SigV4-signed. ask.py signs with whatever profile you pass.         │
│   Needs: lambda:InvokeFunctionUrl                                    │
│                                                                      │
│   ArgusAdmin already has this if it is an admin role.                │
└──────────────────────────────────────────────────────────────────────┘
```

**Layer 2 is the one people expect to have to build by hand, and it's the one that's
already done.** Every `Policies:` block in [template.yaml](../template.yaml) *is* an IAM
role definition. Read them — they're a genuinely good IAM tutorial, because each one is
scoped to exactly what that function does. The ingest function, for example, can call the
embedding model but *not* the chat model.

---

## Your current setup

```
~/.aws/credentials
  [argus-base]     IAM user with a long-lived access key (AKIA...)

~/.aws/config
  [profile argus]  source_profile = argus-base
                   role_arn       = arn:aws:iam::<account>:role/ArgusAdmin
                   mfa_serial     = arn:aws:iam::<account>:mfa/sanskar
                   region         = ap-south-1
```

This is the **role-assumption-with-MFA pattern**, and it's correct practice:

- Your long-lived key (`argus-base`) has minimal or no permissions of its own.
- Real permissions live on a *role* (`ArgusAdmin`) that must be explicitly assumed.
- Assuming it requires an MFA code, so a stolen access key alone is not enough.
- The resulting credentials are **temporary** (1 hour by default) and auto-expire.

**Always use `--profile argus`, never `argus-base`.** `argus-base` is just the key that
lets you assume the role; it likely can't create anything itself.

---

## Step 1 — Install the tooling

Three things are missing on this machine. On Windows:

```powershell
winget install --id Amazon.AWSCLI
winget install --id Amazon.SAM-CLI
winget install --id Python.Python.3.12
```

**Why Python 3.12 specifically, when you already have 3.14:** `sam build` compiles your
function against the *exact* runtime declared in the template (`python3.12`). It looks
for a matching interpreter on your PATH and fails with
`Binary validation failed ... did not satisfy constraints for runtime: python3.12` if it
can't find one. Python 3.14 does not satisfy it, and 3.14 is not a Lambda runtime.

Installing 3.12 side-by-side is harmless — the `py` launcher keeps them separate and 3.14
stays your default for everything else.

**Alternative if you'd rather not install another Python:** Docker is already installed on
this machine, so `sam build --use-container` works instead. SAM pulls an official
Lambda build image and compiles inside it. Slower on first run (it downloads ~1GB) but
more faithful to the real Lambda environment, which is what CI pipelines use.

Close and reopen your terminal, then verify:

```bash
aws --version     # aws-cli/2.x
sam --version     # SAM CLI, version 1.x
py --list         # should now list 3.14 AND 3.12
```

---

## Step 2 — Verify your credentials work

```bash
aws sts get-caller-identity --profile argus
```

You'll be prompted for your MFA code. Expected output:

```json
{
  "Account": "<your-account-id>",
  "Arn": "arn:aws:sts::<account>:assumed-role/ArgusAdmin/botocore-session-..."
}
```

The `assumed-role/ArgusAdmin` part is what you want to see — it confirms the role
assumption worked, rather than falling back to the base user.

> **Tip:** MFA prompts on every command get tedious. The CLI caches the assumed-role
> session in `~/.aws/cli/cache`, so you'll typically be asked once per hour.

---

## Step 3 — Check the role can deploy this stack

If `ArgusAdmin` has `AdministratorAccess`, skip ahead — you're fine.

```bash
aws iam list-attached-role-policies --role-name ArgusAdmin --profile argus
```

If you see `AdministratorAccess`, you're done. Otherwise, the stack needs the deploying
principal to be able to create these:

| Service | Why the stack needs it |
|---|---|
| `cloudformation:*` | SAM deploys through CloudFormation |
| `s3:*` | The documents bucket + SAM's own artifact bucket |
| `s3vectors:*` | The vector bucket and index |
| `lambda:*` | Three functions and a Function URL |
| `iam:CreateRole`, `DeleteRole`, `GetRole`, `PassRole`, `PutRolePolicy`, `DeleteRolePolicy`, `AttachRolePolicy`, `DetachRolePolicy`, `TagRole` | **Lambda execution roles + the Textract publish role** |
| `sqs:*` | The dead-letter queue |
| `sns:*` | Alarm topic + Textract completion topic |
| `cloudwatch:*` | Alarms and the dashboard |
| `logs:*` | Log groups with retention |
| `budgets:*` | The cost guardrail |
| `bedrock:ListFoundationModels`, `ListInferenceProfiles` | Your own verification commands |

The IAM permissions are the ones people forget. **You cannot deploy a stack that creates
IAM roles unless you can create IAM roles** — and that's effectively admin-level power,
because anyone who can create a role can create an admin role. For a personal sandbox
account, `AdministratorAccess` on an assumed, MFA-protected role is the pragmatic and
defensible choice. On a shared or company account it would not be.

---

## Step 4 — Enable Bedrock model access

New AWS accounts cannot invoke *any* Bedrock model until you request access. This is a
console-only, one-time action, and its absence produces an `AccessDeniedException` that
looks like an IAM problem but isn't.

1. Open the [Bedrock console](https://console.aws.amazon.com/bedrock/) — **make sure the
   region selector says Asia Pacific (Mumbai) ap-south-1**
2. Left nav → **Model access** → **Modify model access**
3. Enable:
   - **Amazon → Titan Text Embeddings V2**
   - **Anthropic → Claude Haiku 4.5**
4. Submit. Usually granted instantly.

Verify from the terminal:

```bash
# Embedding model - must be available IN ap-south-1
aws bedrock list-foundation-models --region ap-south-1 --profile argus \
  --query "modelSummaries[?contains(modelId,'titan-embed-text-v2')].modelId"

# Chat model - must show a global.* profile
aws bedrock list-inference-profiles --region ap-south-1 --profile argus \
  --query "inferenceProfileSummaries[?contains(inferenceProfileId,'haiku-4-5')].inferenceProfileId"
```

The second command should include
`global.anthropic.claude-haiku-4-5-20251001-v1:0`. If it returns a different ID, use
that one as `ChatModelId` — **trust this command over any hardcoded string**, including
the default in `samconfig.toml`.

---

## Step 5 — The region question

Your profile is `ap-south-1` (Mumbai), so that's what `samconfig.toml` is now set to.
Verified support there:

| Requirement | ap-south-1 (Mumbai) | us-east-1 (N. Virginia) |
|---|---|---|
| S3 Vectors | ✅ | ✅ |
| Titan Embeddings V2 (in-region) | ✅ | ✅ |
| Claude Haiku 4.5 in-region | ❌ | ❌ |
| Claude Haiku 4.5 geo profile | ❌ **none exists for APAC** | ✅ `us.` |
| Claude Haiku 4.5 global profile | ✅ `global.` | ✅ `global.` |
| Textract | ✅ | ✅ |

**This is why the model ID matters.** From Mumbai, `us.anthropic.claude-haiku-4-5-...`
fails — the US geo profile does not accept ap-south-1 as a source region. Only
`global.anthropic.claude-haiku-4-5-20251001-v1:0` works, which is now the default.

One trade-off to be aware of: a **global** profile can route your request to any
commercial AWS region worldwide, so it gives no data-residency guarantee. For a learning
project that's irrelevant. If you ever needed Indian data residency for the generation
step, no Claude option currently satisfies it from Mumbai — you'd deploy in a region with
a geo profile instead.

---

## Step 6 — Deploy

```bash
cd C:\Users\sanskar\Desktop\aws-bedrock

sam build

# --guided the first time; it confirms region, profile and parameters,
# then writes your answers back to samconfig.toml
sam deploy --guided --profile argus
```

When prompted:

| Prompt | Answer |
|---|---|
| Stack Name | `serverless-rag` |
| AWS Region | `ap-south-1` |
| `AlarmEmail` | **your email** — the budget resource is skipped if blank |
| `ChatModelId` | `global.anthropic.claude-haiku-4-5-20251001-v1:0` |
| Confirm changes before deploy | `y` — always read the changeset |
| Allow SAM CLI IAM role creation | `y` — this is Layer 2 being built for you |
| Disable rollback | `N` — you want automatic rollback on failure |
| Save arguments to configuration file | `y` |

Subsequent deploys are just:

```bash
sam build && sam deploy
```

Confirm the SNS subscription email that arrives, or alarms will fire silently into a
topic nobody is listening to.

---

## Step 7 — Use it

Every script takes the same flags:

```bash
python scripts/upload.py samples/handbook.pdf --stack serverless-rag --region ap-south-1
python scripts/ask.py "How long is the refund window?" --stack serverless-rag --region ap-south-1
```

To avoid repeating the profile, set it for the session:

```bash
# Git Bash
export AWS_PROFILE=argus

# PowerShell
$env:AWS_PROFILE = "argus"
```

---

## Troubleshooting

| Error | Cause | Fix |
|---|---|---|
| `Unable to locate credentials` | No profile specified | `export AWS_PROFILE=argus` or pass `--profile argus` |
| `AccessDeniedException` on `bedrock:InvokeModel` | Model access not enabled in the console | Step 4 — and check you did it in **ap-south-1** |
| `AccessDeniedException` naming a `foundation-model` ARN in an unexpected region | Inference profile routing to a region your IAM policy doesn't cover | Already handled — the template wildcards the region |
| `ValidationException: invocation of model ID ... isn't supported` | Using a bare model ID or the wrong geo prefix | Use the `global.` profile ID |
| `UnknownServiceError: s3vectors` | boto3 too old | Already handled — `requirements.txt` pins a recent boto3 |
| `User is not authorized to perform iam:CreateRole` | The deploy role lacks IAM permissions | Step 3 |
| MFA prompt on every command | Normal | Cached ~1 hour in `~/.aws/cli/cache` |
| `Stack ... does not exist` from a script | Wrong region | Pass `--region ap-south-1` |

---

## Security notes on what you already have

Worth stating because your setup mostly gets these right:

- ✅ **Not using root.** Root access keys should never exist; if any do, delete them.
- ✅ **MFA on role assumption.** A stolen access key alone cannot assume `ArgusAdmin`.
- ✅ **Temporary credentials at the point of use.** The role session expires in an hour.
- ⚠️ **`argus-base` still holds a long-lived key.** That's unavoidable with this pattern,
  but rotate it periodically, and confirm it has no permissions beyond `sts:AssumeRole`.
- ⚠️ **`~/.aws/credentials` is plaintext on disk.** Never copy it into a repo, a chat
  message, or a screenshot. `.gitignore` in this project does not protect files outside
  the project.
- 💡 **The upgrade path is IAM Identity Center** (`aws configure sso`), which removes the
  long-lived key entirely in favour of short-lived SSO sessions. Worth doing eventually;
  not worth blocking on today.
