# n8n Lead Automation

An n8n workflow that turns an inbound webhook into a qualified CRM record.

`lead-automation.workflow.json` is the importable export (19 nodes). It validates and
normalises the incoming lead, looks the contact up in GoHighLevel, scores it with an
LLM call, upserts the contact with the score and reasoning in custom fields, opens an
opportunity, sends a follow-up email, and posts a summary to Telegram.

Custom field IDs are resolved by name at runtime, so the export imports into any
sub-account without editing nodes. Retried webhooks update the existing contact rather
than duplicating it. If the CRM write fails, the full lead is sent to Telegram and the
webhook answers 502, so nothing is silently dropped.

## Requirements

Three n8n credentials. The names matter, the import references them.

| Credential | Type | Header |
|---|---|---|
| `GHL Private Integration Token` | Header Auth | `Authorization: Bearer <token>` |
| `Anthropic API Key` | Header Auth | `x-api-key: <key>` |
| `Telegram Bot` | Telegram API | bot token |

Environment variables:

```
GHL_LOCATION_ID=
GHL_PIPELINE_ID=
TELEGRAM_CHAT_ID=
```

n8n only exposes these to expressions when `N8N_BLOCK_ENV_ACCESS_IN_NODE=false`.

The GHL sub-account needs contact custom fields named `AI Lead Score` (number),
`AI Qualification` (dropdown Hot/Warm/Cold), `AI Reason` (multi-line), `Lead Source Raw`
(text) and `Intake Timestamp` (date), plus a pipeline for the opportunity step.

## Usage

Import the JSON, open each HTTP node once to bind its credential, activate, then POST a
lead:

```bash
curl -X POST https://<host>/webhook/<path> \
  -H 'Content-Type: application/json' \
  -d '{"firstName":"Dana","email":"dana@example.com","company":"Northgate Logistics",
       "message":"40 vehicles, despatch still on spreadsheets. Looking to move in Q1."}'
```

```json
{"ok":true,"contactId":"...","tier":"Hot","score":82,"duplicate":false}
```
