# Email Triage Agent - n8n + Claude Workflow

Intelligent email classification and routing system built with n8n and Claude AI.

## Workflow (18 Nodes)

```
IMAP Trigger → Content Extraction → Claude Classification → Switch Router
                                                                    ↓
     URGENT_ACTION → Slack + SMS
     CLIENT_REQUEST ⿢ Asana Task
     INFORMATIONAL → Google Sheet Log
     SPAM → Auto-Archive
     INTERNAL → Team Channel
```

## Features

- **5-Category Classification**: URGENT, CLOENT, INFORMATIONAL, INTERNAL, SPAM
- **Smart Routing**: Each category gets appropriate action
- **Data Extraction**: Pulls key info (dates, amounts, names)
- **Error Handling**: Fallback routing, retry logic
- **Audit Trail**: Every email logged with classification

## Setup

1. Import `workflow.json` into n8n
2. Configure IMAP credentials
3. Set up Claude API key
4. Connect Slack, Asana, Google Sheets
5. Activate

## Technology

n8n | Claude AI | IMAP | Slack | Asana | Google Sheets | Twilio
