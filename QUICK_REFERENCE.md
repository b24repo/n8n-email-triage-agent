# Email Triage Agent - Quick Reference Card

Print this card and keep it handy during setup and operation.

---

## Setup Checklist (30 minutes)

### 1. Prepare Credentials (5 min)
```
□ Email address & app password
□ OpenAI API key (GPT-4 access)
□ Slack workspace name
□ Twilio account (optional)
□ Asana token (optional)
□ Google Sheets ID (optional)
```

### 2. Import Workflow (5 min)
```
□ Open n8n → Workflows → Import from File
□ Select workflow.json
□ Click Import
□ Name: "Email Triage Agent - MVP3"
□ Click Save
```

### 3. Configure IMAP (5 min)
```
IMAP Email Trigger node:
□ Click node → Credentials → Create New
□ Host: imap.gmail.com (or your provider)
□ Port: 993
□ Username: your-email@gmail.com
□ Password: [app password]
□ SSL: ✓ Enabled
□ Click Save
```

### 4. Configure OpenAI (5 min)
```
Classify Email & Extract Key Data Points nodes:
□ Click node → OpenAI credential → Create New
□ Paste API key (sk-...)
□ Click Save
□ Repeat for both OpenAI nodes
```

### 5. Configure Slack (5 min)
```
All Slack nodes:
□ Create Slack App at api.slack.com
□ Add scopes: chat:write, chat:write.public, files:write
□ Copy Bot Token (xoxb-...)
□ Click Slack credential → Create New → Paste token
□ Create channels: #urgent, #client-requests, #team, #automation-logs
□ Add bot to channels: /invite @Email Triage Bot
```

### 6. Set Environment Variables (3 min)
```
Settings → Environment Variables:
□ ADMIN_PHONE_NUMBER = "+1234567890"
□ PROJECT_MGMT_DOC_ID = "[asana-id]"
□ LOGGING_SHEET_ID = "[google-sheet-id]"
□ SMS_TWILIO_FROM = "[twilio-number]"
```

### 7. Activate & Test (2 min)
```
□ Click blue Activate toggle
□ Send test email to monitored inbox
□ Check execution history
□ Verify message in Slack #urgent (or appropriate channel)
```

---

## Workflow Node Reference

| # | Node Name | Type | Purpose |
|---|-----------|------|---------|
| 1 | IMAP Email Trigger | Trigger | Monitor inbox |
| 2 | Extract Email Content | Code | Normalize email |
| 3 | Classify Email | OpenAI | AI classification |
| 4 | Parse Classification | Code | Validate result |
| 5 | Extract Key Data Points | OpenAI | Extract details |
| 6 | Parse Extracted Data | Code | Validate details |
| 7 | Switch Router | Switch | Route by category |
| 8 | Slack - Urgent Channel | Slack | Send to #urgent |
| 9 | SMS Alert (Twilio) | Twilio | Send SMS |
| 10 | Slack - Client Requests | Slack | Send to #client-requests |
| 11 | Create Project Task | Asana | Create task |
| 12 | Log to Google Sheet | Sheets | Archive email |
| 13 | Slack - Team Channel | Slack | Send to #team |
| 14 | Format Logging Data | Code | Prepare logging |
| 15 | Move to Trash | IMAP | Delete spam |
| 16 | Error Handler - Slack | Slack | Log errors |
| 17 | Log Error to External | Webhook | External logging |

---

## Email Categories

```
URGENT_ACTION
├─ What: Needs immediate response
├─ Triggers: Slack #urgent + SMS alert
├─ Example: "Server down", "P1 bug", "VIP client"
└─ Response Time: <1 hour

CLIENT_REQUEST
├─ What: Customer/client asking something
├─ Triggers: Slack #client-requests + Asana task
├─ Example: "Feature request", "Support ticket", "Quote needed"
└─ Response Time: <24 hours

INTERNAL
├─ What: Team communication
├─ Triggers: Slack #team channel
├─ Example: "Team standup", "Meeting", "Policy update"
└─ Response Time: N/A (FYI)

INFORMATIONAL
├─ What: For your information only
├─ Triggers: Google Sheets archive only
├─ Example: "Newsletter", "Status page", "Vendor update"
└─ Response Time: N/A

SPAM
├─ What: Junk/promotional/unsolicited
├─ Triggers: Move to Trash
├─ Example: "Cold sales", "Marketing promo", "Spam"
└─ Response Time: None (deleted)
```

---

## Slack Message Examples

### URGENT_ACTION
```
🚨 URGENT EMAIL ALERT

From: John Smith (john@client.com)
Subject: Server is down - customer impact
Priority: HIGH
Action Items: Immediate investigation, Customer notification
Deadlines: None specified
```

### CLIENT_REQUEST
```
📋 CLIENT REQUEST

From: Sarah Johnson (sarah@company.com)
Subject: Can we add dark mode to dashboard?
Priority: MEDIUM
Action Items: Evaluate feature, Cost/benefit analysis
```

### INTERNAL
```
📬 INTERNAL UPDATE

From: Mike Davis
Subject: Team standup moved to 2pm today
```

---

## Common Configuration Changes

### Change Slack Channel
```
Node: [Any Slack node]
Parameter: channel
Old: "#urgent"
New: "#critical-alerts"
```

### Add New Email Category
```
1. Edit "Classify Email" node → Add to prompt
2. Edit "Switch Router" node → Add new branch
3. Create output node for new category
4. Connect switch → new output node
```

### Use Cheaper AI Model
```
Nodes: "Classify Email" & "Extract Key Data Points"
Change: model: "gpt-4-turbo" → "gpt-3.5-turbo"
Cost: -50% per email
Speed: +30% faster
```

### Disable SMS Alerts
```
Node: "SMS Alert (Twilio)"
Option 1: Delete the node
Option 2: Disconnect from Switch Router
(Slack alert continues)
```

### Change Email Body Limit
```
Node: "Classify Email"
Parameter: Prompt text "substring(0, 2000)"
Change 2000 to: 1000 (shorter) or 3000 (longer)
```

---

## Troubleshooting Flowchart

```
WORKFLOW NOT STARTING?
  ├─ Is "Activate" toggle ON?
  │  └─ No → Click toggle
  │
  └─ Check IMAP connection
     ├─ Test button on IMAP node
     └─ If fails → Check credentials

EMAIL NOT TRIGGERING?
  ├─ Check IMAP polling
  │  └─ Wait 1-2 minutes for next poll
  │
  ├─ Send test email
  │  └─ Check inbox for it
  │
  └─ Check execution history
     └─ Look for errors

CLASSIFICATIONS WRONG?
  ├─ Review AI prompts
  │  └─ Edit "Classify Email" node
  │
  ├─ Add training examples to prompt
  │  └─ Include real email examples
  │
  └─ Increase temperature for variety
     └─ Change 0.3 → 0.5

SLACK MESSAGES NOT SENDING?
  ├─ Is bot member of channel?
  │  └─ Type: /invite @Email Triage Bot
  │
  ├─ Does channel exist?
  │  └─ Create it if missing
  │
  └─ Check bot token
     └─ Paste new token if expired

OPENAI API ERRORS?
  ├─ Check quota at platform.openai.com
  │  └─ Add billing if needed
  │
  ├─ Verify API key
  │  └─ Create new key if expired
  │
  └─ Check timeout
     └─ Increase to 60000ms in node settings
```

---

## Environment Variables Explained

```
ADMIN_PHONE_NUMBER
├─ Format: "+1-555-123-4567" (with country code)
├─ Used by: SMS Alert (Twilio)
├─ Required for: Urgent email SMS notifications
└─ Example: "+14155552671"

PROJECT_MGMT_DOC_ID
├─ Format: Asana project ID (numeric string)
├─ Find in: Asana project URL /projects/XXXXX/
├─ Used by: Create Project Task node
├─ Required for: Client request task creation
└─ Example: "1234567890123456"

LOGGING_SHEET_ID
├─ Format: Google Sheet ID (string)
├─ Find in: Sheet URL /spreadsheets/d/XXXXX/
├─ Used by: Log to Google Sheet node
├─ Required for: Email archiving/audit trail
└─ Example: "1abc2def3ghi4jkl5mno6pqr7stu8vwx"

SMS_TWILIO_FROM
├─ Format: "+1-555-123-4567" (Twilio phone number)
├─ Find in: Twilio console (phone numbers)
├─ Used by: SMS Alert node
├─ Required for: Sending SMS alerts
└─ Example: "+14155552671"
```

---

## Performance Tips

### Reduce Cost
```
1. Use gpt-3.5-turbo (-50% cost)
2. Disable data extraction (+30% speed)
3. Limit email body to 1000 chars
4. Batch emails for classification
5. Cache results for common subjects
```

### Improve Speed
```
1. Reduce timeout values (30000 → 20000)
2. Skip unnecessary parsing
3. Remove unused integrations
4. Use gpt-3.5-turbo (faster model)
5. Optimize regex patterns in Code nodes
```

### Increase Accuracy
```
1. Add training examples to prompts
2. Lower temperature (0.3 → 0.1)
3. Add organization-specific keywords
4. Include context in classification prompt
5. Review and adjust monthly
```

---

## Monthly Maintenance Tasks

```
Week 1: Monitoring
□ Check #automation-logs for errors
□ Review execution success rate
□ Monitor API usage and costs

Week 2: Accuracy Review
□ Sample 20-30 emails
□ Score classification accuracy
□ Note misclassified emails

Week 3: Optimization
□ Analyze error patterns
□ Adjust prompts if needed
□ Test configuration changes

Week 4: Planning
□ Team feedback survey
□ Plan next month improvements
□ Document changes
□ Schedule monthly review
```

---

## Emergency Procedures

### Workflow Not Processing (All Stuck)
```
1. Check n8n status dashboard
2. Restart n8n service if needed
3. Check error logs
4. Manually re-trigger by sending email
5. If still stuck, deactivate and reactivate
```

### Too Many Errors (>10%)
```
1. Deactivate workflow (turn off toggle)
2. Review last 10 executions for patterns
3. Check OpenAI API status
4. Verify Slack token still valid
5. Fix issues
6. Reactivate when ready
```

### Missing Important Emails
```
1. Check Spam filter didn't catch them
2. Verify IMAP connection is active
3. Check execution history for gaps
4. Manually move to INBOX if in Spam
5. Monitor next 24 hours for similar issues
```

---

## Contact & Resources

### n8n Support
- **Documentation:** https://docs.n8n.io
- **Community Forum:** https://community.n8n.io
- **Issues:** https://github.com/n8n-io/n8n

### API Documentation
- **OpenAI:** https://platform.openai.com/docs
- **Slack:** https://api.slack.com
- **Asana:** https://developers.asana.com
- **Google Sheets:** https://developers.google.com/sheets
- **Twilio:** https://www.twilio.com/docs

### This Package
- **Start Setup:** SETUP_GUIDE.md
- **Full Docs:** README.md
- **Technical:** ARCHITECTURE.md
- **Examples:** CONFIGURATION_EXAMPLES.md

---

## Key Metrics to Monitor

### Daily
- [ ] Workflow execution status (green = success)
- [ ] Error rate (target: <1%)
- [ ] Processing time per email (target: <30 sec)

### Weekly
- [ ] Total emails processed
- [ ] Distribution by category
- [ ] Error patterns
- [ ] API costs

### Monthly
- [ ] Classification accuracy (sample review)
- [ ] False positive rate (misclassifications)
- [ ] User satisfaction
- [ ] Cost per email
- [ ] Total savings vs. manual

---

## Credentials Checklist

```
□ IMAP (Email)
  └─ Host, Port, Username, Password, SSL enabled

□ OpenAI (2 copies needed)
  └─ API Key with GPT-4 access

□ Slack
  └─ Bot Token with chat:write scopes

□ Twilio (optional)
  └─ Account SID, Auth Token, Phone number

□ Asana (optional)
  └─ Personal Access Token, Project ID

□ Google Sheets (optional)
  └─ OAuth authorization, Sheet ID
```

---

## Quick Fixes

| Problem | Fix | Time |
|---------|-----|------|
| Workflow won't activate | Check IMAP connection, restart n8n | 2 min |
| Slack messages blank | Check node template syntax | 5 min |
| OpenAI timeouts | Increase timeout to 60000ms | 2 min |
| Asana task creation fails | Verify project ID, check token | 5 min |
| Spam emails not deleting | Verify IMAP move-to-trash support | 5 min |
| Wrong Slack channel | Edit channel name in node | 2 min |
| Missing action items | Review extraction prompt, lower temp | 10 min |
| High costs | Switch to gpt-3.5-turbo model | 5 min |

---

## Success Indicators

```
✓ Email workflow activated and running
✓ Emails received and processed within 1 minute
✓ Classification matches expected categories >90% of time
✓ Slack messages formatted and posting correctly
✓ No errors in execution history
✓ Monthly costs within budget ($0.50 - $200+)
✓ Team comfortable with categorization
✓ Response times meeting organizational SLAs
```

---

**Print this card and tape it to your monitor during setup!**

For complete information, see the full documentation files.
