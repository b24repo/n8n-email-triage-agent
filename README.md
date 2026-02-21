# Email Triage Agent - MVP3

A production-grade n8n workflow for intelligent email classification and routing using AI-powered analysis. This system automatically categorizes incoming emails and routes them to appropriate channels based on urgency, sender type, and content analysis.

## Architecture Overview

The Email Triage Agent follows a modular, scalable design pattern with clear separation of concerns:

```
IMAP Trigger
    ↓
Extract Email Metadata
    ↓
AI Classification (Claude/OpenAI)
    ↓
Parse Classification Results
    ↓
AI Data Extraction (Key Points)
    ↓
Parse Extracted Data
    ↓
Switch Router (Classification-Based Routing)
    ↓
[5 Output Paths]
├── URGENT_ACTION → Slack #urgent + SMS Alert
├── CLIENT_REQUEST → Slack #client-requests + Asana Task
├── INFORMATIONAL → Google Sheets Log
├── INTERNAL → Slack #team
└── SPAM → Move to Trash
```

## Workflow Components

### 1. **IMAP Email Trigger**
- **Type:** `n8n-nodes-imap.imap`
- **Function:** Monitors configured IMAP mailbox for new emails
- **Configuration:**
  - **Mailbox:** INBOX (configurable)
  - **Post Processing:** Mark as read after processing
  - **SSL/TLS:** Enabled for security
- **Output:** Raw email object with headers, body, attachments, sender info

**Setup Requirements:**
```
Credentials needed:
- IMAP Server (e.g., imap.gmail.com)
- Email Address
- App Password (for Gmail: generate at myaccount.google.com/apppasswords)
- Port: 993 (SSL)
```

### 2. **Extract Email Content**
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Function:** Normalizes and structures email data
- **Extracts:**
  - Subject
  - Body (HTML or plaintext)
  - Sender name and address
  - Date
  - Message ID
  - CC/BCC recipients
  - Attachment count
  - Raw email for audit trail

**Processing Logic:**
- Handles both HTML and plaintext emails
- Preserves Message-ID for email threading
- Extracts sender metadata with fallback handling
- Tracks attachment presence without loading content

### 3. **Classify Email (Claude/OpenAI)**
- **Type:** `n8n-nodes-base.openai`
- **Model:** GPT-4-Turbo (configurable)
- **Function:** AI-powered email classification
- **Temperature:** 0.3 (low variance for consistency)

**Classification Categories:**
1. **URGENT_ACTION** - Time-sensitive, requires immediate response
2. **INFORMATIONAL** - FYI, newsletters, announcements
3. **CLIENT_REQUEST** - External client/customer inquiries
4. **INTERNAL** - Team communication, meetings
5. **SPAM** - Junk, promotional, unsubscribe

**Prompt Engineering:**
- Analyzes subject line and first 2000 characters of body
- Returns structured JSON with classification, confidence score, and reasoning
- Temperature set low for deterministic results
- Fallback parsing for unexpected response formats

### 4. **Parse Classification**
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Function:** Extract and validate AI response
- **Features:**
  - JSON parsing with error handling
  - Fallback pattern matching
  - Confidence score preservation
  - Classification reasoning capture

### 5. **Extract Key Data Points**
- **Type:** `n8n-nodes-base.openai`
- **Function:** Second-pass AI analysis for structured data extraction
- **Extracts:**
  - Action items (array)
  - Deadlines/dates
  - People mentioned
  - Priority level (HIGH/MEDIUM/LOW)
  - Sender category (internal/external/VIP)
  - Required response flag
  - Estimated time to respond

**Prompt Includes:**
- Classification context from previous step
- First 2000 characters of body
- Temperature: 0.2 for high consistency

### 6. **Parse Extracted Data**
- **Type:** `n8n-nodes-base.code` (JavaScript)
- **Function:** Validate and normalize extracted data
- **Ensures:**
  - Proper array structures
  - Default values for missing fields
  - Type consistency
  - Complete data objects

### 7. **Switch Router**
- **Type:** `n8n-nodes-base.switch`
- **Function:** Route emails based on classification
- **Branches:**
  - `urgent` → URGENT_ACTION path
  - `client` → CLIENT_REQUEST path
  - `informational` → INFORMATIONAL path
  - `internal` → INTERNAL path
  - `spam` → SPAM path

## Output Paths & Routing

### Path 1: URGENT_ACTION
**Nodes:**
- Slack Message to #urgent channel with emoji notification
- SMS Alert via Twilio to configured admin phone

**Slack Message Format:**
```
🚨 URGENT EMAIL ALERT

From: [Sender Name]
Subject: [Email Subject]
Priority: [HIGH/MEDIUM/LOW]
Action Items: [Item 1, Item 2, ...]
Deadlines: [Deadline 1, Deadline 2, ...]
```

**SMS Alert:**
```
URGENT EMAIL ALERT: [Subject] from [Sender Name]
```

**Configuration:**
- Phone number: `$env.ADMIN_PHONE_NUMBER`
- Requires Twilio API credentials

---

### Path 2: CLIENT_REQUEST
**Nodes:**
- Slack Message to #client-requests channel
- Create task in Asana/Project Management

**Slack Message Format:**
```
📋 CLIENT REQUEST

From: [Sender Name]
Subject: [Email Subject]
Priority: [Priority Level]
Action Items: [Items identified by AI]
```

**Task Creation (Asana):**
- Title: `[CLIENT] [Email Subject]`
- Description includes sender and action items
- Status based on priority (HIGH → pending, else → backlog)

**Configuration:**
- Project/Document ID: `$env.PROJECT_MGMT_DOC_ID`
- Requires Asana API credentials

---

### Path 3: INFORMATIONAL
**Action:**
- Log to Google Sheet (audit trail, searchable archive)

**Sheet Columns:**
| Timestamp | Sender | Sender Name | Subject | Classification | Priority | Action Items | Deadlines | Message ID |
|-----------|--------|-------------|---------|-----------------|----------|--------------|-----------|------------|

**Configuration:**
- Sheet ID: `$env.LOGGING_SHEET_ID`
- Range: A:F (auto-extends)
- Requires Google Sheets API credentials

---

### Path 4: INTERNAL
**Action:**
- Slack Message to #team channel

**Slack Message Format:**
```
📬 INTERNAL UPDATE

From: [Sender Name]
Subject: [Email Subject]
```

**Configuration:**
- Channel name configurable
- Requires Slack workspace access

---

### Path 5: SPAM
**Action:**
- Move to Trash folder in IMAP

**Processing:**
- Email marked for deletion
- Removed from inbox immediately
- Recoverable from trash for 30 days

---

## Setup & Installation

### Prerequisites
- n8n instance (v0.200+)
- Active email account (Gmail, Office 365, etc.)
- Slack workspace with admin access
- OpenAI API key (GPT-4-Turbo access recommended)
- (Optional) Twilio account for SMS
- (Optional) Asana/project management integration
- (Optional) Google Sheets access for logging

### Step 1: Import Workflow
1. In n8n, go to **Workflows** → **Import from URL** or **Import from File**
2. Select the `workflow.json` file
3. Click **Import**

### Step 2: Configure Credentials

#### 2.1 IMAP Email Connection
```
Credentials → Create New → IMAP
├── Host: imap.gmail.com (or your provider)
├── Port: 993
├── Username: your-email@gmail.com
├── Password: [App Password, not account password]
├── SSL: Enabled
└── TLS: Enabled
```

#### 2.2 OpenAI API
```
Credentials → Create New → OpenAI
├── API Key: sk-...
└── Resource: OpenAI
```

#### 2.3 Slack API
```
Credentials → Create New → Slack
├── Authentication: OAuth 2.0
├── Bot Token Scopes:
│   ├── chat:write
│   ├── chat:write.public
│   └── files:write
└── Authorize with your Slack workspace
```

#### 2.4 Twilio (Optional)
```
Credentials → Create New → Twilio
├── Account SID: [from Twilio Dashboard]
├── Auth Token: [from Twilio Dashboard]
└── From Number: [your Twilio phone number]
```

#### 2.5 Asana (Optional)
```
Credentials → Create New → Asana
├── Personal Access Token: [from Asana account settings]
└── Authentication: Personal Access Token
```

#### 2.6 Google Sheets (Optional)
```
Credentials → Create New → Google Sheets
├── Authentication: OAuth 2.0
└── Authorize with Google account
```

### Step 3: Set Environment Variables

In n8n **Settings** → **Environment Variables**, add:

```env
ADMIN_PHONE_NUMBER="+1234567890"
PROJECT_MGMT_DOC_ID="asana-project-id-or-doc-id"
LOGGING_SHEET_ID="google-sheet-id"
SMS_TWILIO_FROM="+1234567890"
```

### Step 4: Configure Slack Channels
Ensure your Slack workspace has the following channels:
- `#urgent` - For urgent emails
- `#client-requests` - For client requests
- `#team` - For internal emails
- `#automation-logs` - For error notifications

Create channels if they don't exist, or update channel names in workflow nodes.

### Step 5: Activate Workflow
1. Click **Activate** toggle in top-right
2. Workflow will monitor configured inbox for new emails
3. Test by sending a sample email to the configured address

---

## Customization Guide

### Changing Classification Categories

To add or modify classifications:

1. **Update AI Prompt** (Node: "Classify Email")
   - Edit the prompt in the OpenAI node
   - Add new category to the list
   - Update prompt text with examples

2. **Update Switch Router** (Node: "Switch Router")
   - Add new branch with the new category value
   - Specify output key for routing

3. **Add Output Path**
   - Create new nodes for the new classification
   - Connect from switch router to new nodes

**Example: Add VIP category**
```json
{
  "value": "VIP_CLIENT",
  "outputKey": "vip"
}
```

Then create output nodes for VIP path (e.g., Slack #vip-clients, email team lead).

### Adjusting AI Model
- **Temperature:** 0-1 scale (lower = more consistent, higher = more creative)
- **Model:** Change from `gpt-4-turbo` to `gpt-4`, `gpt-3.5-turbo`, or `claude-3-opus`
- **Max Tokens:** Adjust for longer/shorter responses

**Location:** Nodes "Classify Email" and "Extract Key Data Points"

### Modifying Output Channels

**For Slack routing:**
1. Edit node (e.g., "Slack - Urgent Channel")
2. Change `channel` parameter to target different channel
3. Customize message template using expressions

**For other integrations:**
- Remove node and replace with alternative (Microsoft Teams, Discord, etc.)
- Map same data fields to new node format

### Adding New Output Destinations

**Pattern:**
1. Add new node of desired type (e.g., Email, Webhook, HTTP)
2. Connect from Switch Router output
3. Configure credentials
4. Map email data to node parameters

**Example: Send email notification for urgent items**
```
Switch Router (urgent branch)
    ↓
Email node
├── To: [Team Lead Email]
├── Subject: [Email Subject]
└── Body: [Structured message]
```

### Performance Optimization

**Rate Limiting:**
- Adjust IMAP polling interval (default: per-minute)
- Add delay between operations if hitting API rate limits

**Batch Processing:**
- Use **Loop** node to process multiple emails
- Implement queue-based processing for high volume

**Caching:**
- Cache AI classification results for identical subjects
- Reduce API calls for duplicate email patterns

---

## Error Handling & Monitoring

### Error Handler Nodes
The workflow includes:
- **Error Handler - Slack:** Posts errors to #automation-logs
- **Log Error to External Service:** Webhook for external monitoring

### Common Issues & Solutions

| Issue | Cause | Solution |
|-------|-------|----------|
| "Authentication failed" | Invalid IMAP credentials | Verify app password for Gmail; check server settings |
| "Classification empty" | API timeout or rate limit | Check OpenAI quota; increase timeout in node settings |
| "Slack message failed" | Invalid channel or permissions | Verify channel exists; check bot scopes and token |
| "Task creation failed" | Invalid project ID or permissions | Confirm Asana project; verify token has project access |
| "Email moved twice" | Duplicate trigger execution | Implement idempotency check with message ID |

### Monitoring Checklist
- Monitor workflow executions in n8n dashboard
- Check #automation-logs channel for error notifications
- Review Google Sheet logging for processing trends
- Test monthly with sample emails from each category
- Monitor API usage (OpenAI, Slack, Asana, Google)

---

## Security Best Practices

1. **Credentials Management:**
   - Use n8n's encrypted credential storage
   - Never hardcode API keys or passwords
   - Rotate credentials regularly (especially app passwords)
   - Use least-privilege service accounts

2. **Data Privacy:**
   - Configure email content retention policy
   - Limit email body preview to 2000 characters
   - Archive emails in compliance with regulations
   - Encrypt sensitive data in transit and at rest

3. **Access Control:**
   - Restrict workflow execution to specific users
   - Audit who can modify workflow
   - Use separate Slack tokens for production/staging
   - Implement approval workflows for critical changes

4. **Logging & Audit Trail:**
   - Enable manual execution logging
   - Log all classification decisions to sheet
   - Preserve message IDs for audit
   - Monitor for unusual patterns

---

## Workflow Logic Explanation

### Flow Decision Points

1. **Email Extraction:** Normalizes diverse email formats into consistent structure
2. **Classification:** Uses AI to understand context and intent
3. **Data Extraction:** Pulls actionable information from email content
4. **Routing:** Single decision point (switch) directs to appropriate handlers
5. **Output Diversification:** Different channels for different stakeholders

### Key Features

- **Confidence Scoring:** AI provides confidence in classification (0-1)
- **Action Item Detection:** Automatically identifies tasks from email content
- **Deadline Recognition:** Extracts dates and deadlines mentioned
- **Sender Categorization:** Distinguishes internal vs. external senders
- **Idempotency:** Uses message ID to prevent duplicate processing
- **Fallback Handling:** Gracefully handles parsing errors with defaults

---

## Maintenance & Scaling

### Regular Maintenance Tasks

**Weekly:**
- Review #automation-logs for errors
- Monitor email processing volume
- Check API usage and costs

**Monthly:**
- Audit classification accuracy (sample 20-30 emails)
- Review customer feedback on categorization
- Update classification rules if needed
- Test all integrations (Slack, Asana, Google Sheets)

**Quarterly:**
- Review workflow performance metrics
- Optimize slow steps
- Update documentation
- Plan for feature additions

### Scaling for High Volume

1. **Rate Limiting:** Implement backoff strategy for API calls
2. **Batching:** Group emails for batch processing
3. **Caching:** Cache classification results for common patterns
4. **Async Processing:** Use n8n workflows queue for async execution
5. **Database Logging:** Move from Google Sheets to database for large scale

---

## Advanced Features

### Custom Classification Prompts

Modify the AI prompt to include organization-specific categories:

```javascript
// Example: Add security-related classification
You are an expert email triage assistant.
Analyze emails for your company's specific needs:

Categories:
1. URGENT_ACTION - Immediate response required
2. SECURITY_INCIDENT - Security/privacy concerns
3. FINANCIAL - Invoice, expense, payment
4. COMPLIANCE - Regulatory, audit, legal
5. TECHNICAL_SUPPORT - Bugs, incidents
6. INFORMATIONAL - FYI, newsletters
7. SPAM - Junk, promotional
```

### Integration Extensions

**Available integrations:**
- **Messaging:** Microsoft Teams, Discord, Telegram, Rocket.Chat
- **Project Management:** Jira, Monday.com, ClickUp, Notion
- **CRM:** Salesforce, HubSpot, Pipedrive
- **Databases:** PostgreSQL, MySQL, MongoDB, Firebase
- **Analytics:** Datadog, New Relic, Segment
- **Webhooks:** Custom HTTP endpoints for any service

### Machine Learning Feedback Loop

Implement user feedback to improve classification:

1. Add "Feedback" buttons in Slack messages
2. Capture corrections in database
3. Periodically retrain classification logic
4. Update prompts based on error patterns

---

## Troubleshooting

### Workflow Won't Start
- Check IMAP credentials in test mode
- Verify email account allows IMAP access
- Ensure "Less secure app access" is enabled (Gmail)

### Emails Not Being Processed
- Check workflow is activated (blue toggle)
- Verify IMAP polling interval
- Check execution history for errors
- Test with manual workflow trigger

### Classification Errors
- Review AI prompt wording
- Check OpenAI API quota
- Verify API key is valid
- Increase temperature if classifications too random

### Slack Messages Not Sending
- Verify bot token and scopes
- Confirm channels exist
- Check bot is member of channel
- Review Slack API rate limits

---

## Support & Documentation

For n8n-specific help:
- Official Docs: https://docs.n8n.io
- Community Forum: https://community.n8n.io
- Discord: https://discord.gg/n8n

For integration-specific help:
- OpenAI: https://platform.openai.com/docs
- Slack: https://api.slack.com
- Asana: https://developers.asana.com
- Google Sheets: https://developers.google.com/sheets

---

## License & Attribution

This Email Triage Agent workflow is provided as-is for production use. Customize as needed for your organization.

**Version:** MVP3 (2026-02-21)
**Compatibility:** n8n v0.200+
**Status:** Production-Ready
