---
name: choreo-maestro
version: 1.0.0
description: Orchestrates multi-step automated workflows for OpenClaw game bot operations
author: OpenClaw Team
tags: [workflow, orchestration, coordination, process, automation]
type: orchestration
dependencies:
  - jq >= 1.6
  - curl >= 7.68.0
  - redis-cli (optional, for state persistence)
  - systemd (for service integration)
enabled: true
---

# Choreo Maestro Skill

## Purpose

Choreo Maestro orchestrates complex, multi-step automation sequences across OpenClaw's distributed bot ecosystem. It coordinates bot behaviors, manages state transitions, handles error recovery, and ensures atomic execution of dependent operations.

Real use cases:
- Coordinate 5-bot farming rotation with staggered start times and loot distribution
- Execute multi-stage boss fight sequences with role assignments (tank, healer, DPS)
- Manage server maintenance windows: graceful bot shutdown, config updates, restart orchestration
- Handle dynamic quest chains requiring bot-to-bot item handoffs at specific checkpoints
- Run seasonal event automation with timed triggers and conditional branching

## Scope

Commands available:
- `choreo:create-workflow` - Define new orchestrated sequence
- `choreo:register-bot` - Add bot to coordination pool
- `choreo:assign-role` - Map bots to workflow roles
- `choreo:start-sequence` - Execute orchestrated workflow
- `choreo:pause-sequence` - Gracefully halt all bots
- `choreo:resume-sequence` - Continue from paused state
- `choreo:abort-sequence` - Emergency stop with cleanup
- `choreo:check-status` - Monitor workflow state
- `choreo:validate-dependencies` - Pre-flight check
- `choreo:rollback` - Revert to previous stable state

## Work Process

1. **Workflow Definition**
   ```
   choreo:create-workflow --name "farm-rotation-5" \
     --steps 12 \
     --duration 45m \
     --concurrency 3 \
     --failure-policy "retry:3,backoff:exponential"
   ```

2. **Bot Registration**
   ```
   choreo:register-bot --id "bot-01" \
     --endpoint "ws://192.168.1.101:8080" \
     --capabilities "melee,ranged,tank" \
     --max-load 0.8
   ```

3. **Role Assignment**
   ```
   choreo:assign-role --workflow "farm-rotation-5" \
     --bot "bot-01" \
     --role "primary-dps" \
     --position "northeast-corner" \
     --timing-offset 0s
   ```

4. **Dependency Validation**
   ```
   choreo:validate-dependencies --workflow "farm-rotation-5" \
     --check network-latency \
     --check resource-availability \
     --check auth-tokens
   ```

5. **Sequencing Start**
   ```
   choreo:start-sequence --workflow "farm-rotation-5" \
     --start-time "now" \
     --notify-on-completion \
     --log-level detailed
   ```

6. **Runtime Monitoring**
   ```
   choreo:check-status --workflow "farm-rotation-5" \
     --format json \
     --watch
   ```

7. **Controlled Pause/Resume**
   ```
   choreo:pause-sequence --workflow "farm-rotation-5" \
     --grace-period 30s \
     --save-state
   
   choreo:resume-sequence --workflow "farm-rotation-5" \
     --from-checkpoint "last-saved"
   ```

## Golden Rules

1. **Always validate dependencies** before starting any sequence > 5 minutes duration
2. **Never assign overlapping positions** to bots without explicit collision avoidance rules
3. **Set max-load per bot** - never exceed 85% CPU or 90% memory threshold
4. **Implement checkpointing** every 10 minutes for long-running workflows
5. **Use exponential backoff** for retries (min: 2s, max: 30s, factor: 2)
6. **Maintain audit logs** - all state changes must be journaled to `/var/log/choreo/`
7. **Test rollback paths** monthly with staging environment
8. **Never hardcode credentials** - use vault integration: `choreo:get-secret <key>`
9. **Resource cleanup mandatory** - always include `--cleanup` flag or explicit cleanup step
10. **Graceful degradation** - workflows must continue with reduced capacity if bots fail

## Examples

### Example 1: Farming Rotation
```
Input:
choreo:create-workflow --name "gw2-farm-route" --steps 8 --duration 2h --concurrency 4
choreo:register-bot --id "anna" --endpoint "ws://10.0.0.1:8080" --capabilities "guardian"
choreo:assign-role --workflow "gw2-farm-route" --bot "anna" --role "tank" --path "/routes/farm01.txt"
choreo:start-sequence --workflow "gw2-farm-route" --start-time "2026-03-10T14:00:00Z"

Output:
✓ Workflow created (ID: wf_abc123)
✓ Bot registered (ID: bot_anna)
✓ Role assigned: anna → tank on route /routes/farm01.txt
✓ Dependencies validated (network OK, sufficient resources, tokens valid)
✓ Sequence scheduled for 2026-03-10T14:00:00Z
[14:00:00] ▶ Starting workflow gw2-farm-route
[14:00:02] ✓ Anna initialized, loading route file
[14:00:05] ✓ Route loaded (48 nodes, 12 farming spots)
[14:00:10] ▶ Bot "anna" executing step 1/8
```

### Example 2: Boss Fight Coordination
```
Input:
choreo:create-workflow --name "xd-boss-ragnos" --steps 20 --concurrency 5 --failure-policy "abort"
choreo:register-bot --id "tank01" --capabilities "tank,cc-immunity" --max-load 0.75
choreo:register-bot --id "heal01" --capabilities "heal,cleanse" --max-load 0.65
choreo:assign-role --workflow "xd-boss-ragnos" --bot "tank01" --role "main-tank" --phase 1,2,3
choreo:assign-role --workflow "xd-boss-ragnos" --bot "heal01" --role "main-heal" --phase 1,2,3
choreo:validate-dependencies --workflow "xd-boss-ragnos"
choreo:start-sequence --workflow "xd-boss-ragnos" --start-time "now+30s" --notify-discord "# raid-alerts"

Output:
✓ Workflow created (ID: wf_def456)
✓ Bots registered: tank01, heal01
✓ Role assignment complete (2 bots, 3 phases)
✓ Validation passed: all bots online, latency <50ms, healing capacity 12k/s
✓ Sequence starting in 30s
[14:00:30] ▶ Engaging boss: Ragnos the Fractured
[14:00:32] ✓ Tank01 positioning at (123,456)
[14:00:33] ✓ Heal01 monitoring tank
[14:01:15] ▶ Phase 2 transition
[14:01:17] ✓ Positioning adjusted, adds spawning
[14:02:45] ✓ Boss defeated, loot distribution triggered
```

### Example 3: Emergency Recovery
```
Input:
choreo:pause-sequence --workflow "gw2-farm-route" --grace-period 15s --save-state
choreo:check-status --workflow "gw2-farm-route" --format json

Output:
[14:30:00] ⏸ Pausing workflow gw2-farm-route (grace: 15s)
[14:30:01] ✓ Anna completing current node, saving position
[14:30:03] ⏸ State saved at node #7, progress 63%
[14:30:15] ✓ All bots paused, network connections closed

Status:
{
  "workflow": "gw2-farm-route",
  "state": "PAUSED",
  "checkpoint": {
    "node": 7,
    "position": [1240.5, 890.3, 12.1],
    "inventory": [...],
    "timestamp": "2026-03-10T14:30:03Z"
  },
  "bots": {
    "anna": "paused",
    "bot_02": "paused"
  }
}
```

## Dependencies & Requirements

System:
- Linux kernel 5.4+
- Python 3.9+ (for orchestration engine)
- Redis 6.2+ (optional state persistence)
- 100MB free disk space for logs
- Network: <100ms latency to all bot endpoints

Environment variables:
- `CHOREO_VAULT_ADDR` - HashiCorp Vault endpoint (required for secrets)
- `CHOREO_LOG_LEVEL` - debug/info/warn/error (default: info)
- `CHOREO_STATE_DIR` - checkpoint directory (default: /var/lib/choreo)
- `CHOREO_MAX_RETRIES` - global retry limit (default: 3)

File structure:
```
/var/lib/choreo/
├── workflows/          # Workflow definitions
├── checkpoints/        # Auto-saved states
├── bots/               # Bot registry
└── journal.log         # Audit trail
```

## Verification Steps

After any orchestration change:

1. Verify workflow syntax:
   ```
   choreo:validate-dependencies --workflow <name> --strict
   ```

2. Dry-run simulation (no bots affected):
   ```
   choreo:start-sequence --workflow <name> --dry-run --verbose
   ```

3. Check resource allocation:
   ```
   choreo:check-status --workflow <name> --format json | jq '.resource_usage'
   ```

4. Verify state persistence:
   ```
   ls -l /var/lib/choreo/checkpoints/<workflow-id>.json
   ```

5. Test rollback path:
   ```
   choreo:rollback --workflow <name> --to-checkpoint latest --dry-run
   ```

## Troubleshooting

**Issue**: "concurrency limit exceeded"
- **Fix**: Increase `--concurrency` or reduce bot assignments per workflow
- **Check**: `choreo:check-status --all | grep concurrency`

**Issue**: "bot endpoint unreachable"
- **Fix**: Verify bot WebSocket is running: `curl -I ws://<ip>:8080`
- **Check**: Network firewall, bot service status

**Issue**: "checkpoint save failed"
- **Fix**: Ensure `/var/lib/choreo/checkpoints/` is writable, disk not full
- **Check**: `df -h /var/lib/choreo`, `ls -ld /var/lib/choreo/checkpoints`

**Issue**: "state deserialization error on resume"
- **Fix**: Use `choreo:rollback --to-checkpoint <timestamp>` to revert to known-good state
- **Check**: Validate checkpoint JSON: `jq . /var/lib/choreo/checkpoints/<file>`

**Issue**: "dependency validation timeout"
- **Fix**: Increase timeout: `export CHOREO_VALIDATE_TIMEOUT=30s`
- **Check**: Bot response latency: `choreo:ping-bot --id <bot-id>`

**Issue**: "role assignment conflict"
- **Fix**: Ensure unique `(position + phase)` pairs per workflow
- **Check**: `choreo:list-assignments --workflow <name> --format table`

## Rollback Commands

Immediate rollback to previous checkpoint:
```
choreo:rollback --workflow "gw2-farm-route" --to-checkpoint latest
```

Rollback to specific timestamp:
```
choreo:rollback --workflow "gw2-farm-route" --to-checkpoint "2026-03-10T14:30:03Z"
```

Full workflow revert with cleanup:
```
choreo:rollback --workflow "gw2-farm-route" \
  --to-checkpoint initial \
  --cleanup-paused-bots \
  --notify-ops
```

Emergency abort + rollback (use cautiously):
```
choreo:abort-sequence --workflow "gw2-farm-route" \
  --force \
  --rollback-to "stable-state" \
  --disconnect-bots
```

Partial rollback (only failed bots):
```
choreo:rollback --workflow "gw2-farm-route" \
  --bots "bot_03,bot_07" \
  --to-checkpoint "last-known-good"
```

Verify rollback success:
```
choreo:check-status --workflow "gw2-farm-route" | grep "state"
# Should show: ROLLED_BACK or STABLE
```

Rollback audit trail:
```
grep "ROLLBACK" /var/log/choreo/journal.log | tail -10
```

Note: All rollbacks preserve audit entries and cannot be undone without manual intervention. Always validate bot health after rollback: `choreo:ping-all --workflow <name>`.
```