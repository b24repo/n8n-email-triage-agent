# Email Triage Agent - Quick Setup Guide

Complete this guide in approximately 30 minutes to get the workflow running.

## Quick Checklist

- [ ] Step 1: Prepare Credentials
- [ ] Step 2: Import Workflow
- [ ] Step 3: Configure IMAP
- [ ] Step 4: Configure OpenAI
- [ ] Step 5: Configure Slack
- [ ] Step 6: Configure Optional Services
- [ ] Step 7: Set Environment Variables
- [ ] Step 8: Test & Activate

---

## Step 1: Prepare Credentials

### Collect Required Information

Before starting in n8n, gather these credentials:

#### Email (IMAP)
- [ ] Email address: `_______________________`
- [ ] IMAP Server: `_______________________`
- [ ] App Password (Gmail) or regular password: `_______________________`
- [ ] IMAP Port: `993` (default)

#### OpenAI
- [ ] API Key: `sk-_______________________`
- [ ] Verify GPT-4-Turbo access at https://platform.openai.com/account/billing/overview

#### Slack
- [ ] Workspace name: `_______________________`
- [ ] Admin user account ready for OAuth

#### Twilio (Optional)
- [ ] Account SID: `_______________________`
- [ ] Auth Token: `_______________________`
- [ ] Phone Number: `_______________________`

#### Asana (Optional)
- [ ] Personal Access Token: `_______________________`
- [ ] Project ID or Workspace ID: `_______________________`

#### Google Sheets (Optional)
- [ ] Sheet ID (from URL): `_______________________`
- [ ] Column headers setup: Subject, Sender, Classification, etc.

---

## Step 2: Import Workflow

1. Open n8n instance (http://localhost:5678 or your domain)
2. Click **Workflows** in left sidebar
3. Click **Create New** → **Import from File**
4. Select `workflow.json` from this package
5. Click **Import**
6. Name: `Email Triage Agent - MVP3`
7. Click **Save** and wait for import to complete

**Expected:** Workflow appears with 16 nodes and connections visible

---

## Step 3: Configure IMAP

### For Gmail Users

1. **Enable 2-Factor Authentication:**
   - Visit https://myaccount.google.com/security
   - Enable 2-Step Verification (if not already enabled)

2. **Generate App Password:**
   - Visit https://myaccount.google.com/apppasswords
   - Select "Mail" and "Windows Computer"
   - Copy the generated 16-character password

3. **Create IMAP Credential in n8n:**
   - In workflow, click **IMAP Email Trigger** node
   - Click "IMAP" under Credentials
   - Click "Create New Credential"
   - Fill in:
     ```
     Host: imap.gmail.com
     Port: 993
     Username: your-email@gmail.com
     Password: [16-char app password from step 2]
     SSL: ✓ Enabled
     TLS: ✓ Enabled
     ```
   - Click **Save**

### For Other Email Providers

| Provider | Host | Port |
|----------|------|------|
| Outlook/Office365 | imap-mail.outlook.com | 993 |
| Apple iCloud | imap.mail.me.com | 993 |
| Yahoo | imap.mail.yahoo.com | 993 |
| Fastmail | imap.fastmail.com | 993 |
| Custom | [Your provider] | 993 |

**Test IMAP Connection:**
1. Click **IMAP Email Trigger** node
2. Click **Test Connection** (if available)
3. Should see "Connection successful"

---

## Step 4: Configure OpenAI

1. Get your API key:
   - Visit https://platform.openai.com/api/keys
   - Create new secret key if needed
   - Copy the key (starts with `sk-`)

2. Add credential in n8n:
   - Click any **OpenAI** node (there are 2: "Classify Email" and "Extract Key Data Points")
   - Under Credentials, click "Create New"
   - Select "OpenAI"
   - Paste API key
   - Click **Save**

3. Verify model access:
   - Check https://platform.openai.com/account/billing/usage
   - Ensure account has GPT-4 Turbo access (or switch model to gpt-3.5-turbo if needed)

**Cost Estimate:**
- Classification: ~0.01-0.02 per email
- Data extraction: ~0.01-0.02 per email
- **Total: ~0.02-0.04 per email**

---

## Step 5: Configure Slack

1. **Create Slack App:**
   - Visit https://api.slack.com/apps
   - Click **Create New App**
   - Choose "From scratch"
   - App name: `Email Triage Bot`
   - Pick your workspace
   - Click **Create App**

2. **Set Permissions:**
   - Go to **OAuth & Permissions** (left sidebar)
   - Under "Scopes" → "Bot Token Scopes", add:
     - `chat:write`
     - `chat:write.public`
     - `files:write`
   - Click **Save Scopes**

3. **Install App:**
   - Click **Install to Workspace** (top of page)
   - Authorize permissions
   - Copy **Bot User OAuth Token** (starts with `xoxb-`)

4. **Create Channels** (if not exists):
   ```
   #urgent
   #client-requests
   #team
   #automation-logs
   ```

5. **Add Bot to Channels:**
   - For each channel, type: `/invite @Email Triage Bot`

6. **Add Slack Credential to n8n:**
   - Click any **Slack** node
   - Under Credentials, click "Create New"
   - Select "Slack"
   - Authentication method: "Custom OAuth Credentials"
   - **Client ID:** From your Slack App page (Basic Information)
   - **Client Secret:** From your Slack App page (Basic Information)
   - Click **Connect** and authorize
   - Or paste Bot Token directly if preferred

---

## Step 6: Configure Optional Services

### Twilio (For SMS Alerts on Urgent Emails)

1. Create Twilio account: https://www.twilio.com/try-twilio
2. Get credentials from Twilio Console:
   - Account SID
   - Auth Token
   - Verified phone number
3. In n8n, click **SMS Alert (Twilio)** node
4. Create new Twilio credential with SID and Token
5. Update phone number in **Node:** SMS Alert settings
   - Set from: Your Twilio number
   - Set to: `$env.ADMIN_PHONE_NUMBER`

### Asana (For Task Creation from Client Requests)

1. Create Asana account: https://asana.com
2. Generate Personal Access Token:
   - Account Settings → Apps → Personal Access Tokens
   - Create token, copy it
3. Create a project for incoming requests
   - Note the Project ID from URL: `/projects/XXXXXXX/`
4. In n8n, click **Create Project Task** node
5. Create new Asana credential with token
6. Set project ID in node parameters

### Google Sheets (For Logging/Archive)

1. Create new Google Sheet at https://sheets.google.com
2. Add headers in row 1:
   ```
   Timestamp | Sender | Sender Name | Subject | Classification | Priority | Action Items | Deadlines | Message ID
   ```
3. Get Sheet ID from URL: `/spreadsheets/d/XXXXXXX/`
4. Share sheet with your Google account (if not already owner)
5. In n8n, click **Log to Google Sheet** node
6. Create new Google Sheets credential:
   - Click **Sign in with Google**
   - Authorize n8n to access sheets
7. Set Sheet ID in node parameters

---

## Step 7: Set Environment Variables

1. In n8n, go to **Settings** (gear icon, bottom left)
2. Click **Environment Variables**
3. Add these variables:

```
Name: ADMIN_PHONE_NUMBER
Value: +1234567890
Type: String
Note: Phone number for urgent SMS alerts (include country code)

Name: PROJECT_MGMT_DOC_ID
Value: [Your Asana Project ID or doc ID]
Type: String
Note: ID for creating tasks from client requests

Name: LOGGING_SHEET_ID
Value: [Your Google Sheet ID]
Type: String
Note: ID of the logging sheet for audit trail

Name: SMS_TWILIO_FROM
Value: +1234567890
Type: String
Note: Your Twilio phone number
```

---

## Step 8: Test & Activate

### Pre-Activation Tests

1. **Test IMAP Connection:**
   - Click **IMAP Email Trigger** node
   - Click **Test** button
   - Should show "No messages found" or list recent emails

2. **Test OpenAI:**
   - Click **Classify Email** node
   - Click **Test** with sample email
   - Should return classification JSON

3. **Test Slack:**
   - Click any **Slack** node
   - Click **Test**
   - Should post test message to configured channel

### Activate Workflow

1. Click blue **Activate** toggle (top right)
2. Workflow is now live and monitoring inbox
3. Watch workflow executions in real-time on **Executions** tab

### Send Test Emails

Send emails to your configured inbox with different subjects to test routing:

| Subject | Expected Result |
|---------|-----------------|
| "URGENT: Server down" | → #urgent + SMS alert |
| "Can you provide a quote for X?" | → #client-requests + Asana task |
| "Team lunch on Friday" | → #team |
| "Great deals on widgets!" | → Trash |
| "FYI: New office policy" | → Google Sheet log |

---

## Troubleshooting Quick Fixes

### "Authentication failed for IMAP"
- [ ] Verify email address and password/app password
- [ ] Confirm IMAP is enabled in email account settings
- [ ] Check firewall isn't blocking port 993

### "OpenAI API error"
- [ ] Verify API key is valid (not expired)
- [ ] Check account has sufficient credit
- [ ] Confirm GPT-4 access (or switch to gpt-3.5-turbo)

### "Slack message failed"
- [ ] Verify bot is member of channel (try `/invite @bot`)
- [ ] Check bot token hasn't been rotated
- [ ] Confirm channel exists and name is correct

### "Workflow not triggering"
- [ ] Click **Activate** toggle to turn workflow on
- [ ] Check IMAP node for connection issues
- [ ] Send test email and check executions tab
- [ ] Verify email arrives in configured mailbox

### "Classification returning empty"
- [ ] Check OpenAI API is responding (test in node)
- [ ] Increase OpenAI timeout (node settings → 60000ms)
- [ ] Review API quota at platform.openai.com

---

## Next Steps

1. **Monitor First Week:**
   - Let workflow run for 5-7 days
   - Review classification accuracy (target: 95%+)
   - Adjust AI prompts if needed

2. **Customize Classifications:**
   - Add your organization-specific categories
   - Update Slack channel names
   - Add/remove integrations as needed

3. **Set Performance Alerts:**
   - Configure error notifications
   - Monitor API usage and costs
   - Set up weekly accuracy reviews

4. **Plan Scaling:**
   - Test with 100+ emails
   - Optimize slow steps
   - Consider database logging instead of sheets

---

## Support Resources

- **n8n Docs:** https://docs.n8n.io
- **n8n Community:** https://community.n8n.io
- **OpenAI API Docs:** https://platform.openai.com/docs
- **Slack API Docs:** https://api.slack.com
- **Workflow Troubleshooting:** See README.md Troubleshooting section

---

## Configuration Summary Checklist

```
Credentials Configured:
[ ] IMAP Email Connection
[ ] OpenAI API
[ ] Slack Workspace
[ ] Twilio (optional)
[ ] Asana (optional)
[ ] Google Sheets (optional)

Environment Variables Set:
[ ] ADMIN_PHONE_NUMBER
[ ] PROJECT_MGMT_DOC_ID
[ ] LOGGING_SHEET_ID
[ ] SMS_TWILIO_FROM

Slack Setup:
[ ] Channels created (#urgent, #client-requests, #team, #automation-logs)
[ ] Bot added to all channels
[ ] OAuth token obtained

Ready for Activation:
[ ] All credentials tested
[ ] All environment variables set
[ ] Slack bot has proper permissions
[ ] Test email can be received
```

---

**Setup Time Estimate:** 20-30 minutes
**Difficulty Level:** Intermediate
**Support:** See README.md for detailed documentation
