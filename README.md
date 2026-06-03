# Caregiver Call Intelligence

AI-powered call analysis for dementia care coordination. Analyzes caregiver-navigator phone call transcripts, extracts structured risk data, and writes it to HubSpot — giving care teams real-time visibility into patient safety without manual review.

## The Problem

Care navigators at dementia-focused health organizations handle hundreds of caregiver calls weekly. Each call contains critical signals — falls, medication issues, cognitive decline, caregiver burnout — that determine whether a patient needs immediate intervention or routine follow-up. No navigator can review every call in detail, and critical signals get missed in volume.

This workflow automates the analysis layer. Raw transcripts go in; structured, actionable clinical intelligence comes out, logged directly in the CRM where care teams already work.

## Architecture

```
┌─────────────┐     ┌───────────────────────────┐     ┌─────────────────┐
│  Transcript  │────▶│  Anthropic Claude API     │────▶│  Parse & Map    │
│  Source      │     │  (Analyze transcript)      │     │  (Code node)    │
│  (Webhook)   │     └───────────────────────────┘     └────────┬────────┘
└─────────────┘                                                 │
                                                       ┌────────┴────────┐
                                                       │                 │
                                                       ▼                 ▼
                                              ┌──────────────┐  ┌──────────────┐
                                              │  HubSpot     │  │  HubSpot     │
                                              │  Update      │  │  Create Note │
                                              │  Contact     │  │  (Timeline)  │
                                              └──────────────┘  └──────────────┘
```

Orchestrated via n8n (self-hosted). The workflow receives a transcript via webhook, sends it to Claude for structured analysis, then writes results to HubSpot in two parallel paths: contact properties are updated with the latest call data (current-state view), and a Note engagement is created on the contact's activity timeline (historical log).

## What It Extracts

From a single raw transcript, the LLM returns:

- **Summary** — 2-3 sentence synopsis of the call
- **Caregiver Sentiment** — Positive, Neutral, Concerned, or Distressed
- **Risk Score** — 0-10 scale (0-3 routine, 4-6 monitor, 7-8 escalate, 9-10 urgent)
- **Risk Signals** — Specific clinical indicators (e.g., patient falls, medication non-adherence, cognitive decline, caregiver burnout)
- **Recommended Followup** — Suggested next action for the care team

## Example

**Input transcript:**

> "I need to talk to someone. Mom fell twice this week, once in the bathroom and once getting out of bed. She also got lost driving to the grocery store yesterday, the police found her two hours later. I took her keys. I am not sleeping. I do not know how much longer I can do this alone."

**Output written to HubSpot:**

| Property | Value |
|---|---|
| Risk Score | 8 |
| Caregiver Sentiment | Distressed |
| Risk Signals | patient falls (multiple incidents), cognitive decline, wandering/getting lost, caregiver burnout, sleep deprivation |
| Recommended Followup | Immediate clinical assessment for fall injuries and cognitive evaluation. Refer caregiver to respite care services. |
| Call Summary | Caregiver reports two falls this week and a wandering incident requiring police intervention. Patient's driving privileges revoked. Caregiver expresses significant burnout and sleep deprivation, indicating unsustainable care situation. |

A formatted Note with the full analysis and complete transcript is also created on the contact's activity timeline, preserving the historical record.

## Tech Stack

- **Orchestration:** n8n (self-hosted on Hetzner VPS)
- **LLM:** Anthropic Claude API (claude-sonnet-4-5)
- **CRM:** HubSpot (Service Keys for authentication)
- **Auth:** HubSpot Service Keys (account-level, not user-tied — released Feb 2026, replacing legacy Private Apps)

## Setup

### Prerequisites

- n8n instance (self-hosted or cloud)
- Anthropic API key ([console.anthropic.com](https://console.anthropic.com))
- HubSpot account with Service Key ([developers.hubspot.com](https://developers.hubspot.com))

### HubSpot Configuration

Create the following custom Contact properties (group: "Caregiver Intelligence"):

| Property Label | Internal Name | Field Type |
|---|---|---|
| Risk Score | `risk_score` | Number (unformatted) |
| Call Summary | `call_summary` | Multi-line text |
| Caregiver Sentiment | `caregiver_sentiment` | Dropdown (Positive, Neutral, Concerned, Distressed) |
| Risk Signals | `risk_signals` | Multi-line text |
| Recommended Followup | `recommended_followup` | Multi-line text |
| Last Call Analyzed | `last_call_analyzed` | Date picker |

Service Key scopes required: `crm.objects.contacts.read`, `crm.objects.contacts.write`.

### Import the Workflow

1. Download `Caregiver_Call_Intelligence.json` from this repo
2. In n8n: Workflow menu → Import from File
3. Configure credentials:
   - Anthropic API Key: Header Auth with Name `x-api-key`, Value = your Anthropic key
   - HubSpot Service Key: Header Auth with Name `Authorization`, Value = `Bearer YOUR_SERVICE_KEY`
4. Update the contact ID in your test payload to match your HubSpot contact

### Test

```bash
curl -X POST 'YOUR_N8N_WEBHOOK_URL' \
  -H 'Content-Type: application/json' \
  -d '{
    "contactId": "YOUR_CONTACT_ID",
    "transcript": "Caregiver mentioned mom fell twice this week and seems more confused than usual."
  }'
```

## Production Considerations

This repository is a working reference architecture. Deploying to a production care-coordination environment requires additional engineering across compliance, reliability, and operational maturity. Below is the implementation path scoped for organizations like dementia-focused health plans participating in programs such as Medicare GUIDE.

### HIPAA and PHI Handling

Caregiver call transcripts are Protected Health Information. No real patient data should flow through non-BAA-covered services.

**Transcription:** Use a HIPAA-covered transcription service. RingCentral's native AI transcription (RingSense) is covered under their BAA. AWS Transcribe Medical is an alternative if the telephony platform does not provide transcription. Self-hosted Whisper on HIPAA-compliant infrastructure (AWS, Azure, GCP with BAA) is a third option that avoids third-party data exposure entirely.

**LLM analysis:** The Anthropic API consumer tier used in this demo does not include a BAA. Production options include Anthropic's Enterprise plan (BAA available), Claude via AWS Bedrock (covered under AWS BAA), or OpenAI Enterprise. For maximum data residency control, a self-hosted model (e.g., Llama or Mistral on dedicated GPU infrastructure) eliminates third-party PHI exposure.

**Hosting:** The n8n instance must run on BAA-covered infrastructure. AWS, Azure, or GCP with a signed BAA and appropriate access controls. The current Hetzner deployment is suitable for development only.

### Transcript Source Integration

In production, transcripts originate from the telephony platform rather than manual webhook calls.

**RingCentral integration path:**
1. Register a webhook subscription in RingCentral's developer console for call-recording-ready events
2. On webhook receipt, validate the request signature (RingCentral signs all webhooks)
3. Call the RingCentral API to retrieve the recording and transcript (if RingSense is enabled) or download the audio file for transcription
4. Pass the transcript into the existing analysis chain

The core workflow (transcript → LLM analysis → structured CRM data) remains identical. The new upstream step is: telephony event → audio retrieval → transcription → existing pipeline.

### HubSpot Architecture (Enterprise)

With HubSpot Enterprise, the data model should evolve from the demo's Contact-property approach:

**Custom Object — "Call Analysis":** Each call creates a new Call Analysis record with its own properties (transcript, summary, risk_score, sentiment, risk_signals, recommended_followup, call_date, call_duration, navigator_id). Each record is associated with the parent Contact. This provides full relational structure, per-call reporting, and unlimited history without polluting the Contact object.

**Contact rollup properties:** The Contact retains summary fields (latest_risk_score, last_call_date, total_calls_analyzed, highest_risk_30d) updated on each new analysis. These power list views, lead scoring, and quick navigator reference.

**HubSpot Workflows (native automation):**
- Risk score ≥ 7 → auto-create Task assigned to the navigator with 4-hour SLA
- Risk score ≥ 9 → Slack alert to clinical escalation channel + auto-schedule physician review
- No call in 14 days for high-risk contacts → reminder Task to navigator
- Weekly enrollment of all new contacts into caregiver education sequences based on risk tier

**Dashboards:** Call volume by week, risk score distribution, average time-to-followup on escalations, navigator caseload, trend lines per patient (improving/declining), Medicare GUIDE program reporting metrics.

### Reliability and Error Handling

**Webhook validation:** Verify incoming request signatures before processing. Reject unsigned or invalid requests.

**Retry logic:** Wrap external API calls (Anthropic, HubSpot) in retry with exponential backoff (3 attempts, 2s/4s/8s delays). LLM APIs have transient failures; retries prevent data loss.

**Error routing:** Failed executions route to a dedicated error workflow that logs the failure, sends an alert (Slack/email), and queues the event for replay. No call analysis should be silently dropped.

**Idempotency:** Telephony platforms occasionally fire duplicate webhooks. Deduplicate by checking a hash of (contactId + transcript + timestamp) before processing. Skip if already analyzed.

**Output validation:** Schema-check every LLM response before writing to HubSpot. If risk_score is not 0-10, sentiment is not in the allowed list, or required fields are missing, flag for human review rather than writing potentially invalid data.

### LLM Quality Assurance

**Prompt versioning:** The analysis prompt is business-critical logic. Store it in version control, not embedded in the orchestration node. Changes go through review.

**Sampling:** Route 5% of analyses to a human review queue. Clinical staff validate accuracy weekly. Incorrect analyses become regression test cases.

**Feedback loop:** Care navigators can flag an analysis as incorrect directly in HubSpot (e.g., via a dropdown property "Analysis Accuracy: Correct / Incorrect / Partially Correct" on the Call Analysis object). Flagged records feed back into prompt improvement cycles.

**Model evaluation:** When upgrading model versions or revising prompts, run the full regression suite (accumulated from flagged errors + hand-labeled samples) and compare output quality before deploying.

### Security

- API keys and service credentials stored in a secrets manager (AWS Secrets Manager, HashiCorp Vault, Doppler), not in orchestration platform credential stores
- n8n instance behind a VPN or private network, not publicly accessible
- Role-based access control on HubSpot properties — clinical data visible only to authorized care team members
- Audit logging on all data writes for compliance review
- Service Key rotation on a 90-day schedule

### Implementation Timeline

| Phase | Weeks | Scope |
|---|---|---|
| Compliance & Infrastructure | 1-2 | BAA execution with LLM provider, HIPAA-compliant hosting setup, security review |
| Integration Build | 3-4 | RingCentral webhook integration, transcription pipeline, Custom Object in HubSpot, hardened n8n workflow |
| Pilot (Shadow Mode) | 5-6 | Deploy with 5-10 navigators, AI runs alongside manual review, no automated actions |
| Prompt Tuning | 7-8 | Refine prompts based on pilot accuracy data, build HubSpot automation (alerts, tasks, dashboards) |
| Soft Launch | 9-10 | One care team fully live, monitoring SLAs and escalation accuracy |
| Full Rollout | 11-12 | All care teams live, navigator training, documentation, runbooks |

## Disclaimer

All transcripts and contact data in this repository are synthetic, generated for demonstration purposes. No real patient, caregiver, or protected health information was used at any stage of development or testing. This project is a reference architecture and is not approved for use with real PHI without the compliance measures described above.

## License

MIT
