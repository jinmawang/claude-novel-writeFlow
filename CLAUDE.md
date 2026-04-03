# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Claude Code slash-command plugin for AI-assisted novel writing. It installs into any novel project directory via `.claude/commands/`. There are no build, lint, or test commands — all "code" is Markdown prompt files that Claude Code executes as slash commands.

## Slash Commands

All commands live in `.claude/commands/` and are invoked with `/` in Claude Code:

| Command | File | Purpose |
|---------|------|---------|
| `/outline` | `outline.md` | Create/edit novel outlines (interactive or `--auto` multi-agent brainstorm) |
| `/style` | `style.md` | Define writing style rules → saves `style-rules.md` |
| `/write` | `write.md` | Write chapters using 3-agent architecture (Writer + Style Reviewer + Continuity Reviewer) |
| `/context` | `context.md` | Initialize or rebuild `context/` tracking files from existing chapters |
| `/status` | `status.md` | Show writing progress report |
| `/export` | `export.md` | Merge all chapters into a single manuscript file |

## Expected Project File Structure

When used in a novel project, these directories/files are created:

```
outline/
  overview.md           # Full novel outline (characters, 3-act structure, chapter table)
  chapter-01.md         # Per-chapter outlines (zero-padded, 3 digits for ch≥100)
style-rules.md          # Writing style rules (10 sections + AI detection checklist)
chapters/
  chapter-01.md         # Written chapters
context/
  characters.md         # Confirmed character details from prose
  world.md              # Confirmed world-building details
  continuity.md         # Key events log
  timeline.md           # In-story timeline
```

## Architecture: Multi-Agent Workflows

### `/write` — Three-agent review loop (max 3 rounds)
1. **Writer Agent**: Produces draft using overview + chapter outline ± 2 adjacent chapter outlines + last 2 chapters' prose + all context files + style rules
2. **Style Reviewer Agent** (parallel): Checks 8 AI-mechanical-feel patterns + style rule compliance
3. **Continuity Reviewer Agent** (parallel): Checks outline adherence + chapter transitions + context file consistency
4. **Context Extractor Agent**: After save, extracts new facts from the chapter into `context/` files (deduplicates, appends changes)

### `/outline --auto` — Four parallel brainstorm agents
Market Research → Character Arc → Narrative Structure → Theme/Motif → synthesized into 5 complete outline proposals

## Key Conventions

- **Chapter file naming**: Zero-padded three digits for all chapters (`chapter-001.md`, `chapter-012.md`, `chapter-100.md`)
- **Context files**: Only record facts explicitly written in prose — not outline plans. Changes to existing facts are appended in-place (e.g., `- broken arm \`ch45\` → healed \`ch52\``)
- **`overview.md` truncation rule**: When >3000 chars, only chapter table rows N±5 are passed to agents; full character and structure sections always included
- **`style-rules.md` truncation rule**: When >4000 chars, Section 9 (examples) is skipped when passing to agents; Sections 10–11 (AI checklist, forbidden list) always included
- **Prose context truncation**: ch(N-1) last 800 chars, ch(N-2) last 300 chars passed to Writer Agent
