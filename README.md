# random-skills

Claude Code skills collection.

## Skills

### `/token-saver`

Before you build — check if it already exists.

Given an app or business idea, does a deep-dive to find products that already cover 80-90% of the concept. Surfaces exact competitors, overlapping features, platform integrations, no-code workarounds, open-source alternatives, and delivers a hard verdict.

**Verdicts:**
- `DON'T BUILD` — funded competition already covers this, here's the best one
- `PROCEED WITH CAUTION` — partial overlap, real gap exists but risky
- `CLEAR LANE` — nothing covers this adequately, here's how to validate

**Usage:**
```
/token-saver I want to build an AI that writes Jira tickets from Slack messages
/token-saver <idea> --angle <specific differentiator>
/token-saver <idea> --market <target segment>
```

### `/hex-infra`

Use this when implementing a feature or behavior change with hexagonal architecture and DDD-informed boundaries.

It enforces a clear build flow:
- classify work as **Command** (writes) or **Query** (reads)
- write application handlers without HTTP/DB/framework imports
- define dependency ports as interfaces
- implement transport/storage/external adapters behind those ports

This keeps domain and application logic isolated from infrastructure and makes behavior easier to test, evolve, and reason about.

**Usage:**
```
/hex-infra Implement order cancellation with refund handling
/hex-infra Add read-only endpoint for order summary by customer id
```

## Install

```bash
npx skills add martinzokov/random-skills
```
