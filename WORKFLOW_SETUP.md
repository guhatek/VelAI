# n8n Workflow Setup Guide

This guide will help you configure the n8n workflow for the SRE On-Call AI Agent.

## Overview

The workflow consists of several interconnected components:

1. **Webhook Triggers** - Receive alerts from monitoring systems
2. **Slack Event Listeners** - Monitor for bot mentions
3. **Vexa AI Processors** - Analyze and generate content
4. **Approval Workflow** - Route to admins for review
5. **Publication Nodes** - Send approved content to channels
6. **Schedulers** - Trigger periodic summaries

## Required Credentials

Before setting up the workflow, configure these credentials in n8n:

### 1. Slack API Credentials

```
Name: Slack Bot Token
Type: Slack API
Bot Token: xoxb-your-bot-token-here
```

**Required Scopes:**
- `app_mentions:read`
- `channels:history`
- `channels:read`
- `chat:write`
- `chat:write.public`
- `reactions:write`
- `users:read`

### 2. Vexa AI Credentials

```
Name: Vexa AI
Type: HTTP Header Auth
Header Name: Authorization
Header Value: Bearer YOUR_VEXA_API_KEY
```

### 3. Webhook Authentication (Optional)

```
Name: Webhook Auth
Type: Header Auth
Header Name: X-Webhook-Secret
Header Value: your-secret-key
```

## Workflow Nodes Configuration

### Node 1: Webhook Trigger (Alert Receiver)

**Type:** Webhook  
**Method:** POST  
**Path:** `/webhook/alerts`  
**Authentication:** Optional (recommended for production)

**Expected Payload:**
```json
{
  "alert_name": "string",
  "severity": "critical|high|medium|low",
  "service": "string",
  "timestamp": "ISO8601",
  "details": "string",
  "slack_channel": "optional_channel_id"
}
```

### Node 2: Slack Event Trigger (Bot Mentions)

**Type:** Slack Trigger  
**Event:** app_mention  
**Credential:** Slack Bot Token

**Configuration:**
- Listen for mentions in all channels where bot is added
- Extract thread_ts for thread context
- Capture channel_id and user_id

### Node 3: Route Incident Type

**Type:** Switch  
**Mode:** Rules

**Rules:**
1. `{{ $json.body.severity === 'critical' }}` → Critical Path
2. `{{ $json.body.severity === 'high' }}` → High Priority Path
3. `{{ $json.event.type === 'app_mention' }}` → Slack Mention Path
4. Default → Standard Path

### Node 4: Extract Context

**Type:** Code (JavaScript)

**Code:**
```javascript
// Extract and normalize incident context
const incident = {
  id: $input.item.json.body?.alert_id || Date.now().toString(),
  source: $input.item.json.source || 'slack',
  severity: $input.item.json.body?.severity || 'medium',
  service: $input.item.json.body?.service || 'unknown',
  channel: $input.item.json.body?.slack_channel || $input.item.json.event?.channel,
  thread_ts: $input.item.json.event?.thread_ts || $input.item.json.event?.ts,
  timestamp: new Date().toISOString(),
  context: {
    alert_details: $input.item.json.body?.details || '',
    user_mention: $input.item.json.event?.user || null
  }
};

// Store in workflow static data for later retrieval
$execution.customData.set('incident', incident);

return { json: incident };
```

### Node 5: Fetch Thread History (Slack)

**Type:** Slack  
**Operation:** Get Channel Messages  
**Credential:** Slack Bot Token

**Parameters:**
- Channel: `{{ $json.channel }}`
- Oldest: `{{ $json.thread_ts }}`
- Inclusive: `true`
- Thread TS: `{{ $json.thread_ts }}`

### Node 6: Call Vexa AI - Analyze Incident

**Type:** HTTP Request  
**Method:** POST  
**URL:** `https://<vexa host>/v1/chat/completions`  
**Authentication:** Vexa AI Credentials

**Body (JSON):**
```json
{
  "model": "vexa-latest",
  "messages": [
    {
      "role": "system",
      "content": "You are an SRE assistant analyzing incidents. Provide structured analysis including severity assessment, potential root cause, and recommended actions."
    },
    {
      "role": "user",
      "content": "Analyze this incident:\n\nService: {{ $json.service }}\nSeverity: {{ $json.severity }}\nDetails: {{ $json.context.alert_details }}\n\nThread Context:\n{{ $node['Fetch Thread History'].json.messages }}"
    }
  ],
  "temperature": 0.3
}
```

### Node 7: Post Initial Response (Slack)

**Type:** Slack  
**Operation:** Send Message  
**Credential:** Slack Bot Token

**Parameters:**
- Channel: `{{ $json.channel }}`
- Thread TS: `{{ $json.thread_ts }}`
- Text: 
```
🤖 SRE On-Call Agent activated for incident: *{{ $json.service }}*

Severity: {{ $json.severity }}
Status: Monitoring active
Next summary: 30 minutes

I'll be tracking this thread and providing periodic updates.
```

### Node 8: Schedule Trigger (Periodic Summaries)

**Type:** Schedule Trigger  
**Interval:** Every 30 minutes

**Trigger Rules:**
- Only run when active incidents exist
- Check workflow static data for incident list

### Node 9: Generate Summary (Vexa AI)

**Type:** HTTP Request  
**Method:** POST  
**URL:** `https://<vexa host>/v1/chat/completions`  
**Authentication:** Vexa AI Credentials

**Body (JSON):**
```json
{
  "model": "vexa-latest",
  "messages": [
    {
      "role": "system",
      "content": "You are an SRE assistant creating incident summaries. Be concise, factual, and structured. Include: current status, key updates since last summary, action items, and next steps."
    },
    {
      "role": "user",
      "content": "Create a summary for this ongoing incident:\n\nIncident ID: {{ $json.id }}\nService: {{ $json.service }}\nDuration: {{ $json.duration }}\n\nRecent messages:\n{{ $json.recent_messages }}\n\nProvide summary in this format:\n## Status\n[Current status]\n\n## Key Updates\n[Updates since last summary]\n\n## Action Items\n[Identified tasks]\n\n## Next Steps\n[Recommended actions]"
    }
  ],
  "temperature": 0.4
}
```

### Node 10: Extract Action Items (Code)

**Type:** Code (JavaScript)

**Code:**
```javascript
// Parse Vexa AI response for action items
const summary = $input.item.json.choices[0].message.content;
const actionItemsRegex = /## Action Items\n([\s\S]*?)(?=\n##|\n*$)/;
const match = summary.match(actionItemsRegex);

const actionItems = [];
if (match && match[1]) {
  const items = match[1].trim().split('\n').filter(line => line.trim());
  items.forEach(item => {
    actionItems.push({
      text: item.replace(/^[-*]\s*/, ''),
      status: 'pending',
      created_at: new Date().toISOString()
    });
  });
}

return {
  json: {
    summary: summary,
    action_items: actionItems,
    incident_id: $input.item.json.id
  }
};
```

### Node 11: Send to Admin for Review (Slack)

**Type:** Slack  
**Operation:** Send Message  
**Credential:** Slack Bot Token

**Parameters:**
- Channel: `{{ $env.ADMIN_CHANNEL_ID }}`
- Blocks (JSON):
```json
[
  {
    "type": "header",
    "text": {
      "type": "plain_text",
      "text": "📋 Incident Summary Review Required"
    }
  },
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "*Incident:* {{ $json.service }}\n*Status:* Ongoing\n*Duration:* {{ $json.duration }}"
    }
  },
  {
    "type": "divider"
  },
  {
    "type": "section",
    "text": {
      "type": "mrkdwn",
      "text": "{{ $json.summary }}"
    }
  },
  {
    "type": "actions",
    "elements": [
      {
        "type": "button",
        "text": {
          "type": "plain_text",
          "text": "✅ Approve & Publish"
        },
        "style": "primary",
        "value": "approve_{{ $json.incident_id }}",
        "action_id": "approve_summary"
      },
      {
        "type": "button",
        "text": {
          "type": "plain_text",
          "text": "✏️ Edit"
        },
        "value": "edit_{{ $json.incident_id }}",
        "action_id": "edit_summary"
      },
      {
        "type": "button",
        "text": {
          "type": "plain_text",
          "text": "❌ Reject"
        },
        "style": "danger",
        "value": "reject_{{ $json.incident_id }}",
        "action_id": "reject_summary"
      }
    ]
  }
]
```

### Node 12: Wait for Admin Approval

**Type:** Slack Trigger  
**Event:** interaction (Button Click)  
**Filter:** `action_id === 'approve_summary'`

### Node 13: Publish to Stakeholders (Slack)

**Type:** Slack  
**Operation:** Send Message  
**Credential:** Slack Bot Token

**Parameters:**
- Channel: `{{ $env.INCIDENT_NOTIFICATION_CHANNEL }}`
- Text:
```
📊 *Incident Update: {{ $json.service }}*

{{ $json.summary }}

_Last updated: {{ $now.format('HH:mm') }}_
_Status updates provided every 30 minutes_
```

### Node 14: Error Handler

**Type:** If  
**Condition:** `{{ $execution.mode === 'manual' || $runData.lastNodeExecuted?.error }}`

**True Branch:**
- Send error notification to admin channel
- Log error details
- Do not fail workflow

## Environment Variables Setup

Set these in your n8n instance:

```bash
# Vexa AI Configuration
VEXA_AI_API_KEY=your_api_key_here
VEXA_AI_ENDPOINT=https://<vexa host>/v1

# Slack Configuration
SLACK_BOT_TOKEN=xoxb-your-token
SLACK_SIGNING_SECRET=your_signing_secret
ADMIN_CHANNEL_ID=<channel id>
INCIDENT_NOTIFICATION_CHANNEL=<channel id>

# Workflow Configuration
SUMMARY_INTERVAL_MINUTES=30
MAX_THREAD_MESSAGES=50
AUTO_APPROVE=false
```

## Workflow Static Data Structure

The workflow uses n8n's workflow static data to maintain state:

```javascript
{
  "active_incidents": {
    "incident_id_1": {
      "id": "incident_id_1",
      "service": "api-gateway",
      "channel": "<id>",
      "thread_ts": "1699999999.123456",
      "started_at": "2025-11-10T10:30:00Z",
      "last_summary_at": "2025-11-10T11:00:00Z",
      "status": "active",
      "action_items": []
    }
  },
  "summaries_pending_approval": []
}
```

## Testing the Workflow

### Test Alert Webhook
```bash
curl -X POST https://your-n8n-instance.com/webhook/alerts \
  -H "Content-Type: application/json" \
  -d '{
    "alert_name": "Test Alert",
    "severity": "medium",
    "service": "test-service",
    "timestamp": "2025-11-10T10:30:00Z",
    "details": "This is a test alert",
    "slack_channel": "YOUR_TEST_CHANNEL_ID"
  }'
```

### Test Slack Mention
In your Slack workspace:
```
@SRE-Agent test incident response
```

## Monitoring and Debugging

### Enable Debug Mode
1. Open n8n workflow
2. Click on any node
3. Enable "Always Output Data"
4. Run workflow manually and inspect outputs

### View Execution Logs
1. Navigate to "Executions" in n8n
2. Click on any execution to see detailed logs
3. Check each node's input/output

### Common Issues

**Issue: Vexa AI not responding**
- Verify API key is correct
- Check API endpoint URL
- Ensure sufficient API credits

**Issue: Slack bot not posting**
- Verify bot token has correct scopes
- Ensure bot is added to target channels
- Check if channel IDs are correct

**Issue: Summaries not generating**
- Check schedule trigger is enabled
- Verify workflow static data contains active incidents
- Review Vexa AI request format

## Advanced Configuration

### Custom Prompt Templates
Store prompts in workflow static data for easy modification:

```javascript
$workflow.staticData.prompts = {
  analysis: "Your custom analysis prompt...",
  summary: "Your custom summary prompt...",
  action_items: "Your custom action items prompt..."
};
```

### Integration with Other Tools
Add HTTP Request nodes to integrate with:
- Jira (ticket creation)
- PagerDuty (incident updates)
- Datadog (metrics context)
- Custom internal APIs

## Production Checklist

- [ ] All credentials configured securely
- [ ] Environment variables set
- [ ] Webhook endpoints secured with authentication
- [ ] Error handling implemented
- [ ] Admin notification channel configured
- [ ] Rate limiting considered for Vexa AI calls
- [ ] Backup workflow exported
- [ ] Testing completed in staging environment
- [ ] Team trained on approval workflow
- [ ] Monitoring alerts configured

## Support

For issues with this workflow:
1. Check n8n execution logs
2. Verify all credentials are valid
3. Review Vexa AI API status
4. Consult the main README.md

For questions, open an issue on the GitHub repository.
