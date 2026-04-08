# clarify

Adaptive interview plugin for Claude Code. Asks binary questions before coding begins, outputs a concrete `PLAN.md`.

## Install

```bash
# From local path
claude plugins install /path/to/clarify

# Or symlink into plugins cache
ln -s /path/to/clarify ~/.claude/plugins/cache/clarify-local/clarify/0.5.0
```

## Usage

```
/clarify I need a REST API for user management
```

Or with no arguments (will prompt for description):

```
/clarify
```

## How It Works

1. **SCAN** — reads project context (stack, patterns, existing plans)
2. **PREFILL** — confirms derived facts (e.g., "Stack: Next.js, TypeScript — correct?")
3. **INTERVIEW** — adaptive binary questions in batches of 5
4. **SYNTHESIZE** — generates `PLAN.md` with files, changes, implementation order
5. **PERSIST** — saves session to `.clarify/sessions/` for resumption
6. **OFFER** — asks whether to start execution

## Depth Modes

| Mode | Questions | When |
|------|-----------|------|
| compact | ~5 | Small, well-defined changes |
| standard | ~15 | New feature or module |
| thorough | 30+ | New project or architecture decision |

Auto-detected from request complexity. Override at the prompt.

## Session Data

Sessions saved to `.clarify/sessions/<timestamp>.json`. Contains all questions, answers, and contradictions detected. Enables resuming interrupted interviews.

## Files Generated

- `PLAN.md` — execution plan in project root
- `.clarify/sessions/*.json` — interview session data
- `CLAUDE.local.md` — key decisions appended (with permission)
