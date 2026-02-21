# Email Triage Agent - Architecture Documentation

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          EMAIL TRIAGE AGENT FLOW                        │
└─────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│  STAGE 1: EMAIL INGESTION & NORMALIZATION                              │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────┐                                            │
│  │  IMAP Email Trigger     │                                            │
│  │  ─────────────────────  │                                            │
│  │  • Monitor inbox        │                                            │
│  │  • Poll every 1 min     │                                            │
│  │  • Extract raw email    │                                            │
│  │  • Mark as read         │                                            │
│  └──────────────┬──────────┘                                            │
│                 │                                                        │
│                 v                                                        │
│  ┌─────────────────────────────────────────┐                           │
│  │  Extract Email Content (JS Code)        │                           │
│  │  ─────────────────────────────────────  │                           │
│  │  • Normalize email structure            │                           │
│  │  • Extract: subject, body, sender       │                           │
│  │  • Capture: date, message-id, cc/bcc   │                           │
│  │  • Count attachments                    │                           │
│  │  • Preserve raw for audit               │                           │
│  └──────────────┬──────────────────────────┘                           │
│                 │                                                        │
│         Output: Structured email object                                 │
│         Fields: subject, body, sender, senderName, date,               │
│                messageId, cc, bcc, attachmentCount, isHtml, rawEmail   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│  STAGE 2: AI CLASSIFICATION                                            │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────┐                           │
│  │  Classify Email (OpenAI GPT-4-Turbo)    │                           │
│  │  ─────────────────────────────────────  │                           │
│  │  • Input: subject + first 2000 chars    │                           │
│  │  • Temperature: 0.3 (consistent)        │                           │
│  │  • Prompt: Multi-choice classification  │                           │
│  │  • Output: JSON with confidence score   │                           │
│  │  • Model: gpt-4-turbo (configurable)    │                           │
│  └──────────────┬──────────────────────────┘                           │
│                 │                                                        │
│         Classifications:                                                 │
│         ├─ URGENT_ACTION (requires immediate response)                  │
│         ├─ CLIENT_REQUEST (customer/client inquiry)                     │
│         ├─ INFORMATIONAL (FYI/newsletter/announcement)                  │
│         ├─ INTERNAL (team communication)                                │
│         └─ SPAM (junk/promotional)                                      │
│                 │                                                        │
│                 v                                                        │
│  ┌─────────────────────────────────────────┐                           │
│  │  Parse Classification (JS Code)         │                           │
│  │  ─────────────────────────────────────  │                           │
│  │  • Validate JSON response               │                           │
│  │  • Fallback pattern matching            │                           │
│  │  • Preserve confidence score            │                           │
│  │  • Error handling with defaults         │                           │
│  └──────────────┬──────────────────────────┘                           │
│                 │                                                        │
│         Output: {                                                        │
│           classification: string,                                       │
│           classificationConfidence: 0-1,                                │
│           classificationReasoning: string,                              │
│           ...previousFields                                             │
│         }                                                                │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│  STAGE 3: DATA EXTRACTION & ENRICHMENT                                  │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────┐                           │
│  │  Extract Key Data Points (OpenAI)       │                           │
│  │  ─────────────────────────────────────  │                           │
│  │  • Input: Classification + email body   │                           │
│  │  • Temperature: 0.2 (very consistent)   │                           │
│  │  • Structured JSON extraction           │                           │
│  │  • Context-aware (includes category)    │                           │
│  └──────────────┬──────────────────────────┘                           │
│                 │                                                        │
│         Extracts:                                                        │
│         ├─ actionItems: [item1, item2, ...]                             │
│         ├─ deadlines: [date1, date2, ...]                               │
│         ├─ mentionedPeople: [name1, name2, ...]                         │
│         ├─ priorityLevel: HIGH/MEDIUM/LOW                               │
│         ├─ sender_category: internal/external/vip                       │
│         ├─ requiredResponse: boolean                                    │
│         └─ estimatedTimeToRespond: minutes                              │
│                 │                                                        │
│                 v                                                        │
│  ┌─────────────────────────────────────────┐                           │
│  │  Parse Extracted Data (JS Code)         │                           │
│  │  ─────────────────────────────────────  │                           │
│  │  • Validate data structures             │                           │
│  │  • Apply default values                 │                           │
│  │  • Type consistency checking            │                           │
│  │  • Merge with previous data             │                           │
│  └──────────────┬──────────────────────────┘                           │
│                 │                                                        │
│         Output: Enriched email object with:                              │
│         ├─ Original email metadata                                      │
│         ├─ Classification results                                       │
│         ├─ Extracted key data                                           │
│         └─ Parsed values with defaults                                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│  STAGE 4: ROUTING & OUTPUT                                             │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌─────────────────────────────────────────┐                           │
│  │  Switch Router (Classification-based)   │                           │
│  │  ─────────────────────────────────────  │                           │
│  │  • Input field: classification          │                           │
│  │  • 5 branches (one per category)        │                           │
│  │  • Output keys: urgent/client/info/     │                           │
│  │    internal/spam                        │                           │
│  └──────────────┬──────────────────────────┘                           │
│                 │                                                        │
│    ┌────────────┼────────────┬──────────────┬────────────┐             │
│    │            │            │              │            │             │
│    v            v            v              v            v             │
│                                                                          │
│  URGENT_ACTION    CLIENT_REQUEST   INFORMATIONAL   INTERNAL     SPAM    │
│  Branch           Branch           Branch          Branch      Branch   │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────────────┐
│  OUTPUT BRANCHES                                                        │
├──────────────────────────────────────────────────────────────────────────┤

URGENT_ACTION BRANCH:
├─ Slack Message → #urgent channel
│  ├─ Message: 🚨 URGENT EMAIL ALERT
│  ├─ Content: From, Subject, Priority, Action Items, Deadlines
│  └─ Emoji: :rotating_light:
│
├─ SMS Alert → Twilio
│  ├─ To: $env.ADMIN_PHONE_NUMBER
│  └─ Text: "URGENT EMAIL ALERT: [Subject] from [Sender]"
│
└─ Parallel processing (both nodes execute)

CLIENT_REQUEST BRANCH:
├─ Slack Message → #client-requests channel
│  ├─ Message: 📋 CLIENT REQUEST
│  ├─ Content: From, Subject, Priority, Action Items
│  └─ Emoji: :clipboard:
│
├─ Asana Task Creation
│  ├─ Title: [CLIENT] [Email Subject]
│  ├─ Description: Sender email + Action Items
│  ├─ Status: HIGH=pending, else=backlog
│  └─ Project: $env.PROJECT_MGMT_DOC_ID
│
└─ Parallel processing

INFORMATIONAL BRANCH:
└─ Google Sheets Logging
   ├─ Sheet: $env.LOGGING_SHEET_ID
   ├─ Columns: Timestamp, Sender, Subject, Classification,
   │            Priority, Action Items, Deadlines, Message ID
   └─ Auto-append row

INTERNAL BRANCH:
└─ Slack Message → #team channel
   ├─ Message: 📬 INTERNAL UPDATE
   ├─ Content: From, Subject
   └─ Emoji: :mailbox:

SPAM BRANCH:
└─ IMAP Delete → Move to Trash folder
   ├─ Action: Mark for deletion
   ├─ Recoverable: 30 days (folder dependent)
   └─ Permanent removal: Configure IMAP setting

└──────────────────────────────────────────────────────────────────────────┘
```

## Data Flow Diagram

```
┌─────────────┐
│   Email     │
│   Server    │
└──────┬──────┘
       │
       │ IMAP Protocol (Port 993, SSL)
       │
       v
┌────────────────────────────────────────────────────────────────┐
│                    n8n Workflow                                │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │   Trigger    │→ │   Extract    │→ │  Classify    │        │
│  │   (IMAP)     │  │   Content    │  │   (OpenAI)   │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
│         │                   │                  │              │
│         │    Email Data     │    Classification │              │
│         v                   v                  v              │
│       Data               Data             JSON:               │
│       ├─ from           ├─ subject       {                    │
│       ├─ to             ├─ body          class,               │
│       ├─ subject        ├─ sender        conf,                │
│       ├─ body           ├─ date          reason              │
│       ├─ date           ├─ attachments   }                    │
│       ├─ cc             └─ cc/bcc                             │
│       └─ bcc                                                  │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │   Parse      │←─│   Extract    │→ │    Parse     │        │
│  │  Classification   Data Points      Extracted     │        │
│  └────────┬─────┘  └──────────────┘  └──────────────┘        │
│           │                │               │                  │
│           │    OpenAI      │               │                  │
│           │    Response    v               v                  │
│           │            JSON:          JSON:                  │
│           │            {              {                      │
│           │            actions,       actions,               │
│           │            deadlines,     deadlines,             │
│           │            people,        priority,              │
│           │            priority,      sender_cat,            │
│           │            ...            ...                    │
│           │            }              }                      │
│           │                                                   │
│           v                                                   │
│  ┌──────────────────────────────────────────┐               │
│  │        Enriched Email Object             │               │
│  │  ┌──────────────────────────────────┐    │               │
│  │  │ Original Fields:                 │    │               │
│  │  │ ├─ subject                       │    │               │
│  │  │ ├─ body                          │    │               │
│  │  │ ├─ sender                        │    │               │
│  │  │ ├─ date                          │    │               │
│  │  │ └─ messageId                     │    │               │
│  │  │                                  │    │               │
│  │  │ Classification Results:          │    │               │
│  │  │ ├─ classification                │    │               │
│  │  │ ├─ classificationConfidence      │    │               │
│  │  │ └─ classificationReasoning       │    │               │
│  │  │                                  │    │               │
│  │  │ Extracted Data:                  │    │               │
│  │  │ ├─ actionItems: []               │    │               │
│  │  │ ├─ deadlines: []                 │    │               │
│  │  │ ├─ mentionedPeople: []           │    │               │
│  │  │ ├─ priorityLevel                 │    │               │
│  │  │ ├─ sender_category               │    │               │
│  │  │ ├─ requiredResponse              │    │               │
│  │  │ └─ estimatedTimeToRespond        │    │               │
│  │  └──────────────────────────────────┘    │               │
│  └────────────────┬─────────────────────────┘               │
└─────────────────┼──────────────────────────────────────────┘
                  │
                  v
        ┌─────────────────────┐
        │   Switch Router     │
        │  (Classification)   │
        └────────┬────────────┘
                 │
    ┌────────────┼─────────────┬──────────────┬──────────┐
    │            │             │              │          │
    v            v             v              v          v
┌─────────┐ ┌─────────┐ ┌─────────────┐ ┌─────────┐ ┌──────┐
│ Urgent  │ │ Client  │ │Information  │ │Internal │ │ Spam │
│ Path    │ │ Path    │ │ al Path     │ │ Path    │ │ Path │
└────┬────┘ └────┬────┘ └──────┬──────┘ └────┬────┘ └───┬──┘
     │           │              │             │          │
     │           │              │             │          │
  ┌──v─────┐  ┌──v──────┐  ┌───v──────┐ ┌────v──┐  ┌──v────┐
  │Slack   │  │Slack    │  │Google    │ │Slack  │  │Delete │
  │#urgent │  │#client- │  │Sheets    │ │#team  │  │to     │
  │        │  │requests │  │          │ │       │  │Trash  │
  └──┬─────┘  ├────┬────┤  └───┬──────┘ └───────┘  └───────┘
     │        │    │    │      │
  ┌──v─────┐ │    │  ┌──v─────┐
  │Twilio  │ │    │  │Logging │
  │SMS     │ │    │  │Archive │
  │Alert   │ │    │  └────────┘
  └────────┘ │    │
             │  ┌─v────┐
             │  │Asana  │
             │  │Task   │
             │  └───────┘
             │
    ┌────────┴──────────┐
    │ Parallel Output   │
    │ (Both execute)    │
    └───────────────────┘
```

## Node Processing Details

### Node 1: IMAP Email Trigger
```
Type: n8n-nodes-imap.imap (v1)
Position: [250, 350]
Configuration:
  - mailbox: "INBOX"
  - postProcessing: "mark" (mark as read after retrieval)
  - allowSelfSignedCerts: true
  - polling: ~1 minute (configurable)

Credentials: imapConnection

Output Structure:
{
  id: string,
  subject: string,
  from: { name?: string, address: string },
  to: array<email>,
  cc?: array<email>,
  bcc?: array<email>,
  text: string,
  html?: string,
  date: ISO8601,
  inReplyTo?: string,
  messageId: string,
  attachments?: array<{filename, content, contentType}>
}
```

### Node 2: Extract Email Content
```
Type: n8n-nodes-base.code (JavaScript, v2)
Position: [550, 350]

Process:
1. Receive raw email from IMAP
2. Normalize sender info (handle missing name)
3. Prefer HTML or text (fallback logic)
4. Extract all metadata
5. Preserve raw for audit trail

Output Structure:
{
  subject: string,
  body: string (HTML or plaintext),
  sender: string (email@domain.com),
  senderName: string,
  date: ISO8601,
  messageId: string,
  inReplyTo?: string,
  cc: array<email>,
  bcc: array<email>,
  attachmentCount: number,
  isHtml: boolean,
  rawEmail: {original email object}
}
```

### Node 3: Classify Email (OpenAI)
```
Type: n8n-nodes-base.openai (v4)
Position: [850, 350]
Model: gpt-4-turbo (configurable)
Temperature: 0.3 (low variance)

Prompt (Dynamic):
"You are an expert email triage assistant. Analyze the following email
and classify it into exactly ONE of these categories:

Categories:
1. URGENT_ACTION - Requires immediate response or action, time-sensitive
2. INFORMATIONAL - FYI, newsletters, updates (no response needed)
3. CLIENT_REQUEST - A client/customer asking for something
4. INTERNAL - Internal team communication, meetings
5. SPAM - Junk, promotional, unsubscribe requests

Email Subject: [subject]
Email Body: [first 2000 chars]

Respond in JSON format:
{
  \"classification\": \"CATEGORY_NAME\",
  \"confidence\": 0.0-1.0,
  \"reasoning\": \"brief explanation\"
}"

Input Variables:
- subject: from previous node
- body: first 2000 characters to prevent API bloat
- structured JSON output

Output:
{
  message: [{content: "...json..."}],
  response: string (JSON)
}
```

### Node 4: Parse Classification
```
Type: n8n-nodes-base.code (JavaScript, v2)
Position: [1150, 350]

Process:
1. Receive OpenAI response
2. Extract JSON from response (regex match)
3. Validate classification value
4. Fallback to text pattern matching if JSON invalid
5. Merge with previous email data
6. Ensure all fields present

Fallback Logic:
- If JSON parse fails, scan response for category keywords
- Assigns confidence 0.7 for pattern matches
- Returns full reasoning string on error

Output:
{
  ...previousEmailData,
  classification: "URGENT_ACTION" | "CLIENT_REQUEST" | ... | "SPAM",
  classificationConfidence: 0-1,
  classificationReasoning: string
}
```

### Node 5: Extract Key Data Points (OpenAI)
```
Type: n8n-nodes-base.openai (v4)
Position: [1150, 550]
Temperature: 0.2 (very consistent)

Prompt (Dynamic):
"Extract key data points from this email in JSON format:

Email Subject: [subject]
Email Body: [first 2000 chars]
Classification: [classification from previous]

Return ONLY valid JSON (no markdown):
{
  \"actionItems\": [\"item1\", \"item2\"],
  \"deadlines\": [\"deadline1\"],
  \"mentionedPeople\": [\"name1\"],
  \"priorityLevel\": \"HIGH/MEDIUM/LOW\",
  \"sender_category\": \"internal/external/vip\",
  \"requiredResponse\": true/false,
  \"estimatedTimeToRespond\": \"min\"
}"

Output:
{
  message: [{content: "...json..."}],
  response: string (JSON)
}
```

### Node 6: Parse Extracted Data
```
Type: n8n-nodes-base.code (JavaScript, v2)
Position: [1450, 550]

Process:
1. Receive OpenAI extraction response
2. Parse JSON response
3. Apply defaults for missing fields
4. Validate array structures
5. Merge all previous data
6. Return complete enriched object

Defaults Applied:
- actionItems: []
- deadlines: []
- mentionedPeople: []
- priorityLevel: "MEDIUM"
- sender_category: "external"
- requiredResponse: false
- estimatedTimeToRespond: "0"

Output: Complete enriched email object
{
  ...allPreviousData,
  actionItems: string[],
  deadlines: string[],
  mentionedPeople: string[],
  priorityLevel: string,
  sender_category: string,
  requiredResponse: boolean,
  estimatedTimeToRespond: string
}
```

### Node 7: Switch Router
```
Type: n8n-nodes-base.switch (v1)
Position: [1750, 200]

Field to Match: "classification"
Branches:
  ├─ "URGENT_ACTION" → output: "urgent"
  ├─ "CLIENT_REQUEST" → output: "client"
  ├─ "INFORMATIONAL" → output: "informational"
  ├─ "INTERNAL" → output: "internal"
  └─ "SPAM" → output: "spam"

Logic: Matches classification field against branches
Output: Routes to appropriate output nodes
```

### Output Nodes: Slack Messages (3 instances)

```
Type: n8n-nodes-base.slack (v3)
Instances:
  1. "Slack - Urgent Channel" [1a2b3c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d]
  2. "Slack - Client Requests" [5f6a7b8c-9d0e-1f2a-3b4c-5d6e7f8a9b0c]
  3. "Slack - Team Channel" [9d0e1f2a-3b4c-5d6e-7f8a-9b0c1d2e3f4a]

Template (Urgent):
"🚨 URGENT EMAIL ALERT

*From:* [Sender Name] ([email])
*Subject:* [Subject]
*Priority:* [Priority Level]
*Action Items:* [Items or 'None identified']
*Deadlines:* [Deadlines or 'None specified']

_Message ID: [messageId]_"

Similar templates for other channels with appropriate emojis.
```

### Output Node: SMS Alert (Twilio)

```
Type: n8n-nodes-base.twilio (v2)
Position: [2100, 250]

Configuration:
  - phoneNumber: $env.ADMIN_PHONE_NUMBER
  - message: "URGENT EMAIL ALERT: [Subject] from [Sender Name]"
  - from: SMS_TWILIO_FROM
  - credentials: twilioApi

Trigger: Only executes on URGENT_ACTION classification
```

### Output Node: Create Project Task (Asana)

```
Type: n8n-nodes-base.asana (v4)
Position: [2100, 550]

Configuration:
  - docId: $env.PROJECT_MGMT_DOC_ID
  - resource: "task"
  - operation: "create"
  - title: "[CLIENT] [Email Subject]"
  - description: "Email from: [sender]\n\nAction Items:\n[items as bullets]"
  - status: "[HIGH → pending, else → backlog]"

Trigger: Only executes on CLIENT_REQUEST classification
```

### Output Node: Google Sheets Logging

```
Type: n8n-nodes-base.googleSheets (v4)
Position: [2100, 850]
Preceding Node: "Format Logging Data" [3b4c5d6e-7f8a-9b0c-1d2e-3f4a5b6c7d8e]

Configuration:
  - documentId: $env.LOGGING_SHEET_ID
  - range: "A:F"
  - keyRow: 0 (use first row as headers)

Format Node (JS):
Prepares data object:
{
  timestamp: ISO8601,
  sender: email,
  senderName: name,
  subject: subject,
  classification: classification,
  priority: priorityLevel,
  actionItems: "item1; item2; ...",
  deadlines: "deadline1; deadline2; ...",
  messageId: messageId
}

Appends as new row to sheet.
```

### Output Node: Move to Trash

```
Type: n8n-nodes-base.imap (v1)
Position: [2100, 1000]

Configuration:
  - mailbox: "Trash"
  - markAsRead: false
  - moveToFolder: "Trash"

Trigger: Only executes on SPAM classification
```

## Error Handling Architecture

```
┌────────────────────────────────────────────────────────┐
│            ERROR HANDLING NODES                        │
├────────────────────────────────────────────────────────┤
│                                                        │
│ Node: "Error Handler - Slack"                         │
│ ├─ Type: n8n-nodes-base.slack                         │
│ ├─ Channel: #automation-logs                          │
│ ├─ Message: "❌ ERROR in Email Triage Workflow"       │
│ ├─ Content: Error node, message, email subject        │
│ └─ Emoji: :warning:                                   │
│                                                        │
│ Node: "Log Error to External Service"                 │
│ ├─ Type: n8n-nodes-base.webhook                       │
│ ├─ Method: HTTP POST (configurable)                   │
│ ├─ Payload: {error, workflow, timestamp, email_subject}
│ └─ Endpoint: (configure for your monitoring)          │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Performance Characteristics

### API Calls per Email
```
1. OpenAI Classification: 1 call (~$0.01)
2. OpenAI Data Extraction: 1 call (~0.01-0.02)
3. Slack Message: 0-3 calls (depending on route)
4. Asana Task: 0-1 calls
5. Google Sheets: 0-1 calls
6. IMAP Operations: 1 call

Total per email: 2-4 API calls
Estimated cost: $0.02-0.04 per email
Processing time: 10-30 seconds per email
```

### Throughput Capacity
```
Single Instance:
- IMAP polling: 1 check per minute (configurable)
- Sequential processing: ~60 emails/hour
- Concurrent: Limited by n8n instance resources

Scaling:
- Multiple workflows on same instance
- Separate email accounts per workflow
- Distributed n8n deployments for high volume
```

## Security Architecture

```
┌────────────────────────────────────────────────────────┐
│         SECURITY & PRIVACY LAYERS                      │
├────────────────────────────────────────────────────────┤
│                                                        │
│ CREDENTIAL STORAGE:                                   │
│ ├─ Encrypted at rest in n8n database                  │
│ ├─ Never logged in execution history                  │
│ ├─ Isolated per credential type                       │
│ └─ Rotatable/refreshable via UI                       │
│                                                        │
│ EMAIL DATA HANDLING:                                  │
│ ├─ Body limited to 2000 chars for AI (privacy)        │
│ ├─ Raw email archived (configurable retention)        │
│ ├─ Attachment content NOT processed                   │
│ ├─ Message ID used for deduplication                  │
│ └─ Audit trail in Google Sheets                       │
│                                                        │
│ API COMMUNICATION:                                    │
│ ├─ HTTPS/TLS for all outbound calls                   │
│ ├─ API keys never in logs or execution history        │
│ ├─ OAuth 2.0 for Google and Slack                     │
│ ├─ No credential sharing between workflows            │
│ └─ Rate limiting to prevent abuse                     │
│                                                        │
│ ENVIRONMENT VARIABLES:                                │
│ ├─ Phone number masked in logs ($env.*)               │
│ ├─ Project IDs not exposed in messages                │
│ ├─ Sensitive data isolated in .env                    │
│ └─ Separate prod/staging configs                      │
│                                                        │
└────────────────────────────────────────────────────────┘
```

## Scalability Considerations

### Horizontal Scaling
```
Option 1: Multiple Workflows
├─ One workflow per email account
├─ Separate classification rules per workflow
└─ Distributed processing via webhook triggers

Option 2: Distributed n8n
├─ Multiple n8n instances
├─ Shared workflow definitions
├─ Load balanced API calls
└─ Centralized logging database

Option 3: Webhook-based
├─ Replace IMAP trigger with HTTP
├─ Email server webhooks → n8n
├─ Real-time processing
└─ Automatic batching
```

### Optimization Points
```
1. Classification Model
   ├─ Batch multiple emails per API call
   ├─ Cache results for identical subjects
   └─ Consider local/self-hosted model

2. Data Storage
   ├─ Move from Google Sheets to database
   ├─ Implement data retention policies
   └─ Archive old emails to cold storage

3. Routing
   ├─ Add Redis cache for recent classifications
   ├─ Implement deduplication by message ID
   └─ Parallel branch execution

4. Cost Optimization
   ├─ Use cheaper GPT-3.5-Turbo for classification
   ├─ Batch operations where possible
   └─ Monitor API usage and adjust
```

---

**Document Version:** 1.0
**Last Updated:** 2026-02-21
**Compatibility:** n8n v0.200+
