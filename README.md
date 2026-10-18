AI Lead Management for Real Estate (n8n)

An n8n workflow that takes inquiries from a web form, cleans and validates them, scores them with AI, syncs them to a CRM, creates follow-up tasks, and alerts the sales team. Built for a real estate business in Bangladesh (BDT, +880 phone numbers), but easy to adapt.

Status: practice project built to learn n8n end to end. Test it with your own accounts before using it with real customers.

What it does
Tally form
   │  webhook
   ▼
Validate & normalize ──► invalid? ──► Exceptions Log
   │
   ▼
Idempotency ledger ──► duplicate submission? ──► Exceptions Log
   │
   ▼
AI qualification (Claude): category, score, priority, out-of-area
   │
   ▼
Spam? ──► Exceptions Log
   │
   ▼
HubSpot: search contact ──► found: update │ not found: create
   │
   ▼
Priority router
   ├─ Hot  ─► (optional auto email) ─► HubSpot call task ─► Telegram alert
   ├─ Warm ─► HubSpot call task ─► Telegram alert
   └─ Cold ─► log only
   │
   ▼
Google Sheet "Leads" ──► ledger marked completed

Separate flows: Error workflow (alert + log), internal Grok sales chatbot
Features
Validation and normalization: flattens the Tally payload, formats phone numbers to +880…, lowercases emails, splits names, checks consent, honeypot and disposable email domains.
Duplicate protection: a Google Sheet ledger blocks the same submission from being processed twice. A CRM search stops the same person from being created twice.
AI qualification: Claude returns a structured result (category, score 0–100, priority, budget signal, out-of-area flag). A Structured Output Parser enforces the format.
Your rules, not the AI's: hot/warm thresholds come from the Config node. If the AI fails, the lead is flagged for human review instead of breaking the run.
CRM sync: HubSpot contact search, create and update, plus a call task due in 2h (hot), 24h (warm) or 72h (cold).
Notifications: Telegram message to the sales team for hot and warm leads.
Logging: every lead goes to a Google Sheet; invalid, duplicate and spam leads go to an Exceptions Log.
Error handling: retries on API calls, plus a global error workflow that alerts and logs.
Sales assistant (optional): an n8n AI agent (Grok) that looks up contacts in HubSpot and drafts replies. It never sends messages itself.
Tech stack

n8n (self-hosted, v2.x) · Tally · Anthropic Claude · HubSpot (Service Key) · Google Sheets · Telegram · xAI Grok (optional)

Setup
1. Google Sheet

Create one spreadsheet with three tabs. Header names must match exactly (no extra spaces).

Leads

lead_key | first_name | last_name | email | phone | company | source | message | category | score | priority | owner | crm_contact_id | intent_summary | ai_status | needs_review | last_updated | intent | property_type | location | bedrooms | budget | timeline | contact_time | out_of_area

Processing Ledger

submission_id | lead_key | status | updated_at

Exceptions Log

logged_at | type | lead_key | submission_id | details | payload
2. Credentials in n8n
Credential	Used by
Google Sheets (OAuth2)	all Sheets nodes
Anthropic API key	Claude Chat Model
HubSpot Service Key	search, create, update, tasks
Telegram Bot token	Notify Sales, error alerts
xAI API key (optional)	Grok chatbot

Create the HubSpot service key under Development → Keys → Service keys with the scopes crm.objects.contacts.read and crm.objects.contacts.write. If task creation returns a scope error, add the tasks scope too.

3. Import and configure
Import workflow.json into n8n.
Open the Config node and set sheet_id, company_name, owners_csv, support_email, thresholds and service_areas.
Select your credentials in each node.
Keep auto_followup_enabled set to false until you have reviewed the email text.
4. Connect the form

In Tally: Integrations → Webhooks, and paste the production URL:

https://YOUR-N8N-DOMAIN/webhook/lead-intake

Use the /webhook-test/ URL only while testing, and unpin any pinned node data before going live. Pinned data makes a node reuse an old result instead of running.

5. Test before activating
Test	Expected
New valid lead	HubSpot contact + task, Sheet row, Telegram alert
Same submission sent twice	second one logged as duplicate, nothing else runs
Same email, new submission	existing HubSpot contact updated, no duplicate
Spam text	logged to Exceptions Log, not sent to HubSpot
Missing phone and email, or no consent	logged as invalid

Then activate the workflow and set the Error Workflow in the workflow settings.

Configuration reference
Config field	Meaning
default_country_code	applied to local phone numbers (+880)
hot_threshold / warm_threshold	score cut-offs (default 70 / 40)
owners_csv	sales emails, assigned round-robin
auto_followup_enabled	allow automatic follow-up email
ledger_lock_minutes	how long a "processing" row blocks retries (default 10)
service_areas	areas the company serves, used for the out-of-area flag
Known limitations
The Google Sheets ledger is not atomic. Two identical submissions arriving in the same second can both pass the check. A database with a unique constraint would fix this.
HubSpot search can lag by a few seconds after a contact is created. HubSpot still rejects a duplicate email with a 409 error, which the workflow does not yet recover from automatically.
Task owner is not set in HubSpot yet. Owners are chosen by the workflow, but HubSpot needs numeric owner IDs, which would need a lookup step.
Custom HubSpot properties (budget, area, property type) are saved in the Sheet only. Create custom properties in HubSpot to sync them.
Round-robin owner assignment only persists in production executions, not test runs.
Security notes
Never commit API keys or tokens. Credentials live in n8n, not in the exported JSON.
Review the exported JSON for real emails, sheet IDs and webhook URLs before publishing.
The webhook uses no authentication because browser forms cannot hide a secret. A honeypot field and consent check reduce abuse. Add rate limiting at your reverse proxy for production.
