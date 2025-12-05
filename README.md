# VelAI - SRE On-Call AI Agent
An intelligent AI-powered agent that automates incident management workflows, generates summaries, and facilitates collaboration during on-call incidents.

## 🚀 Features

- **Alert Webhook Integration**: Automatically receives and processes alert webhooks from monitoring systems
- **Slack Thread Participation**: Joins incident-specific Slack threads when mentioned
- **Real-time Incident Tracking**: Monitors ongoing incident conversations
- **Periodic Summaries**: Generates automated summaries of incident progress at configurable intervals
- **Action Item Extraction**: Identifies and tracks action items from incident discussions
- **Review Workflow**: Submits summaries to SRE/admin for approval before publishing
- **Stakeholder Communication**: Publishes approved summaries to broader audience channels

## 🏗️ Architecture

```
┌─────────────────┐         ┌──────────────────┐
│  Monitoring     │         │   Slack Thread   │
│  Systems        │         │   Mention        │
└────────┬────────┘         └────────┬─────────┘
         │                           │
         │ Webhook                   │ @bot
         │                           │
         └───────────┬───────────────┘
                     ▼
         ┌───────────────────────┐
         │        VelAI          │
         │    (Orchestration)    │
         └───────────┬───────────┘
                     │
                     ▼
         ┌───────────────────────┐
         │         Vexa          │
         │   (LLM Processing)    │
         └───────────┬───────────┘
                     │
         ├───────────┼───────────┤
         ▼           ▼           ▼
    ┌────────┐  ┌────────┐  ┌──────────┐
    │Summary │  │Action  │  │Review    │
    │Generate│  │Items   │  │Queue     │
    └────────┘  └────────┘  └──────────┘
         │           │           │
         └───────────┴───────────┘
                     ▼
         ┌───────────────────────┐
         │   Admin Approval      │
         └───────────┬───────────┘
                     ▼
         ┌───────────────────────┐
         │  Publish to Channel   │
         └───────────────────────┘
```

## 📋 Prerequisites

- **n8n**: Version 1.0.0 or higher
  - Self-hosted or cloud instance
  - Access to create and manage workflows
- **Vexa**: Local docker setup with API access
  - API key with appropriate permissions
- **Slack**: 
  - Workspace admin access
  - Ability to create Slack apps and bots
  - OAuth tokens with required scopes
- **Monitoring System**: Any system capable of sending webhooks (PagerDuty, Grafana, Prometheus, etc.)

## 🔧 Installation

### 1. Set Up n8n

#### Self-Hosted
```bash
# Using Docker
docker run -it --rm \
  --name n8n \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

# Using npm
npm install n8n -g
n8n start
```

### 2. Configure Vexa

1. Setup vexa AI code in Docker container
2. Navigate to API settings
3. Generate an API key
4. Note your endpoint URL

### 3. Set Up Slack App

1. Go to [Slack API Apps](https://api.slack.com/apps)
2. Click "Create New App" → "From scratch"
3. Name your app (e.g., "SRE On-Call Agent")
4. Select your workspace

#### Configure Bot Scopes
Add the following OAuth scopes under "OAuth & Permissions":
- `app_mentions:read` - Read mentions
- `channels:history` - Read channel messages
- `channels:read` - View basic channel info
- `chat:write` - Send messages
- `chat:write.public` - Send messages to channels the app isn't in
- `im:history` - View DM messages
- `reactions:write` - Add reactions
- `users:read` - View users

#### Enable Event Subscriptions
Under "Event Subscriptions":
- Enable Events: ON
- Request URL: `https://your-n8n-instance.com/webhook/slack-events`
- Subscribe to bot events:
  - `app_mention`
  - `message.channels`

#### Install App to Workspace
1. Navigate to "Install App"
2. Click "Install to Workspace"
3. Authorize the app
4. Copy the "Bot User OAuth Token"

### 4. Import n8n Workflow

1. Download the workflow file: `agent.json` from on-call agent source
2. In n8n, click "Import from File"
3. Select the downloaded workflow file
4. Configure credentials for:
   - Slack (Bot OAuth Token)
   - Vexa AI (API Key)
   - Your monitoring system webhooks

### 5. Configure Environment Variables

Set the following credentials in n8n:

```
VEXA_AI_API_KEY=your_vexa_api_key
VEXA_AI_ENDPOINT=https://your-vexa-instance.com/vexa
SLACK_BOT_TOKEN=xoxb-your-bot-token
SLACK_SIGNING_SECRET=your_signing_secret
ADMIN_CHANNEL_ID=C0123456789
INCIDENT_NOTIFICATION_CHANNEL=<channel-id>
SUMMARY_INTERVAL_MINUTES=30
```

## 🎯 Usage

### Starting the Agent

1. Activate the n8n workflow
2. Ensure all credentials are configured
3. The agent will now listen for:
   - Alert webhooks at: `https://your-n8n-instance.com/webhook/alerts`
   - Slack mentions in incident threads

### Triggering via Webhook

```bash
curl -X POST https://your-n8n-instance.com/webhook/alerts \
  -H "Content-Type: application/json" \
  -d '{
    "alert_name": "High CPU Usage",
    "severity": "critical",
    "service": "api-gateway",
    "timestamp": "2025-11-10T10:30:00Z",
    "details": "CPU usage exceeded 90% threshold"
  }'
```

### Triggering via Slack

In any Slack channel or thread:
```
@SRE-Agent help with this incident
```

The agent will:
1. Join the conversation
2. Analyze the incident context
3. Begin monitoring the thread
4. Generate periodic summaries
5. Extract action items
6. Send summaries to admin for review

### Workflow Behavior

1. **Initial Response**: Agent acknowledges the incident and confirms monitoring
2. **Active Monitoring**: Tracks all messages in the incident thread
3. **Periodic Summaries** (every 30 minutes by default):
   - Summary of incident progress
   - Key decisions made
   - Current status
   - Next steps
4. **Action Items**: Continuously extracts and tracks action items
5. **Admin Review**: Sends draft summaries to admin channel for approval
6. **Publication**: After approval, publishes to stakeholder channel

## ⚙️ Configuration

### Customize Summary Intervals

Edit the n8n workflow and modify the "Schedule Trigger" node:
```json
{
  "interval": 30,
  "unit": "minutes"
}
```

### Customize AI Prompts

Modify the Vexa AI node prompts in the workflow:
- Summary generation prompt
- Action item extraction prompt
- Incident classification prompt

### Add Custom Integrations

The workflow can be extended to integrate with:
- PagerDuty
- Jira (for ticket creation)
- StatusPage (for status updates)
- Custom databases
- Other communication platforms

## 📊 Monitoring and Logs

View agent activity:
1. In n8n: Navigate to "Executions" to see workflow runs
2. Check Slack audit logs for bot activity
3. Monitor Vexa AI usage in your dashboard

## 🔒 Security Best Practices

1. **API Keys**: Store all API keys in n8n's credential manager, never in workflow directly
2. **Webhook Security**: Use signing secrets to verify webhook authenticity
3. **Access Control**: Limit admin review channel to authorized personnel only
4. **Network Security**: Use HTTPS for all webhook endpoints
5. **Audit Logging**: Enable logging for all agent actions

## 🤝 Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/new-feature`
3. Commit your changes: `git commit -am 'Add new feature'`
4. Push to the branch: `git push origin feature/new-feature`
5. Submit a pull request

## 📝 License

This project is licensed under the Apache License 2.0 - see the [LICENSE](LICENSE) file for details.

## 🐛 Troubleshooting

### Agent Not Responding to Mentions
- Verify Slack bot token is valid
- Check that bot is added to the channel
- Ensure Event Subscriptions are properly configured

### Summaries Not Generating
- Check Vexa AI API key validity
- Verify n8n workflow is activated
- Review execution logs for errors

### Webhook Failures
- Confirm webhook URL is publicly accessible
- Check firewall rules
- Verify webhook payload format matches expected schema

## 📚 Additional Resources

- [n8n Documentation](https://docs.n8n.io)
- [Vexa AI Documentation](https://docs.vexa.ai)
- [Slack API Documentation](https://api.slack.com/docs)

## 💬 Support

For issues and questions:
- Open an issue on GitHub
- Join our community Slack channel
- Check the documentation wiki

## 🙏 Acknowledgments

- Built with [n8n](https://n8n.io) - Workflow automation platform
- Powered by [Vexa AI](https://vexa.ai) - AI processing engine
- Integrates with [Slack](https://slack.com) - Team communication

---

**Note**: This is a community-maintained project. Please ensure you comply with your organization's security policies before deploying in production environments.
