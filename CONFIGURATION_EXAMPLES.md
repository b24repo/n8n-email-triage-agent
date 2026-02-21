# Email Triage Agent - Configuration Examples

## Common Configuration Scenarios

This document provides real-world configuration examples for different organizational needs.

---

## Scenario 1: Startup with Basic Triage

**Use Case:** Small startup that needs basic email filtering into urgent/non-urgent

### Email Categories
- URGENT_ACTION: Server alerts, critical customer issues, founder emails
- CLIENT_REQUEST: Customer inquiries, sales leads
- INFORMATIONAL: All other emails (log only)
- SPAM: Automatically delete

### Modified AI Prompt
```javascript
// In "Classify Email" node, replace prompt with:

const prompt = `You are an email triage assistant for a startup.
Classify emails into ONE category based on urgency and action required:

URGENT_ACTION: Server down, critical customer issue, security alert,
              CEO/founder email, payment issue
CLIENT_REQUEST: Customer asking for feature, demo, pricing, support
INFORMATIONAL: Newsletters, status updates, FYI messages, notifications
SPAM: Marketing, unsubscribe, unsolicited offers

Email Subject: ${subject}
Email Body: ${body}

Return JSON: {"classification": "...", "confidence": 0-1}`;
```

### Slack Channel Setup
```
Create channels:
#urgent        (for URGENT_ACTION)
#customers     (for CLIENT_REQUEST)
#inbox-log     (optional, for archiving)
```

### Integration Stack
```
IMAP → Classification → Slack routing only
(No Asana, No SMS, No Google Sheets)
```

### Environment Variables
```env
ADMIN_PHONE_NUMBER=""  # Leave empty, SMS disabled
PROJECT_MGMT_DOC_ID="" # Leave empty, Asana disabled
LOGGING_SHEET_ID=""    # Leave empty, logging disabled
SMS_TWILIO_FROM=""     # Leave empty
```

### Cost Optimization
```
- Use gpt-3.5-turbo instead of gpt-4-turbo
- Disable data extraction step (remove "Extract Key Data Points" node)
- Batch 10 emails per classification call
```

---

## Scenario 2: Enterprise SaaS with Full Features

**Use Case:** Medium SaaS company with customer support, sales, and internal teams

### Email Categories (Extended)
- **URGENT_ACTION**: Escalated support, system alerts, VIP customer, revenue-impacting
- **CLIENT_REQUEST**: Feature requests, bug reports, billing inquiries, support tickets
- **INTERNAL**: Team updates, meetings, policy changes, all-hands
- **INFORMATIONAL**: Newsletters, vendor communications, FYI
- **SPAM**: Promotional, unsubscribe, marketing

### Advanced AI Prompt
```javascript
const prompt = `You are an expert email triage system for a SaaS company.

URGENT_ACTION rules:
- Production down/critical alerts
- VIP customer (CEO, key account)
- Payment/revenue impacting
- Security/compliance issue
- Escalated support ticket
- Deadline within 24 hours

CLIENT_REQUEST rules:
- Feature request from customer
- Bug report/support ticket
- Billing/invoice question
- Demo or trial request
- Integration question
- Data export/compliance request

INTERNAL rules:
- Team standup, all-hands
- Internal process, policy
- Company announcement
- HR/recruitment
- Finance/accounting

INFORMATIONAL:
- Industry newsletter
- Vendor update/outage notice
- Promotional (not spam)
- Status page update

SPAM:
- Cold sales pitch
- Unsubscribe request
- Spam/malware attempt
- Duplicate promo

Subject: ${subject}
Body: ${body.substring(0, 2000)}

Return JSON format only:
{
  "classification": "URGENT_ACTION|CLIENT_REQUEST|INTERNAL|INFORMATIONAL|SPAM",
  "confidence": 0.0-1.0,
  "reasoning": "brief explanation",
  "vip_indicator": true/false,
  "revenue_impact": true/false
}`;
```

### Slack Channels & Routing
```
#urgent                (URGENT_ACTION + SMS to ops lead)
#customer-success      (CLIENT_REQUEST)
#support-backlog       (CLIENT_REQUEST - lower priority)
#engineering-alerts    (URGENT_ACTION - system alerts only)
#team-updates          (INTERNAL)
#general-inbox         (INFORMATIONAL)
#spam-archive          (audit trail)
#automation-logs       (errors)
```

### Project Management
```
Asana Project: "Customer Requests"
├─ CLIENT_REQUEST emails → Create tasks
│  ├─ Title: "[CUSTOMER] [Subject]"
│  ├─ Priority: HIGH if urgent, MEDIUM/LOW otherwise
│  ├─ Section: By customer/category
│  └─ Assignee: Auto-assign based on sender
│
└─ URGENT_ACTION → Also create in "Critical Issues" project
```

### Google Sheets Logging
```
Main Log Sheet:
├─ Timestamp, Sender, Subject, Category, Priority
├─ Action Items, People Mentioned, Deadlines
├─ Confidence Score, Processing Time
└─ Response Sent (manual update)

VIP Customer Sheet:
├─ Track all VIP communications
├─ Response time SLA monitoring
└─ Monthly summary report

Statistics Dashboard:
├─ Volume by category (trend analysis)
├─ Average response time
├─ Misclassification rate (manual review)
└─ Cost analysis by source
```

### Environment Variables
```env
ADMIN_PHONE_NUMBER="+1-555-OPS-LEAD"
PROJECT_MGMT_DOC_ID="asana-project-12345678"
LOGGING_SHEET_ID="1abc2def3ghi4jkl5mno6pqr7stu8vwx"
SMS_TWILIO_FROM="+1-555-SMS-BOT"
VIP_EMAILS_SHEET="1zyx9wvu8tsr7qpo5nml4kji3hgf2edc"
```

---

## Scenario 3: Customer Support Team

**Use Case:** Dedicated support team with ticketing and SLA monitoring

### Simplified Categories
```
Categories:
- URGENT_ACTION: P1/P2 issues, SLA breach imminent
- CLIENT_REQUEST: New support requests
- INFORMATIONAL: Customer updates, confirmations
- SPAM: Junk/promotional
- (Remove INTERNAL category)
```

### Modified Extraction Prompt
```javascript
const prompt = `As a support email analyzer, extract:

Subject: ${subject}
Body: ${body}

Return JSON:
{
  "classification": "URGENT_ACTION|CLIENT_REQUEST|INFORMATIONAL|SPAM",
  "severity": "P0|P1|P2|P3|P4",
  "actionItems": ["action1", "action2"],
  "deadlines": ["deadline"],
  "customerId": "if mentioned",
  "issueType": "bug|feature|billing|general|other",
  "affectedProduct": "product name if mentioned",
  "sla_hours": 4-24,
  "customerStatus": "free|paid|enterprise|vip"
}`;
```

### Slack Channels
```
#support-incoming      (All new requests)
#support-urgent        (URGENT_ACTION only)
#support-critical      (P0 only - alerts every 15 min if unresponded)
```

### Ticketing Integration
Replace Asana with:
- **Zendesk**: Auto-create tickets from CLIENT_REQUEST emails
- **HubSpot Service Hub**: Route to correct team queue
- **Jira Service Management**: Track issue resolution

### SLA Monitoring
```
Add node: "Check SLA Status"
- Calculate hours since email receipt
- If > threshold, escalate to #support-urgent
- Add @-mention for urgent emails
- Track response time per support agent
```

### Metrics Dashboard
```
Google Sheets dashboard:
├─ Response time (target: <2 hours)
├─ Resolution time (target: <24 hours)
├─ First-response rate
├─ Customer satisfaction
├─ Agent performance (emails per agent)
└─ Issue category breakdown
```

---

## Scenario 4: Enterprise with Multiple Departments

**Use Case:** Large organization where different departments handle different email types

### Department-Based Routing

```
Sales Department:
├─ Email domain: sales-team@company.com
├─ Recipient filters: [sales@, enterprise@]
├─ Categories:
│  ├─ LEAD_INBOUND (prospect inquiry)
│  ├─ OPPORTUNITY (negotiation)
│  ├─ INTERNAL (team syncs)
│  └─ SPAM
└─ Routes: Slack #sales-leads + Salesforce opportunity creation

Support Department:
├─ Email domain: support@company.com
├─ Categories:
│  ├─ URGENT_ACTION (P1/P2 issues)
│  ├─ BUG_REPORT (technical issues)
│  ├─ FEATURE_REQUEST (enhancement)
│  ├─ BILLING (payment/account)
│  └─ SPAM
└─ Routes: Slack #support + Zendesk tickets

Engineering:
├─ Email domain: eng-team@company.com
├─ Categories:
│  ├─ CRITICAL_ALERT (system down)
│  ├─ SECURITY (vulnerability report)
│  ├─ CODE_REVIEW (PR/pull request)
│  ├─ INTERNAL (standup, meetings)
│  └─ SPAM
└─ Routes: Slack #engineering + GitHub notifications

Operations:
├─ Email domain: ops@company.com
├─ Categories:
│  ├─ INFRASTRUCTURE_ALERT
│  ├─ INCIDENT
│  ├─ COMPLIANCE
│  └─ SPAM
└─ Routes: Slack #ops + PagerDuty incidents
```

### Multi-Department Workflow Architecture

```
Create separate workflow for each department:
├─ email-triage-sales.json
├─ email-triage-support.json
├─ email-triage-engineering.json
└─ email-triage-operations.json

OR create conditional branches in main workflow:
├─ Read email recipient/to address
├─ Use Switch node to route to department logic
├─ Each branch has department-specific categorization
└─ Routes to department Slack channels
```

### Department-Specific Prompts

```javascript
// Sales department prompt
const salesPrompt = `Classify sales emails:
- LEAD_INBOUND: New prospect inquiry, RFP, demo request
- OPPORTUNITY: Existing deal progression, negotiation
- INTERNAL: Team meetings, forecasting, strategy
- SPAM: Unsolicited recruiting, marketing

Subject: ${subject}
Body: ${body}

Additional extraction:
- Company name (if prospect)
- Deal stage
- Budget indicator
- Decision timeframe`;

// Support department prompt
const supportPrompt = `Classify support emails:
- URGENT_ACTION: P1/P2 - system down, data loss, security
- BUG_REPORT: Product not working as expected
- FEATURE_REQUEST: Customer asking for enhancement
- BILLING: Invoice, payment, license question
- SPAM: Unsolicited, unsubscribe

Subject: ${subject}
Body: ${body}

Additional extraction:
- Customer tier (free/paid/enterprise)
- Product/component affected
- Severity/priority level
- Reproducibility`;
```

### Centralized Analytics

```
Master Google Sheet:
├─ All department emails logged
├─ Aggregate metrics dashboard
├─ Department performance comparison
├─ Cross-team trends
├─ Cost allocation by department
└─ SLA tracking by category
```

---

## Scenario 5: Compliance-Heavy Organization

**Use Case:** Financial services, healthcare, or legal firm with strict requirements

### Classification with Compliance Flag

```javascript
const compliancePrompt = `Classify email with compliance focus:

Categories:
- URGENT_ACTION: Regulatory request, compliance deadline, audit
- CLIENT_REQUEST: Customer request (log for compliance)
- INTERNAL: Internal memo (must be archivable)
- COMPLIANCE_REQUIRED: Requires documented handling
- LEGAL_HOLD: Do not delete, litigation-related
- SPAM: Marketing (safe to delete)

Subject: ${subject}
Body: ${body}

Return:
{
  "classification": "...",
  "compliance_level": "standard|elevated|critical",
  "legal_hold": false,
  "retention_period": "1-year|3-year|7-year|indefinite",
  "pii_detected": false,
  "encrypted_recommended": false,
  "audit_trail_required": true/false,
  "approval_required": true/false
}`;
```

### Data Handling & Retention

```
Never Delete:
├─ All URGENT_ACTION emails
├─ LEGAL_HOLD flagged emails
├─ Compliance-level: critical
└─ Emails with PII detected

Conditional Delete:
├─ SPAM: Delete after 30 days
├─ INFORMATIONAL: Archive after 90 days
└─ Depends on content/sender

Archive to:
├─ Cold storage after 1 year
├─ Encrypted backup (3-year minimum)
└─ Audit-logged retrieval
```

### Security Integration

```
Add nodes for:
├─ DLP (Data Loss Prevention) check
│  └─ Scan for PII/sensitive data
├─ Malware scanning
│  └─ Check attachments (if present)
├─ Encryption status
│  └─ Flag unencrypted sensitive content
└─ Audit logging
   └─ Every access recorded
```

### Approval Workflow

```
Add node: "Approval Gate"
├─ For COMPLIANCE_REQUIRED or LEGAL_HOLD emails
├─ Route to Slack #legal-review
├─ Request approval before responding
├─ Log approval decision
└─ Create audit trail

Approval message:
"⚠️ COMPLIANCE REVIEW REQUIRED
Email: [subject]
From: [sender]
Classification: [class]
Please review and respond: [Link to Asana task]"
```

---

## Scenario 6: Non-Profit with Limited Budget

**Use Case:** Non-profit organization wanting email automation without high costs

### Cost-Optimized Configuration

```
Use free/low-cost services:
├─ OpenAI: gpt-3.5-turbo (cheapest model)
├─ Slack: Free tier (1 app)
├─ Google Sheets: Free (logging)
├─ IMAP: Your email provider (free)
├─ Asana: Free tier (limited)
└─ n8n: Self-hosted (open source, free)
```

### Simplified Workflow

```
Remove expensive steps:
├─ ✗ Data extraction (second AI call) → Save $0.01-0.02 per email
├─ ✗ SMS alerts → Only urgent emails to Slack
├─ ✗ Task creation → Manual review from Slack instead
└─ ✗ Google Sheets → Use Slack thread for archiving

Keep essential:
├─ IMAP monitoring
├─ Single AI classification (gpt-3.5-turbo)
└─ Slack routing by category
```

### Alternative: Keyword-Based Classification

```
Instead of AI classification, use rules:

Urgent keywords:
├─ Server, down, critical, emergency, urgent, asap
├─ Action: → #urgent Slack channel

Client keywords:
├─ request, help, support, demo, question, issue, bug
├─ Action: → #requests Slack channel

Spam keywords:
├─ unsubscribe, promo, deal, discount, limited offer
├─ Action: → Delete/Archive

Fallback: → #general (manual review)
```

### Estimated Monthly Cost
```
With gpt-3.5-turbo (500 emails/month):
├─ Classification: 500 × $0.001 = $0.50
├─ OpenAI total: $0.50
├─ Slack: Free
├─ Google Sheets: Free
├─ n8n (self-hosted): Free
└─ TOTAL: ~$0.50/month
```

---

## Scenario 7: Real Estate Brokerage

**Use Case:** Specific domain customization for real estate business

### Custom Categories

```
Classifications:
├─ URGENT_ACTION
│  ├─ Inspection scheduled
│  ├─ Offer received
│  ├─ Closing date confirmed
│  └─ Document signature needed
│
├─ TRANSACTION
│  ├─ Listing update
│  ├─ Showing request
│  ├─ Client inquiry about property
│  └─ MLS update
│
├─ INTERNAL
│  ├─ Team communication
│  ├─ Market analysis
│  └─ Lead assignment
│
└─ SPAM
   ├─ Lead generation spam
   ├─ Market "tips"
   └─ Unsolicited vendor offers
```

### Custom AI Prompt

```javascript
const prompt = `Classify real estate email:

URGENT_ACTION: Offer on property, inspection scheduled,
               closing date set, documents need signature,
               appraisal issue, lender issue

TRANSACTION: New listing active, showing request,
             property inquiry from client,
             MLS update, price reduction, new photo

INTERNAL: Team standup, lead assignment, market report,
          commission split, contract review

SPAM: Unsolicited lead gen, broker spam, market tips,
      vendor solicitation, junk mail

Subject: ${subject}
Body: ${body}

Extract additional data:
- Property address (if mentioned)
- Client name (if mentioned)
- Transaction stage (pre-offer, offer, pending, closing)
- Dollar amount (if mentioned)
- Urgency level (hours, days, weeks)`;
```

### Workflow Customization

```
Additional nodes:
├─ Extract property address (regex pattern)
├─ Look up MLS# in database
├─ Calculate days-to-close
├─ Route by urgency + stage
└─ Log to transaction tracker

Routes:
├─ URGENT_ACTION → SMS alert to agent
├─ URGENT_ACTION → Slack #urgent + create follow-up task
├─ TRANSACTION → Slack #transactions + log to CRM
├─ INTERNAL → Slack #team
└─ SPAM → Delete
```

### CRM Integration

```
Instead of Asana, integrate with:
├─ Zillow/Trulia API
├─ MLS system API
└─ Real estate CRM (BoomTown, Follow Up Boss, etc.)

Create function:
├─ Log transaction to CRM
├─ Update property status
├─ Create follow-up task with deadline
└─ Link to client profile
```

---

## Custom Integration Template

Use this template to build integrations for other services:

```javascript
// Template for custom routing node
const classification = $input.first().json.classification;
const priority = $input.first().json.priorityLevel;
const actionItems = $input.first().json.actionItems;

return {
  json: {
    // Route to appropriate system
    shouldRouteToServiceA: classification === "URGENT_ACTION",
    shouldRouteToServiceB: classification === "CLIENT_REQUEST",
    shouldRouteToServiceC: classification === "INFORMATIONAL",

    // Data formatted for target service
    serviceA_payload: {
      urgency: priority === "HIGH" ? "CRITICAL" : "NORMAL",
      actionItems: actionItems,
      timestamp: new Date().toISOString()
    },

    serviceB_payload: {
      title: `[${classification}] ${$input.first().json.subject}`,
      description: actionItems.join("\n"),
      priority: priority === "HIGH" ? "P0" : "P1"
    },

    // Generic webhook format
    webhookPayload: {
      type: "email_triage",
      classification: classification,
      sender: $input.first().json.sender,
      subject: $input.first().json.subject,
      priority: priority,
      extractedData: {
        actions: actionItems,
        people: $input.first().json.mentionedPeople,
        deadlines: $input.first().json.deadlines
      }
    }
  }
};
```

---

## Migration Checklists

### From Manual Triage to Automated

```
Phase 1: Testing (Week 1-2)
- [ ] Set up workflow in development
- [ ] Configure IMAP and OpenAI credentials
- [ ] Test with 50-100 sample emails
- [ ] Review classification accuracy
- [ ] Adjust prompts if needed

Phase 2: Pilot (Week 3-4)
- [ ] Activate on subset of inbox
- [ ] Monitor for 1-2 weeks
- [ ] Track accuracy metrics
- [ ] Gather team feedback
- [ ] Optimize categories/routing

Phase 3: Full Rollout (Week 5+)
- [ ] Activate on full inbox
- [ ] Archive old emails (if applicable)
- [ ] Monitor daily for issues
- [ ] Monthly accuracy reviews
- [ ] Continuous optimization
```

### From One Tool to Multi-Tool Integration

```
Phase 1: Email + Slack (Week 1)
- [ ] Basic routing to Slack channels
- [ ] Test message formatting
- [ ] Verify bot permissions

Phase 2: Add Project Management (Week 2-3)
- [ ] Integrate Asana/Jira/Monday
- [ ] Map email categories to projects
- [ ] Set task creation rules

Phase 3: Add Analytics (Week 4)
- [ ] Add Google Sheets logging
- [ ] Create dashboard
- [ ] Set up automated reporting

Phase 4: Add Advanced Features (Week 5+)
- [ ] SMS alerts for urgent
- [ ] CRM integration
- [ ] Custom AI prompts per department
```

---

**Document Version:** 1.0
**Last Updated:** 2026-02-21
