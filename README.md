# ZeroTrusted.ai Guardrails Copilot — Claude Code Skill

A [Claude Code](https://claude.ai/claude-code) skill for adding [ZeroTrusted.ai](https://zerotrusted.ai) guardrails to any AI copilot or chat UI. Includes complete React components, Next.js API routes, and the full privacy-safe conversation flow with PII detection, anonymization, auto-deanonymization, and hallucination checking.

## What is a Claude Code Skill?

Claude Code skills are markdown files that give Claude deep knowledge of specific APIs, patterns, and workflows. When loaded into a Claude Code session, this skill enables Claude to add production-ready guardrails to your copilot — with working UI components, API routes, and state management.

## Features Covered

- **PII Detection Modal** — Warns users when sensitive data is detected, with entity-by-entity breakdown
- **One-Click Anonymization** — Replace PII with realistic fake values before sending to LLM
- **Auto-Deanonymization** — Automatically restore original values in AI responses (default ON, toggleable)
- **History Scrubbing** — Ensures original PII never leaks to the LLM through conversation history
- **Hallucination Check** — Reliability scoring with auto/manual modes and visual badges
- **Validation Badges** — Per-message indicators showing PII scan status
- **Configurable Safety Modes** — Block, Warn, Allow, and Manual modes for both PII and hallucination checks
- **Settings UI** — User-facing controls for safety preferences and API key overrides

## Quick Start

### 1. Get your API key

Sign up at [zerotrusted.ai](https://zerotrusted.ai) and get your API key.

### 2. Add to your environment

```bash
# .env.local
ZT_GUARDRAILS_API_KEY=zt-your-api-key-here
ZT_ENCRYPTED_PROVIDER_KEY=your-encrypted-key  # For hallucination checks
```

### 3. Load the skill in Claude Code

Copy `zt-guardrails-copilot.md` into your project's `.claude/skills/` directory:

```bash
mkdir -p .claude/skills
cp zt-guardrails-copilot.md .claude/skills/
```

Then ask Claude Code: *"Add ZeroTrusted.ai guardrails to my copilot"*

## Architecture

```
User types message
  -> PII Pre-Check (detect-sensitive-keywords)
     -> PII found -> Show modal -> Anonymize -> Send safe text to LLM
        -> LLM responds -> Auto-deanonymize -> Show original values
     -> No PII -> Send directly
  -> Post-response: Hallucination check (auto or manual)
```

## What Gets Created

| File | Purpose |
|------|---------|
| `api/copilot/pii-check/route.ts` | Detect PII in user input |
| `api/copilot/anonymize/route.ts` | Anonymize detected PII |
| `api/copilot/deanonymize/route.ts` | Restore original values |
| `api/copilot/hallucination-check/route.ts` | Evaluate response reliability |
| `CopilotPanel.tsx` | Main UI with all guardrails integrated |
| `guardrails.ts` | Shared config (reads from env) |

## UI Components Included

- Animated PII scanning steps indicator
- PII warning modal with detection list and anonymization mappings
- Auto-deanonymize toggle (default ON)
- Reliability/hallucination badge (green/red with score)
- Manual "Check Reliability" button
- Deanonymizing progress indicator
- Safety settings panel

## Related

- [zt-guardrails-api-skill](https://github.com/www-zerotrusted-ai/zt-guardrails-api-skill) — Claude Code skill for the base guardrails API (no UI)

## License

MIT
