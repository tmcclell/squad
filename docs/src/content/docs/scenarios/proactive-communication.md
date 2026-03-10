# Proactive Communication Patterns

> ⚠️ **Experimental** — Squad is alpha software. APIs, commands, and behavior may change between releases.


**Try this to set up outbound notifications:**
```bash
# Store your Teams webhook URL
echo "https://outlook.office.com/webhook/..." > ~/.squad/teams-webhook.url
```

**Try this to scan for external work:**
```
Ralph, check Teams and email for new work
```

Squad can both push notifications to you AND pull information from your environment. These patterns work independently but are most powerful together, creating a two-way communication layer between Squad and your work environment.

---

## Outbound: Teams Webhook Notifications

Any agent can send notifications to you via a Teams webhook by reading a stored URL and POSTing an Adaptive Card. This keeps you informed on mobile without being at the terminal.

### Setup

Store your webhook URL at a known path:

```bash
# Create webhook URL file
mkdir -p ~/.squad
echo "https://outlook.office.com/webhook/..." > ~/.squad/teams-webhook.url
chmod 600 ~/.squad/teams-webhook.url  # Protect webhook URL
```

### Notification Tiers

Different urgency levels for different situations:

| Tier | When | Examples |
|------|------|----------|
| ⚡ **Breaking** | Critical failures | CI broken, merge conflicts, blocked PRs |
| 📰 **Briefings** | Daily summaries | Progress reports, work completed |
| 📊 **Recaps** | Weekly highlights | Stats, trends, milestones |
| 🎯 **Flashes** | Quick snapshots | Board status, queue depth |

**Key rule:** Only send when there's genuinely newsworthy activity. Don't spam yourself.

### Sending a Notification

```powershell
# Read webhook URL
$webhookUrl = Get-Content ~/.squad/teams-webhook.url -Raw

# Build Adaptive Card
$card = @{
    type = "message"
    attachments = @(
        @{
            contentType = "application/vnd.microsoft.card.adaptive"
            content = @{
                type = "AdaptiveCard"
                version = "1.4"
                body = @(
                    @{
                        type = "TextBlock"
                        text = "⚡ CI Failure"
                        size = "Large"
                        weight = "Bolder"
                    }
                    @{
                        type = "TextBlock"
                        text = "Build failed on main branch"
                        wrap = $true
                    }
                )
                actions = @(
                    @{
                        type = "Action.OpenUrl"
                        title = "View Logs"
                        url = "https://github.com/user/repo/actions/runs/123"
                    }
                )
            }
        }
    )
} | ConvertTo-Json -Depth 10

# Send notification
Invoke-RestMethod -Uri $webhookUrl -Method Post -Body $card -ContentType "application/json"
```

```bash
# Bash equivalent
WEBHOOK_URL=$(cat ~/.squad/teams-webhook.url)

curl -X POST "$WEBHOOK_URL" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "message",
    "attachments": [{
      "contentType": "application/vnd.microsoft.card.adaptive",
      "content": {
        "type": "AdaptiveCard",
        "version": "1.4",
        "body": [{
          "type": "TextBlock",
          "text": "⚡ CI Failure",
          "size": "Large",
          "weight": "Bolder"
        }]
      }
    }]
  }'
```

**Note:** This pattern works with any webhook-capable platform (Teams, Slack, Discord) — Teams is just the most common example.

---

## Inbound: Scanning Emails and Teams for Work

Use WorkIQ MCP server or Playwright to periodically read Teams channels and email, evaluating messages for actionability. When items need attention, create GitHub issues automatically.

### Pattern Overview

1. **Agent reads external sources** — Teams channels, email threads, shared documents
2. **Evaluates actionability** — Is this a bug report? Feature request? Question needing documentation?
3. **Creates GitHub issues** — Auto-create issues with `teams-bridge` label for items needing attention
4. **Anti-duplicate logic** — Check existing issues before creating new ones to avoid spam

### Example Implementation

```typescript
// Using WorkIQ MCP to scan Teams
const recentMessages = await workiq.getTeamsMessages({
  channel: "engineering",
  since: "24h"
});

for (const message of recentMessages) {
  // Evaluate: Does this need an issue?
  const needsIssue = await evaluateActionability(message);
  
  if (!needsIssue) continue;
  
  // Check for duplicates
  const existing = await github.searchIssues({
    q: `repo:owner/repo is:issue "${message.subject}"`,
    label: "teams-bridge"
  });
  
  if (existing.length > 0) continue;
  
  // Create issue
  await github.createIssue({
    title: message.subject,
    body: `From Teams channel #${message.channel}:\n\n${message.content}`,
    labels: ["teams-bridge"]
  });
}
```

**Key rule:** Do NOT spam. Only surface items that genuinely need attention. Use filters, keyword matching, and sentiment analysis to avoid creating issues for every casual message.

### Label Convention

Mark auto-created issues with `teams-bridge` (or similar) so the team knows the source. This also enables filtered views and routing rules.

---

## Connecting the Loop

These patterns work together to create a complete feedback cycle:

1. **Inbound scanning creates issues** — External work enters GitHub
2. **Ralph picks them up** — Work monitor detects new issues, dispatches agents
3. **Agents do the work** — Code changes, PRs opened, reviews completed
4. **Outbound notifications report results** — You get notified on mobile

The human stays informed via mobile (GitHub app + Teams notifications) without being at the terminal. You can review PRs from your phone, approve merges from anywhere, and jump in only when needed.

### Example Flow

```
1. Customer posts bug report in Teams → WorkIQ creates issue #123
2. Ralph triages issue #123 → Assigns to Backend agent
3. Backend creates PR #45 → Requests Lead review
4. Lead approves PR #45 → Merge successful
5. Ralph sends Teams notification → "🎯 Issue #123 closed via PR #45"
6. You see notification on phone → No action needed, just awareness
```

---

## Prerequisites

These patterns enhance Squad but are not requirements. Squad works fine without them.

### For Outbound Notifications

- Teams webhook URL (or Slack/Discord equivalent)
- Stored at a known path (e.g., `~/.squad/teams-webhook.url`)
- Agents configured to send notifications for specific events

### For Inbound Scanning

- WorkIQ MCP server installed and authenticated
- Playwright or other automation tool for non-WorkIQ sources
- GitHub token with `repo` scope for creating issues
- Anti-spam filters and duplicate detection logic

---

## See Also

- [Ralph — Work Monitor](../features/ralph.md) — Ralph's work processing and monitoring
- [GitHub Issues Mode](../features/github-issues.md) — Issue-driven workflow basics
- [Ralph Operational Deployment Patterns](./ralph-operations.md) — 24/7 Ralph deployment
