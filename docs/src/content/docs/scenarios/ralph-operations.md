# Ralph Operational Deployment Patterns

> ⚠️ **Experimental** — Squad is alpha software. APIs, commands, and behavior may change between releases.


**Try this to understand Ralph's built-in behavior first:**
```
Ralph, show me what's on the board
```

**Then explore this guide for 24/7 deployment patterns.**

Ralph's built-in work monitoring runs during active Copilot sessions. When you need **persistent, unattended monitoring** — Ralph running 24/7 across server restarts, code updates, and agent configuration changes — you need an outer loop. This guide covers operational patterns for deploying Ralph in production environments.

See [Ralph — Work Monitor](../features/ralph.md) for Ralph's in-session behavior, watch mode, and cloud heartbeat. This guide focuses on the **deployment layer** that wraps Squad.

---

## The Outer Loop Pattern

Ralph's in-session loop processes work until the board is clear, then idles. For continuous operation, wrap Squad in a persistent outer loop:

```powershell
# Simplified pattern - see reference implementation for production version
while ($true) {
    git pull origin main  # Fresh code before each round
    
    # Spawn fresh Copilot process (picks up updated agent definitions)
    gh copilot run "Ralph, process the work queue"
    
    # Log results, check exit code, update metrics
    Write-Log "Round completed at $(Get-Date)"
    
    Start-Sleep -Seconds 600  # 10-minute pause between rounds
}
```

### Why the Outer Loop Matters

- **Fresh context every round** — Agents may update charters, skills, or routing rules between sessions
- **Clean process state** — New Copilot process picks up MCP server changes, tool updates, configuration edits
- **Resilience** — Failed rounds don't crash the watchdog; the outer loop continues
- **Observability** — Each round produces discrete log entries with metrics

**Reference implementation:** [ralph-watch.ps1](https://github.com/tamirdresher/squad-personal-demo/blob/main/ralph-watch.ps1) — A production-ready PowerShell outer loop with all patterns below integrated.

---

## Single-Instance Guard

Prevent multiple Ralph instances from running simultaneously using a system-wide named mutex or lockfile:

```powershell
# Named mutex approach (Windows)
$mutex = [System.Threading.Mutex]::new($false, "Global\SquadRalphWatch")
if (-not $mutex.WaitOne(0)) {
    Write-Error "Ralph is already running (PID found in heartbeat file)"
    exit 1
}

try {
    # Main watchdog loop runs here
} finally {
    $mutex.ReleaseMutex()
    $mutex.Dispose()
}
```

```bash
# Lockfile approach (Linux/macOS)
LOCKFILE="/var/run/squad-ralph.lock"

if [ -f "$LOCKFILE" ]; then
    PID=$(cat "$LOCKFILE")
    if ps -p "$PID" > /dev/null 2>&1; then
        echo "Ralph is already running (PID $PID)"
        exit 1
    else
        echo "Stale lockfile found, cleaning up"
        rm -f "$LOCKFILE"
    fi
fi

echo $$ > "$LOCKFILE"
trap "rm -f $LOCKFILE" EXIT

# Main watchdog loop runs here
```

### Stale Instance Detection

If the watchdog crashes, the mutex releases automatically (Windows) or the lockfile remains but the process is gone (Unix). Before exiting with "already running", check if the PID in the lockfile/heartbeat is still alive. If not, clean up and proceed.

---

## Structured Logging

Each round produces a structured log entry with timestamp, round number, exit code, duration, and parsed metrics:

```json
{
  "timestamp": "2025-03-15T14:32:00Z",
  "round": 47,
  "duration_seconds": 132,
  "exit_code": 0,
  "status": "success",
  "metrics": {
    "issues_closed": 2,
    "prs_merged": 1,
    "agent_actions": 5
  },
  "errors": []
}
```

### Log Rotation

Rotate logs when they reach 500 entries or 1 MB, whichever comes first. Keep the last 5 rotated files for historical analysis:

- `ralph-watch.log` — current log
- `ralph-watch.log.1` — previous rotation
- `ralph-watch.log.2` through `ralph-watch.log.5` — older rotations

---

## Failure Alerting

Transient failures are normal in distributed systems. Alert only after **N consecutive failures** to avoid notification fatigue:

```powershell
$consecutiveFailures = 0
$FAILURE_THRESHOLD = 3

while ($true) {
    $exitCode = Run-RalphRound
    
    if ($exitCode -ne 0) {
        $consecutiveFailures++
        
        if ($consecutiveFailures -ge $FAILURE_THRESHOLD) {
            Send-Alert @{
                Title = "Ralph Watchdog: $consecutiveFailures consecutive failures"
                ExitCode = $exitCode
                Timestamp = Get-Date
                LogTail = Get-Content ralph-watch.log -Tail 50
            }
        }
    } else {
        $consecutiveFailures = 0  # Reset on success
    }
    
    Start-Sleep -Seconds 600
}
```

### Alert Channels

- **Teams webhook** — Send structured cards with failure count, exit code, log excerpts
- **Email** — SMTP delivery for critical alerts
- **PagerDuty/Opsgenie** — Escalate to on-call if threshold exceeds N hours

---

## Heartbeat File

Write a JSON heartbeat file before and after every round with status, metrics, and process ID:

```json
{
  "status": "running",
  "pid": 12345,
  "current_round": 47,
  "last_success": "2025-03-15T14:32:00Z",
  "metrics": {
    "total_issues_closed": 142,
    "total_prs_merged": 89,
    "uptime_hours": 336
  },
  "health": "ok"
}
```

External monitoring tools (Prometheus, Datadog, Azure Monitor) can:

- Poll the heartbeat file every minute
- Alert if `last_success` timestamp is older than expected round interval × 2
- Track cumulative metrics over time
- Verify the PID is still running

### Heartbeat Update Pattern

```powershell
# Before round starts
$heartbeat = @{
    status = "running"
    pid = $PID
    current_round = $roundNumber
    last_success = $lastSuccess
}
$heartbeat | ConvertTo-Json | Set-Content heartbeat.json

# After round completes
$heartbeat.status = "idle"
$heartbeat.last_success = Get-Date -Format "o"
$heartbeat.metrics.total_issues_closed += $issuesClosed
$heartbeat | ConvertTo-Json | Set-Content heartbeat.json
```

---

## Background Activity Monitor

While the Copilot session runs, tail logs every 30 seconds to show what's happening:

```powershell
# Start Ralph round in background
$job = Start-Job -ScriptBlock {
    gh copilot run "Ralph, process the work queue"
}

# Monitor activity while it runs
while ($job.State -eq 'Running') {
    # Show last 5 lines of Squad log
    Get-Content .squad/log/session-*.log -Tail 5 | Write-Host -ForegroundColor Cyan
    
    Start-Sleep -Seconds 30
}

# Collect final results
$result = Receive-Job -Job $job -Wait
```

This prevents the terminal from sitting silently for 10+ minutes while Ralph works through a large backlog. The operator sees progress in real-time.

---

## Deployment Checklist

When deploying Ralph in production:

- [ ] Outer loop pulls latest code before each round
- [ ] Single-instance guard prevents duplicate watchdogs
- [ ] Structured logging with rotation (500 entries or 1 MB)
- [ ] Failure alerting after N=3 consecutive failures
- [ ] Heartbeat file updated before and after rounds
- [ ] Background activity monitor shows progress
- [ ] Monitoring system polls heartbeat file
- [ ] Alert channels tested (Teams, email, PagerDuty)
- [ ] Log rotation tested (verify old logs are kept)
- [ ] Stale instance cleanup tested (kill PID, verify recovery)

---

## Architecture Summary

Ralph operates at three layers (from [Ralph — Work Monitor](../features/ralph.md)):

| Layer | When | How |
|-------|------|-----|
| **In-session** | You're at the keyboard | "Ralph, go" — active loop while work exists |
| **Local watchdog** | You're away but machine is on | `squad watch --interval 10` |
| **Cloud heartbeat** | Fully unattended | `squad-heartbeat.yml` GitHub Actions cron |

This guide adds a **fourth layer** — the operational wrapper that makes Ralph production-ready:

| Layer | Purpose |
|-------|---------|
| **Outer loop** | Persistent deployment — fresh code, fresh process, structured logging, alerting |

---

## Sample Prompts

```
Ralph, process the work queue
```

The outer loop invokes this command each round. Ralph triages issues, dispatches agents, monitors PRs, and reports results.

---

## Notes

- The outer loop is **not part of Squad** — it's infrastructure you build around Squad
- Ralph's built-in watch mode (`squad watch`) is suitable for local development; the outer loop is for production
- Test your outer loop with a small backlog first (1-2 issues) before deploying to a live board
- Monitor heartbeat file staleness — if no update in 2× the expected round interval, investigate

---

## See Also

- [Ralph — Work Monitor](../features/ralph.md) — Ralph's built-in behavior
- [GitHub Issues Mode](../features/github-issues.md) — Issue-driven workflow
- [CI/CD Integration](./ci-cd-integration.md) — Automated Squad workflows
