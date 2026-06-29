# Outbound Prospecting Pipeline

An automated outbound prospecting pipeline built in n8n that enriches prospects, scores fit against an ICP, drafts personalised emails using Claude AI, and routes everything to a human review queue — with full error handling and Slack observability.

Built as a working demonstration of GTM automation engineering.

---

<img width="1822" height="592" alt="image" src="https://github.com/user-attachments/assets/b5003c34-0723-422e-bd9c-e23a7d7c5ffe" />


## What It Does

1. **Fetches unprocessed prospects** from Airtable on a schedule (9AM weekdays) or via manual trigger
2. **Watermark-based deduplication** — only processes records not yet run, using a `Last Processed` timestamp cursor to prevent reprocessing
3. **Enriches each prospect** with industry-specific tech stack, pain points, and buyer context (mock Apollo enrichment — swap node for live Apollo API in production)
4. **Scores fit deterministically** against the ICP using employee count, industry, revenue, and contact title — no AI involved in scoring, keeps it auditable and consistent
5. **Filters low fit prospects** — logged to queue with `Low Fit - Skipped` status, watermark updated, no AI credits spent
6. **Drafts personalised outbound emails** using Claude Haiku via Anthropic API — subject line, email body, and a personalization hook per prospect
7. **Saves drafts to Airtable** Outbound Queue with `Draft - Pending Review` status for human review before any send
8. **Updates watermark** on the Prospects table so records aren't reprocessed next run
9. **Sends a Slack batch summary** on every run — total processed, drafts ready, low fit skipped, and any errors detected
10. **Dead letter queue** — all failures (enrichment, AI draft, queue write) are logged to a Pipeline Errors table with error message, node name, and timestamp

---

## Architecture

```
[Trigger] → [Fetch Prospects] → [Watermark Filter] → [Mock Enrichment]
         → [Enrichment Check] → [Fit Scoring] → [Low Fit Filter]
         → [Claude Haiku Draft] → [Parse Response] → [AI Draft Check]
         → [Save to Outbound Queue] → [Update Watermark] → [Batch Summary] → [Slack]

Error paths at every node → [Pipeline Errors Table] + [Slack Alert]
```

**Design decisions:**
- Fit scoring is **deterministic code**, not AI — scores are consistent, explainable, and free
- AI is used only for qualitative reasoning (email copy, personalization hook) where variability adds value
- Human-in-the-loop by design — no emails are sent automatically, all drafts require manual review
- Watermark pattern preferred over status flags — more resilient to partial failures

---

## Airtable Setup

Create one base with three tables:

### Prospects
| Field | Type |
|---|---|
| Company Name | Single line text |
| Industry | Single line text |
| Employee Count | Number |
| Annual Revenue | Single line text |
| Contact Name | Single line text |
| Contact Title | Single line text |
| Email | Email |
| Date Added | Date |
| Last Processed | Date |
| Enrichment Status | Single select: `Pending`, `Success`, `Failed` |

### Outbound Queue
| Field | Type |
|---|---|
| Company Name | Single line text |
| Contact Name | Single line text |
| Contact Title | Single line text |
| Email | Single line text |
| Fit Score | Number |
| Fit Tier | Single select: `High`, `Medium`, `Low` |
| Fit Reasoning | Long text |
| Email Subject | Single line text |
| Email Draft | Long text |
| Personalization Hook | Long text |
| Prospect Record ID | Single line text |
| Status | Single select: `Draft - Pending Review`, `Low Fit - Skipped`, `AI Draft Failed`, `Enrichment Failed` |
| Processed At | Date |

### Pipeline Errors
| Field | Type |
|---|---|
| Company Name | Single line text |
| Record ID | Single line text |
| Failed At | Single select: `Enrichment`, `AI Draft`, `Queue Write`, `Watermark` |
| Error Message | Long text |
| Timestamp | Date |

---

## Credentials Required

| Credential | Type in n8n | Details |
|---|---|---|
| Airtable | Personal Access Token | Create at airtable.com/create/tokens — scope to this base |
| Anthropic | Header Auth | Header name: `x-api-key`, value: your Anthropic API key |
| Slack | Slack OAuth | Existing Slack app with `chat:write` scope |

---

## How to Import

1. Clone this repo
2. In n8n: **Workflows → Import from file** → select `outbound_pipeline.json`
3. Connect credentials (Airtable, Anthropic, Slack) to the relevant nodes
4. Update Slack channel ID in the three Slack nodes
5. Add prospect records to Airtable with `Enrichment Status: Pending`
6. Run via **Execute Workflow** button for testing, or activate for scheduled runs

---

## Sample Data

`prospects.csv` contains 6 sample records across Logistics, Manufacturing, Retail, Healthcare, Finance, and Food & Beverage (intentional low-fit record to test the filter path).

Import via: Airtable → Grid view → **+ Add or import → CSV**

---

## Fit Scoring Logic

| Signal | Criteria | Points |
|---|---|---|
| Employee Count | 500+ | 3 |
| | 100–499 | 2 |
| | 50–99 | 1 |
| | <50 | 0 |
| Industry | Logistics, Manufacturing, Retail, Healthcare | 3 |
| | Finance, Construction, Energy | 2 |
| | Other | 1 |
| Annual Revenue | $100M+ | 3 |
| | $10M–$99M | 2 |
| | Unknown/Low | 1 |
| Contact Title | CFO, VP Finance | 1 |
| | AP Manager, Controller | 1 |

**Tiers:** High ≥ 8 · Medium 5–7 · Low < 5

---

## Production Upgrade Path

| Current (Demo) | Production Replacement |
|---|---|
| Mock Apollo enrichment | Apollo.io API HTTP Request node |
| Static prospect CSV | CRM sync via HubSpot/Salesforce node |
| Manual review in Airtable | Reply.io or Instantly sequence trigger |
| Single Slack channel | Separate `#gtm-errors` channel for error alerts |

---

## Stack

- **n8n** — workflow orchestration
- **Airtable** — prospect source, outbound queue, error log
- **Anthropic Claude Haiku** — email drafting via REST API
- **Slack** — run summaries and error alerts
