# n8n AI Workflows

Two working n8n workflows that combine webhooks, website scraping, an LLM and structured output. Both are self-built demo projects that I run on my own self-hosted n8n instance. Import them, add your own credentials and they run.

---

## 1. AI Lead Scoring → Google Sheets

![Workflow](lead-scoring-canvas.png)

An inbound lead arrives by webhook. n8n reads the company's website, an LLM scores how well the lead fits the seller's offer, every lead is logged to Google Sheets, and high-fit leads are separated from the rest.

**Flow**

1. **Webhook** receives `Lead Name`, `Email`, `Company Website URL`, `Stated Challenge`
2. **HTTP Request** fetches the company website (adds `https://` if missing, follows redirects, retries, and keeps running if the site is down)
3. **Code** strips scripts, styles and HTML down to about 5,000 characters of plain text
4. **OpenAI (GPT-4o-mini)** returns JSON: company, business model, fit score (1–10), pain points, a personalised opening line, and a recommended next step
5. **Code** parses the JSON and flags the lead for manual review if the AI answer can't be read, instead of writing made-up values
6. **Google Sheets** gets one row per lead
7. **IF** fit score ≥ 7 → sales briefing (Slack-formatted text). Below 7 → qualification email draft

![Results in Google Sheets](lead-scoring-sheet.png)

**Design choices**

- The fit score is measured against a **SELLER CONTEXT** block in the prompt, so the score means "fit for *this* business", not a generic guess. Edit it for your offer.
- The prompt includes a scoring guide (8–10 / 5–7 / 1–4) so scores stay consistent between leads.
- Failures stay visible: an unreachable website or unreadable AI output is marked as such and never hidden behind default text.

**Test request**

```bash
curl -X POST "https://YOUR-N8N-DOMAIN/webhook-test/lead-scoring-demo" \
  -H "Content-Type: application/json" \
  -d '{"Lead Name":"Lies Van den Broeck","Email":"lies.vandenbroeck@example.com","Company Website URL":"lotusbakeries.com","Stated Challenge":"We get many wholesale inquiries and struggle to see which ones are worth a call."}'
```

**Google Sheet columns (row 1):**
`Timestamp | Lead Name | Email | Company | Fit Score | Business Model | Pain Points | Icebreaker | Recommended Action`

---

## 2. AI Support Inquiry Router

A customer message arrives by webhook. The LLM classifies it and writes a draft reply, and the workflow routes it as urgent or standard.

**Flow**

1. **Webhook** receives `{"message": "..."}` (a Tally form submission also works)
2. **OpenAI (GPT-4o-mini)** returns JSON: category (Billing / Technical / General Inquiry), sentiment, priority, a one-line summary and a draft response
3. **Switch** sends high priority *or* frustrated customers to the urgent branch, and everything else to standard
4. **Code** prepares the alert or reply data for each branch

The end nodes prepare the data but don't send anything. Connect a Slack, Gmail or helpdesk node to deliver it.

**Test request**

```bash
curl -X POST "https://YOUR-N8N-DOMAIN/webhook-test/support-inquiry-demo" \
  -H "Content-Type: application/json" \
  -d '{"message":"I was charged twice this month and nobody has answered my last two emails."}'
```

---

## Setup

1. In n8n: **Workflows → Import from File** and choose the `.json` file
2. Open the OpenAI node and select or create your **OpenAI** credential
3. Lead scoring only: select your **Google Sheets** credential, then pick your sheet in the "Append row in sheet" node
4. Lead scoring only: edit **SELLER CONTEXT** in the system prompt of the "Message a model" node
5. Click **Execute workflow** and send one of the test requests above

No credentials, API keys or personal data are included in these files.

---

Built by Ebrahim · n8n, API integrations and AI automation
