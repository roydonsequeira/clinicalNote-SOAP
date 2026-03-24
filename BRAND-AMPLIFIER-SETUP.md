# AI Personal Brand Amplifier — Setup Guide

## Architecture Overview

```
Sources (HN / GitHub / RSS)
  └─> Normalize
        └─> Embed (nomic-embed-text)
              └─> Deduplicate (Qdrant)
                    └─> Scout Agent (llama3.2:3b) → IGNORE or POST
                          └─> Strategist Agent (mistral:7b-instruct)
                                └─> Formatter
                                      └─> Telegram Approval
                                            └─> LinkedIn + X
                                                  └─> Feedback → Qdrant
```

---

## Step 1 — Install & Start Ollama

```bash
# Install Ollama (https://ollama.ai)
# Then pull the required models:
ollama pull nomic-embed-text    # Embedding model (768 dims)
ollama pull llama3.2:3b         # Scout Agent (fast filter)
ollama pull mistral:7b-instruct # Strategist Agent (deep insight)

# Verify Ollama is running
curl http://localhost:11434/api/tags
```

---

## Step 2 — Install & Start Qdrant

```bash
# Using Docker (recommended)
docker pull qdrant/qdrant
docker run -d -p 6333:6333 -p 6334:6334 \
  -v $(pwd)/qdrant_storage:/qdrant/storage \
  --name qdrant \
  qdrant/qdrant

# Verify Qdrant is running
curl http://localhost:6333/health
```

### Create Required Collections

```bash
# Collection for deduplicating seen content (768-dim = nomic-embed-text)
curl -X PUT http://localhost:6333/collections/content_memory \
  -H 'Content-Type: application/json' \
  -d '{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    }
  }'

# Collection for storing pending (awaiting approval) posts
curl -X PUT http://localhost:6333/collections/pending_posts \
  -H 'Content-Type: application/json' \
  -d '{
    "vectors": {
      "size": 768,
      "distance": "Cosine"
    },
    "payload_schema": {
      "postId": { "data_type": "keyword" },
      "status": { "data_type": "keyword" }
    }
  }'

# Create payload index for fast postId lookups
curl -X PUT http://localhost:6333/collections/pending_posts/index \
  -H 'Content-Type: application/json' \
  -d '{ "field_name": "postId", "field_schema": "keyword" }'
```

---

## Step 3 — Create a Telegram Bot

1. Message `@BotFather` on Telegram
2. Run `/newbot` and follow prompts
3. Copy the **Bot Token** (looks like `123456789:ABCdef...`)
4. Get your **Chat ID**:
   - Start a chat with your bot
   - Visit: `https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates`
   - Send any message to the bot, then refresh the URL
   - Find `"chat":{"id":XXXXXXXX}` — that number is your Chat ID

---

## Step 4 — Configure n8n

### Install n8n
```bash
npm install -g n8n
n8n start
# Open http://localhost:5678
```

### Add Telegram Credential
1. Go to **Settings → Credentials → Add Credential**
2. Choose **Telegram**
3. Paste your Bot Token
4. Note the credential ID (shown in the URL after saving)

### Set Environment Variable
In n8n Settings → Environment Variables:
```
TELEGRAM_BOT_TOKEN = 123456789:ABCdef...
```

---

## Step 5 — Import the Workflows

1. In n8n, go to **Workflows → Import from File**
2. Import `n8n-workflow-main.json` → **Main Pipeline**
3. Import `n8n-workflow-approval.json` → **Approval Handler**

### Configure Placeholders in Main Pipeline

Open `n8n-workflow-main.json` and replace:

| Placeholder | Replace With |
|---|---|
| `YOUR_TELEGRAM_CHAT_ID` | Your Telegram numeric Chat ID |
| `TELEGRAM_CREDENTIAL_ID` | ID of your n8n Telegram credential |

Node: **Send Telegram Approval**

### Configure Placeholders in Approval Handler

| Placeholder | Replace With |
|---|---|
| `YOUR_TELEGRAM_CHAT_ID` | Your Chat ID |
| `TELEGRAM_CREDENTIAL_ID` | Same credential ID |
| `YOUR_LINKEDIN_ACCESS_TOKEN` | LinkedIn OAuth2 access token |
| `YOUR_LINKEDIN_PERSON_URN` | Your LinkedIn URN (e.g. `ABC123`) |
| `YOUR_OAUTH_1A_HEADER` | Twitter/X OAuth 1.0a header string |
| `YOUR_WEBHOOK_ID` | Auto-generated when you activate the workflow |

---

## Step 6 — LinkedIn API Setup

1. Go to [LinkedIn Developer Portal](https://developer.linkedin.com)
2. Create an app → add **Sign In with LinkedIn** and **Share on LinkedIn** products
3. Generate an OAuth 2.0 access token with scope `w_member_social`
4. Get your Person URN:

```bash
curl -H "Authorization: Bearer YOUR_ACCESS_TOKEN" \
  https://api.linkedin.com/v2/userinfo
# Look for "sub" field — that is your person URN
```

---

## Step 7 — X (Twitter) API Setup

1. Go to [developer.twitter.com](https://developer.twitter.com)
2. Create a project + app with **Read and Write** permissions
3. Generate OAuth 1.0a keys (App + Access token)
4. The `Authorization` header in the Publish to X node must be a valid OAuth 1.0a signed header

> For simplicity, use a library like [oauth-1.0a](https://www.npmjs.com/package/oauth-1.0a) or the [Twitter v2 client](https://github.com/PLhery/node-twitter-api-v2) to generate the signed header, or configure n8n's built-in Twitter node and copy the credential approach.

---

## Step 8 — Activate & Test

### Activate Approval Handler First
- Open `Approval Handler` workflow → toggle **Active** ON
- This registers the Telegram webhook

### Test the Main Pipeline Manually
1. Open `Main Pipeline` workflow
2. Click **Execute Workflow** (manual run)
3. Watch nodes execute one by one
4. If content passes Scout Agent → you'll receive a Telegram message
5. Tap **Approve & Post** → both LinkedIn and X are posted

### Activate Main Pipeline for Scheduled Runs
- Toggle **Active** ON → runs every 6 hours automatically

---

## Customization

### Change Data Sources

**Add more RSS feeds** — duplicate the `Fetch HuggingFace Blog RSS` node:
- The Batch (DeepLearning.AI): `https://www.deeplearning.ai/the-batch/feed/`
- TechCrunch AI: `https://techcrunch.com/category/artificial-intelligence/feed/`
- Import AI Newsletter: `https://importai.substack.com/feed`

**Change HN query** — edit the `Fetch Hacker News` node's `query` parameter.

**Change schedule** — edit `Schedule Trigger` node (default: every 6 hours).

### Tune Scout Agent Sensitivity
In the `Scout Agent` node's body expression, adjust the system prompt:
- More permissive → lower the bar for POST
- More selective → raise it (requires higher specificity)

### Change Similarity Threshold
In the `Evaluate Similarity` Code node:
```javascript
const isDuplicate = maxScore >= 0.85; // Lower = stricter dedup (0.75), Higher = looser (0.92)
```

### Change Post Template
Edit the `Format Post Content` Code node to change the LinkedIn/X format.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `Generate Embedding` fails | Check Ollama is running: `curl localhost:11434/api/tags` |
| `Search Qdrant Similarity` fails with 404 | Collection doesn't exist — run Step 2 curl commands |
| Scout Agent returns empty | Check `llama3.2:3b` is pulled: `ollama list` |
| Telegram message not received | Verify Chat ID and Bot Token are correct |
| LinkedIn 401 error | Access token expired — regenerate it |
| X 403 error | Check app has Write permissions enabled |

---

## Workflow Diagram

```
[Schedule Trigger: every 6h]
        |
   ┌────┴────────────────────┐
   ↓             ↓           ↓
[HN API]   [GitHub API]  [HF RSS]
   └──────────┬─────────────┘
         [Merge All]
              ↓
      [Normalize Items]
              ↓
       [Split 1-by-1]
              ↓
   [Embed: nomic-embed-text]
              ↓
   [Qdrant: search similar]
              ↓
     [score >= 0.85?]
      YES ↓       ↓ NO
    [SKIP]   [Upsert to Qdrant]
                  ↓
         [Scout: llama3.2:3b]
                  ↓
          [verdict = POST?]
        YES ↓          ↓ NO
  [Strategist: mistral]  [SKIP]
              ↓
      [Format post]
              ↓
  [Telegram: Approve/Reject]
              ↓
    [Store in pending_posts]

--- APPROVAL HANDLER ---

[Telegram callback: approve_XXX]
              ↓
    [Fetch post from Qdrant]
              ↓
  [Publish LinkedIn + X (parallel)]
              ↓
  [Confirm via Telegram + update status]
```
