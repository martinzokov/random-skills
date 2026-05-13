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

## Install

```bash
npx skills add martinzokov/random-skills
```
