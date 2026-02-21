# Email Triage Agent - Deployment Checklist

Use this checklist to ensure successful deployment to production.

---

## Pre-Deployment (1-2 weeks before go-live)

### Planning Phase
- [ ] Identify stakeholders and their email needs
- [ ] Map current manual email triage process
- [ ] Define email categories for organization
- [ ] Identify output channels (Slack, tasks, etc.)
- [ ] Plan budget and timeline
- [ ] Assign deployment owner/champion
- [ ] Schedule kickoff meeting

### Infrastructure Preparation
- [ ] Verify n8n instance is production-ready
- [ ] Confirm uptime SLA and backup procedures
- [ ] Plan monitoring and alerting setup
- [ ] Identify backup/disaster recovery approach
- [ ] Ensure sufficient API quota for providers

### Credential & Access Preparation
- [ ] Request admin access to email account (if needed)
- [ ] Create dedicated service account (if possible)
- [ ] Generate OpenAI API key with GPT-4 access
- [ ] Create Slack app with proper scopes
- [ ] Collect Asana/project management access
- [ ] Set up Google Sheets with proper sharing
- [ ] Prepare Twilio account (if SMS needed)

### Documentation Preparation
- [ ] Customize README.md for organization
- [ ] Prepare runbook for team
- [ ] Create org-specific CONFIGURATION_EXAMPLES.md
- [ ] Prepare training deck for users
- [ ] Create FAQ/troubleshooting guide
- [ ] Document escalation procedures

---

## Setup Phase (1 week)

### Import and Configuration
- [ ] Import workflow.json into n8n
- [ ] Name workflow appropriately
- [ ] Verify all 16+ nodes present
- [ ] Test all connections visible

### Credential Configuration (See SETUP_GUIDE.md)
- [ ] IMAP Email Trigger
  - [ ] Host, port, username configured
  - [ ] App password obtained and valid
  - [ ] Connection test successful
  
- [ ] OpenAI Credentials (2 needed)
  - [ ] API key valid and has GPT-4 access
  - [ ] Both "Classify" and "Extract" nodes configured
  - [ ] API test successful
  
- [ ] Slack Configuration
  - [ ] App created at api.slack.com
  - [ ] Bot token obtained (xoxb-*)
  - [ ] Scopes: chat:write, chat:write.public configured
  - [ ] All 4 channels created (#urgent, #client-requests, #team, #automation-logs)
  - [ ] Bot added to all channels
  - [ ] Slack credential saved in n8n
  
- [ ] Twilio Configuration (if using SMS)
  - [ ] Account SID and Auth Token obtained
  - [ ] Phone number verified
  - [ ] Credential saved in n8n
  
- [ ] Asana Configuration (if using tasks)
  - [ ] Personal Access Token generated
  - [ ] Project ID identified
  - [ ] Credential saved in n8n
  
- [ ] Google Sheets Configuration (if using logging)
  - [ ] Sheet created with headers
  - [ ] Sheet ID identified
  - [ ] OAuth authorization completed
  - [ ] Credential saved in n8n

### Environment Variables
- [ ] ADMIN_PHONE_NUMBER set (with country code)
- [ ] PROJECT_MGMT_DOC_ID set (if using Asana)
- [ ] LOGGING_SHEET_ID set (if using Sheets)
- [ ] SMS_TWILIO_FROM set (if using Twilio)
- [ ] All variables tested and accessible

### Workflow Customization
- [ ] Review AI classification prompt
- [ ] Adjust categories if needed for organization
- [ ] Verify data extraction fields match needs
- [ ] Review Slack message templates
- [ ] Update channel names to match org structure
- [ ] Customize Asana project settings
- [ ] Configure logging sheet columns

---

## Testing Phase (3-5 days)

### Unit Testing (Individual Nodes)
- [ ] Test IMAP trigger
  - [ ] Connection established
  - [ ] Can retrieve emails
  - [ ] Mark as read working
  
- [ ] Test Extract Content node
  - [ ] Properly normalizes email metadata
  - [ ] Handles both HTML and plaintext
  - [ ] Preserves sender information
  
- [ ] Test Classify Email node
  - [ ] Returns valid JSON response
  - [ ] Classification in expected categories
  - [ ] Confidence score reasonable
  
- [ ] Test Extract Data node
  - [ ] Identifies action items
  - [ ] Recognizes deadlines
  - [ ] Extracts mentioned people
  
- [ ] Test Switch Router
  - [ ] Routes to correct branch per category
  - [ ] All 5 branches functional

### Integration Testing
- [ ] End-to-end URGENT_ACTION flow
  - [ ] Email received
  - [ ] Classified correctly
  - [ ] Slack message posted to #urgent
  - [ ] SMS alert sent (if configured)
  
- [ ] End-to-end CLIENT_REQUEST flow
  - [ ] Email received
  - [ ] Slack message posted to #client-requests
  - [ ] Task created in Asana (if configured)
  
- [ ] End-to-end INFORMATIONAL flow
  - [ ] Email logged to Google Sheet (if configured)
  
- [ ] End-to-end INTERNAL flow
  - [ ] Slack message posted to #team
  
- [ ] End-to-end SPAM flow
  - [ ] Email moved to trash
  - [ ] No false alerts

### Load Testing
- [ ] Send 10 sequential emails (different categories)
- [ ] Monitor processing time per email
- [ ] Check no emails missed during burst
- [ ] Verify no duplicate processing
- [ ] Review API rate limits not exceeded

### Error Handling Testing
- [ ] Disconnect OpenAI, verify error handling
- [ ] Disconnect Slack, verify error handling
- [ ] Send malformed email, verify handling
- [ ] Verify error notifications to #automation-logs
- [ ] Verify workflow continues after error

### Accuracy Validation
- [ ] Sample 20-30 test emails
- [ ] Manually verify classifications
- [ ] Score accuracy (target: >90%)
- [ ] Identify misclassification patterns
- [ ] Adjust prompts if needed

---

## UAT Phase (3-5 days)

### User Acceptance Testing
- [ ] Have subject matter experts test
- [ ] One from each department/team
- [ ] Collect feedback on categories
- [ ] Verify channel assignments make sense
- [ ] Test Slack notification format clarity
- [ ] Test task creation in project management
- [ ] Gather feedback on accuracy

### Feedback & Adjustment
- [ ] Document all feedback
- [ ] Prioritize changes
- [ ] Implement critical adjustments
- [ ] Re-test impacted flows
- [ ] Get stakeholder sign-off

### Performance Validation
- [ ] Run for 2-3 days continuously
- [ ] Monitor execution history
- [ ] Check error rates (target: <1%)
- [ ] Verify no timeout issues
- [ ] Confirm API costs reasonable
- [ ] Test with realistic email volume

### Documentation Review
- [ ] Finalize runbook for team
- [ ] Create quick reference guides
- [ ] Prepare training materials
- [ ] Document custom configurations
- [ ] Prepare FAQ with answers

---

## Go-Live Phase (Day 1)

### Pre-Go-Live Checklist
- [ ] All tests passed
- [ ] Stakeholder sign-off obtained
- [ ] Team trained on workflow
- [ ] Support team briefed
- [ ] Escalation procedures documented
- [ ] Monitoring set up
- [ ] Backups verified
- [ ] Rollback plan prepared

### Deployment Steps
- [ ] Backup current configuration
- [ ] Activate workflow (turn on toggle)
- [ ] Verify first emails processed
- [ ] Check Slack messages are posting
- [ ] Monitor execution history
- [ ] Verify no errors in first hour

### Initial Monitoring (Day 1)
- [ ] Check every 30 minutes for first 2 hours
- [ ] Monitor execution history
- [ ] Check #automation-logs for errors
- [ ] Spot-check Slack messages
- [ ] Confirm task creation (if applicable)
- [ ] Monitor API usage
- [ ] Watch for unusual patterns

### Documentation Finalization
- [ ] Update Go-Live date in docs
- [ ] Document any deployment notes
- [ ] Update runbook with live settings
- [ ] Communicate status to stakeholders

---

## Post-Go-Live Phase (1-4 weeks)

### Week 1: Intensive Monitoring
- [ ] Daily check of execution history
- [ ] Review all Slack messages for accuracy
- [ ] Monitor #automation-logs religiously
- [ ] Track any issues immediately
- [ ] Quick-response to urgent issues
- [ ] Daily team check-in
- [ ] Collect and document user feedback

### Week 2: Stabilization
- [ ] Reduce monitoring frequency to 2x daily
- [ ] Review classification accuracy (sample 30 emails)
- [ ] Analyze any error patterns
- [ ] Fine-tune AI prompts if needed
- [ ] Address any user concerns
- [ ] Document any process changes
- [ ] Prepare weekly metrics report

### Week 3-4: Optimization
- [ ] Full accuracy review (100+ emails)
- [ ] Analyze cost and optimization opportunities
- [ ] Update team on performance
- [ ] Plan any additional features
- [ ] Document lessons learned
- [ ] Conduct retrospective meeting

### Monthly Maintenance (Ongoing)
- [ ] Schedule recurring monthly review
- [ ] Set up monthly accuracy audit (20-30 emails)
- [ ] Review error logs and patterns
- [ ] Monitor API usage and costs
- [ ] Check for any model updates
- [ ] Team feedback survey
- [ ] Plan optimizations for next month

---

## Success Criteria

### Deployment Success
- [ ] All 16 nodes functioning correctly
- [ ] All integrations connected and working
- [ ] Zero critical errors in first week
- [ ] Email processing rate: 100%
- [ ] Average processing time: <30 seconds
- [ ] Classification accuracy: >90%
- [ ] Team comfortable with workflow

### Operational Success
- [ ] Uptime: 99%+ availability
- [ ] Error rate: <1% of processed emails
- [ ] False positive rate: <5%
- [ ] API costs within budget
- [ ] Response time SLAs met
- [ ] Team independently operating workflow
- [ ] Documentation complete and accurate

### Business Success
- [ ] Time savings: >2 hours/day (or organizational goal)
- [ ] Improved email organization
- [ ] Faster response to urgent emails
- [ ] Better tracking of client requests
- [ ] Team satisfaction: >4/5 rating
- [ ] Measurable ROI
- [ ] Scalable for growth

---

## Rollback Procedures

### If Critical Issues Discovered

**Immediate Action:**
1. Deactivate workflow (turn off toggle)
2. Document issue clearly
3. Notify stakeholders
4. Return to manual process

**Investigation:**
1. Review execution history for error patterns
2. Check logs for specific failure point
3. Isolate problematic node/configuration
4. Prepare fix

**Reactivation:**
1. Implement fix
2. Test in safe environment
3. Monitor closely on reactivation
4. Have team on standby

---

## Escalation Matrix

```
Issue Severity | Response Time | Escalation | Action
─────────────────────────────────────────────────────
P0 - Critical  | Immediate     | Manager    | Deactivate
(No emails     |               |            | & investigate
processing)    |               |            |

P1 - Major     | 30 minutes    | Team Lead  | Isolate &
(Category      |               |            | fix node
routing wrong) |               |            |

P2 - Moderate  | 2 hours       | Owner      | Monitor &
(Accuracy      |               |            | adjust
issues)        |               |            |

P3 - Minor     | Next day      | None       | Log & plan
(UI formatting)|               |            | for next
                |               |            | release
```

---

## Sign-Off

### Technical Approval
- [ ] Deployment lead: _________________ Date: _____
- [ ] DevOps/Infrastructure: _________________ Date: _____

### Business Approval
- [ ] Project sponsor: _________________ Date: _____
- [ ] Department head: _________________ Date: _____

### Support & Operations
- [ ] Support team trained: _________________ Date: _____
- [ ] Runbook approved: _________________ Date: _____
- [ ] Escalation confirmed: _________________ Date: _____

---

## Post-Implementation Review (30 days after go-live)

- [ ] Schedule retrospective meeting
- [ ] Review metrics and KPIs
- [ ] Collect team feedback
- [ ] Document lessons learned
- [ ] Plan improvements for next phase
- [ ] Update this checklist for future deployments
- [ ] Archive all deployment documentation

---

**Deployment Status: _________________ (In Progress / Complete / Rolled Back)**

**Deployment Date: _________________**

**Last Updated: _________________**

**Deployment Owner: _________________**

---

For questions during deployment, reference:
- SETUP_GUIDE.md - Step-by-step instructions
- README.md - Complete documentation
- QUICK_REFERENCE.md - Fast troubleshooting
- ARCHITECTURE.md - Technical details
