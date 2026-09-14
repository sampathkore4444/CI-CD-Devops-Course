# 84. Incident Communication and Escalation Process

## Scenario

At 9:30 AM, a P1 incident is declared — the banking application is unavailable. Customers are calling support (50+ calls in 10 minutes). Executive leadership (CEO, CTO, CFO) wants updates every 15 minutes. The engineering team is debugging the issue. The PR team is monitoring social media for customer complaints. The legal team is concerned about regulatory implications. You need to structure the incident management process including incident commander, communication, status pages, and escalation paths.

## Interviewer Question

"A P1 incident is declared. Customers are calling, executives want updates, engineering is debugging. How do you structure incident management including roles, communication, status pages, and escalation?"

## What I Should Think About

- Incident Commander (IC) role and responsibilities
- Communication Lead role
- Technical Lead role
- Status page updates (internal and external)
- Stakeholder communication (executives, customers, support)
- Escalation paths (L1 → L2 → L3 → L4)
- Incident timeline and documentation
- Postmortem process
- Blameless culture

## Ideal Answer

**1. Incident Roles**
- **Incident Commander (IC)**: Owns the incident, coordinates response, makes decisions
- **Technical Lead (TL)**: Leads debugging, coordinates engineering effort
- **Communication Lead (CL)**: Manages all communications, status page, stakeholder updates
- **scribe**: Documents timeline, decisions, actions

**2. Communication Cadence**
- **Internal**: Slack #incident channel, every 15 minutes
- **External**: Status page, every 30 minutes
- **Executives**: Email/Slack DM, every 15 minutes
- **Support**: Slack #support-escalations, real-time
- **Social Media**: Twitter/Reddit monitoring, real-time

**3. Escalation Path**
- **L1 (0-5 min)**: On-call engineer acknowledges, initial triage
- **L2 (5-15 min)**: Senior engineer joins, deeper investigation
- **L3 (15-30 min)**: Engineering manager, resource coordination
- **L4 (30+ min)**: VP Engineering, executive communication

**4. Status Page Updates**
- Initial: "Investigating issue with banking application"
- Update 1: "Identified root cause, working on fix"
- Update 2: "Implementing fix, ETA 30 minutes"
- Update 3: "Service restored, monitoring"

**5. Postmortem**
- Blameless postmortem within 48 hours
- Timeline, root cause, impact, action items
- Follow-up on action items

## Architecture

```
┌─────────────────────────────────────────────────┐
│        INCIDENT MANAGEMENT STRUCTURE              │
│                                                   │
│  ┌──────────────────────────────────────────┐   │
│  │           INCIDENT COMMANDER               │   │
│  │  (Owns incident, makes decisions)         │   │
│  └─────────────────────┬────────────────────┘   │
│                        │                          │
│         ┌──────────────┼──────────────┐          │
│         ▼              ▼              ▼          │
│  ┌────────────┐ ┌────────────┐ ┌────────────┐  │
│  │ Technical  │ │Comms Lead  │ │  Scribe    │  │
│  │ Lead       │ │            │ │            │  │
│  │ (Debugging)│ │(Status,    │ │(Timeline,  │  │
│  │            │ │ Stakeholder│ │ Decisions) │  │
│  │            │ │ Updates)   │ │            │  │
│  └────────────┘ └────────────┘ └────────────┘  │
│                                                   │
│  COMMUNICATION CHANNELS:                         │
│  ┌──────────────────────────────────────────┐   │
│  │ #incident (Slack)     → Internal team     │   │
│  │ status.company.com    → External/public   │   │
│  │ #support-escalations  → Support team      │   │
│  │ exec-updates@company  → Leadership        │   │
│  │ @company/status       → Social media      │   │
│  └──────────────────────────────────────────┘   │
│                                                   │
│  ESCALATION PATH:                                │
│  L1 (0-5m) → L2 (5-15m) → L3 (15-30m) → L4 (30m+)│
│  On-call     Senior Eng    Eng Manager    VP Eng   │
└─────────────────────────────────────────────────┘
```

## Investigation

1. **Set up incident channel:**
   ```bash
   # Create Slack channel
   curl -s -X POST "https://slack.com/api/conversations.create" \
     -H "Authorization: Bearer xoxb-xxx" \
     -H "Content-Type: application/json" \
     -d '{
       "name": "incident-2024-01-15-banking-down",
       "purpose": "P1 incident - Banking application unavailable"
     }'
   
   # Invite key personnel
   curl -s -X POST "https://slack.com/api/conversations.invite" \
     -H "Authorization: Bearer xoxb-xxx" \
     -H "Content-Type: application/json" \
     -d '{
       "channel": "C INCIDENT-2024-01-15-BANKING-DOWN",
       "users": "U_ONCALL,U_SENIOR_ENG,U_ENG_MANAGER,U_VP_ENG"
     }'
   ```

2. **Update status page:**
   ```bash
   # Create incident on Statuspage
   curl -s -X POST "https://api.statuspage.io/v1/pages/{page_id}/incidents" \
     -H "Authorization: Bearer {api_key}" \
     -H "Content-Type: application/json" \
     -d '{
       "incident": {
         "name": "Banking Application Unavailable",
         "status": "investigating",
         "body": "We are investigating reports of banking application being unavailable. Engineering team is actively working on the issue.",
         "component_ids": ["COMPONENT_BANKING_APP"],
         "impact_override": "major"
       }
     }'
   ```

3. **Document timeline:**
   ```markdown
   # Incident Timeline - 2024-01-15
   
   ## 9:30 AM - Incident Declared
   - Monitoring alert: "Banking application HTTP 500 errors > 5%"
   - IC assigned: @on-call-engineer
   - TL assigned: @senior-engineer
   
   ## 9:35 AM - Initial Triage
   - Confirmed: Application returning 500 errors
   - Affected: All banking API endpoints
   - Impact: 100% of users affected
   
   ## 9:40 AM - Root Cause Identified
   - Database connection pool exhaustion
   - 100% of connections in use
   - Long-running queries blocking pool
   
   ## 9:45 AM - Fix Implemented
   - Killed long-running queries
   - Restarted connection pool
   - Verified: Connections dropping to normal
   
   ## 9:50 AM - Service Restored
   - HTTP 500 errors: 0%
   - Application responding normally
   - Monitoring for recurrence
   ```

## Commands

```bash
# 1. Create incident runbook
cat > incident-runbook.yml << 'EOF'
name: "P1 Incident Response"
roles:
  incident_commander:
    responsibilities:
      - "Own the incident"
      - "Make decisions"
      - "Coordinate response"
    escalation: "VP Engineering (after 30 minutes)"
  
  technical_lead:
    responsibilities:
      - "Lead debugging"
      - "Coordinate engineering"
      - "Implement fixes"
    escalation: "Multiple senior engineers (after 15 minutes)"
  
  communication_lead:
    responsibilities:
      - "Status page updates"
      - "Stakeholder communication"
      - "Support coordination"
    escalation: "PR team (for external communication)"
  
  scribe:
    responsibilities:
      - "Document timeline"
      - "Record decisions"
      - "Track action items"

communication_cadence:
  internal: "Every 15 minutes in #incident channel"
  external: "Every 30 minutes on status page"
  executives: "Every 15 minutes via email/Slack"
  support: "Real-time in #support-escalations"

escalation_path:
  l1:
    time: "0-5 minutes"
    personnel: "On-call engineer"
    actions: "Acknowledge, initial triage"
  l2:
    time: "5-15 minutes"
    personnel: "Senior engineer"
    actions: "Deeper investigation"
  l3:
    time: "15-30 minutes"
    personnel: "Engineering manager"
    actions: "Resource coordination"
  l4:
    time: "30+ minutes"
    personnel: "VP Engineering"
    actions: "Executive communication"
EOF

# 2. Automate status page updates
cat > status-update.sh << 'EOF'
#!/bin/bash
INCIDENT_ID=$1
STATUS=$2
MESSAGE=$3

curl -s -X PATCH "https://api.statuspage.io/v1/pages/{page_id}/incidents/$INCIDENT_ID" \
  -H "Authorization: Bearer {api_key}" \
  -H "Content-Type: application/json" \
  -d "{
    \"incident\": {
      \"status\": \"$STATUS\",
      \"body\": \"$MESSAGE\"
    }
  }"
EOF

# 3. Send executive update
cat > exec-update.sh << 'EOF'
#!/bin/bash
SUBJECT="P1 Incident Update - $(date '+%H:%M')"
BODY=$1

curl -s -X POST "https://slack.com/api/chat.postMessage" \
  -H "Authorization: Bearer xoxb-xxx" \
  -H "Content-Type: application/json" \
  -d "{
    \"channel\": \"C_EXECS\",
    \"text\": \"$SUBJECT\\n\\n$BODY\"
  }"
EOF

# 4. Create postmortem template
cat > postmortem-template.md << 'EOF'
# Postmortem: [Incident Title]

## Incident Summary
- **Date**: [Date]
- **Duration**: [Start time] - [End time] ([Duration])
- **Severity**: P1/P2/P3
- **Impact**: [Number] users affected, [Revenue] impact

## Timeline
| Time | Event | Person |
|------|-------|--------|
| [Time] | [Event] | [Person] |

## Root Cause
[Description of root cause]

## What Went Well
- [Thing 1]
- [Thing 2]

## What Went Wrong
- [Thing 1]
- [Thing 2]

## Action Items
| Action | Owner | Due Date | Status |
|--------|-------|----------|--------|
| [Action] | [Owner] | [Date] | [Status] |

## Lessons Learned
[Key takeaways]
EOF
```

## Root Cause

| Root Cause | Elimination |
|---|---|
| No defined incident roles | Establish IC, TL, CL, Scribe roles |
| No communication cadence | Define 15-minute internal, 30-minute external |
| No status page | Implement Statuspage or similar |
| No escalation path | Define L1-L4 escalation with time thresholds |
| No postmortem process | Implement blameless postmortem within 48 hours |
| No incident documentation | Automate timeline and decision logging |

## Immediate Mitigation

1. **Assign incident roles** — IC, TL, CL, Scribe
2. **Create incident channel** — dedicated Slack channel
3. **Update status page** — initial "Investigating" message
4. **Notify executives** — initial status update
5. **Document timeline** — start recording events

## Permanent Fix

1. Implement incident response playbook
2. Create incident roles and responsibilities document
3. Automate status page updates
4. Implement escalation policies
5. Regular incident response drills
6. Blameless postmortem culture

## Monitoring

- **Incident response time** — time to acknowledge, triage, resolve
- **Communication effectiveness** — stakeholder satisfaction
- **Postmortem completion rate** — % of incidents with postmortem
- **Action item completion rate** — % of postmortem actions completed
- **Incident recurrence rate** — % of incidents that recur

## Security

- Incident channels should be private
- Sensitive incident details should not be in public status page
- Access to incident data should be restricted
- Postmortems should be confidential

## Production Considerations

- **50+ support calls** — need dedicated support liaison
- **Executive updates every 15 minutes** — need automated reporting
- **Social media monitoring** — need PR team coordination
- **Regulatory implications** — need legal team involvement
- **Customer communication** — need templated messages

## Senior-Level Answer

"I'd implement a structured incident management process: (1) Roles — IC owns the incident, TL leads debugging, CL manages communication, Scribe documents, (2) Communication — 15-minute internal cadence, 30-minute external updates, executive notifications, (3) Escalation — L1 (0-5 min), L2 (5-15 min), L3 (15-30 min), L4 (30+ min), (4) Documentation — automated timeline and decision logging, (5) Postmortem — blameless postmortem within 48 hours with actionable follow-ups."

## Architect-Level Answer

"At the organizational level, I'd establish: (1) Incident Management Platform — standardized tooling (PagerDuty, Statuspage, Slack integration), (2) Incident Response Training — regular drills and simulation exercises, (3) Communication Standards — templates, cadence, escalation paths, (4) Blameless Culture — postmortems focused on systemic improvement, not individual blame, (5) Continuous Improvement — track incident metrics, identify patterns, implement systemic fixes."

## Follow-Up Questions

1. "How do you handle incident communication across multiple time zones?"
2. "What's the difference between incident commander and technical lead roles?"
3. "How do you implement automated incident detection and response?"
4. "How do you handle incidents that span multiple teams or services?"
5. "What's your approach to incident response training and drills?"
