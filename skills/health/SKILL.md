---
name: health
description: Audit Claude Code configuration health across all layers (CLAUDE.md, rules, skills, hooks, MCP). Run periodically or when collaboration feels off.
---

# Claude Code Configuration Health Audit

Systematically audit the current project's Claude Code setup using the six-layer framework:
`CLAUDE.md → rules → skills → hooks → subagents → verifiers`

The goal is not just to find rule violations, but to diagnose which layer is misaligned and why — **calibrated to the project's actual complexity**.

## Step 0: Assess project tier

```bash
# Count source files, languages, contributors
find . -type f \( -name "*.rs" -o -name "*.ts" -o -name "*.py" -o -name "*.go" -o -name "*.lua" \) \
  ! -path "*/node_modules/*" ! -path "*/.git/*" | wc -l

git log --format="%ae" 2>/dev/null | sort -u | wc -l   # unique contributors
ls .github/workflows/*.yml 2>/dev/null | wc -l          # CI workflows
ls .claude/skills/ 2>/dev/null | wc -l                  # existing skills
wc -l CLAUDE.md 2>/dev/null                             # current CLAUDE.md size
```

Use this rubric to pick the audit tier before proceeding:

| Tier | Signal | What's expected |
|------|--------|-----------------|
| **Simple** | <500 source files, 1 contributor, no CI | CLAUDE.md only; 0–1 skills; no rules/; hooks optional |
| **Standard** | 500–5K files, small team or CI present | CLAUDE.md + 1–2 rules files; 2–4 skills; basic hooks |
| **Complex** | >5K files, multi-contributor, multi-language, active CI | Full six-layer setup required |

**Apply the tier's standard throughout the audit. Do not flag missing layers that aren't required for the detected tier.**

## Step 1: Collect configuration snapshot

```bash
PROJECT_DIR=$(pwd)

echo "=== CLAUDE.md (global) ===" && cat ~/.claude/CLAUDE.md
echo "=== CLAUDE.md (local) ===" && cat "$PROJECT_DIR/CLAUDE.md" 2>/dev/null || echo "(none)"
echo "=== rules/ ===" && find "$PROJECT_DIR/.claude/rules" -name "*.md" 2>/dev/null | while read f; do echo "--- $f ---"; cat "$f"; done
echo "=== skill descriptions ===" && grep -r "^description:" "$PROJECT_DIR/.claude/skills" ~/.claude/skills 2>/dev/null
echo "=== hooks ===" && python3 -m json.tool "$PROJECT_DIR/.claude/settings.local.json" 2>/dev/null | grep -A 30 '"hooks"'
echo "=== MCP servers ===" && python3 -m json.tool "$PROJECT_DIR/.claude/settings.local.json" 2>/dev/null | grep -A 10 '"enabledMcp"'
echo "=== allowedTools count ===" && python3 -m json.tool "$PROJECT_DIR/.claude/settings.local.json" 2>/dev/null | grep -c "Bash\|mcp__"
```

## Step 2: Collect conversation evidence

Write to `~/.claude/tmp/audit/` so subagents can read it (subagents cannot access `/tmp`).

```bash
PROJECT_PATH=$(pwd | sed 's|/|-|g' | sed 's|^-||')
CONVO_DIR=~/.claude/projects/-${PROJECT_PATH}
SCRATCH=~/.claude/tmp/audit
mkdir -p "$SCRATCH"

# Extract last 15 conversations
for f in $(ls -t "$CONVO_DIR"/*.jsonl 2>/dev/null | head -15); do
  base=$(basename "$f" .jsonl)
  cat "$f" | jq -r '
    if .type == "user" then "USER: " + (.message.content // "")
    elif .type == "assistant" then
      "ASSISTANT: " + ((.message.content // []) | map(select(.type == "text") | .text) | join("\n"))
    else empty
    end
  ' 2>/dev/null | grep -v "^ASSISTANT: $" > "$SCRATCH/${base}.txt"
done

ls -lhS "$SCRATCH"
echo "SCRATCH=$SCRATCH"
```

## Step 3: Launch parallel diagnostic agents

Spin up **three focused subagents** in parallel, each examining one diagnostic dimension:

### Agent A — Context Layer Audit
Prompt:
```
Read: ~/.claude/CLAUDE.md, [project]/CLAUDE.md, [project]/.claude/rules/**, [project]/.claude/skills/**/SKILL.md

This project is tier: [SIMPLE / STANDARD / COMPLEX] — apply only the checks appropriate for this tier.

Tier-adjusted CLAUDE.md checks:
- ALL tiers: Is CLAUDE.md short and executable? No prose, no background, no soft guidance.
- ALL tiers: Does it have build/test commands?
- STANDARD+: Is there a "Verification" section with per-task done-conditions?
- STANDARD+: Is there a "Compact Instructions" section?
- COMPLEX only: Is content that belongs in rules/ or skills already split out?

Tier-adjusted rules/ checks:
- SIMPLE: rules/ is NOT required — do not flag its absence.
- STANDARD+: Language-specific rules (e.g., Rust, Lua) should be in rules/ not CLAUDE.md.
- COMPLEX: Path-specific rules should be isolated; no rules in root CLAUDE.md.

Tier-adjusted skill checks:
- SIMPLE: 0–1 skills is fine. Do not flag absence of skills.
- ALL tiers: If skills exist, descriptions should be <12 tokens and say WHEN to use.
- STANDARD+: Low-frequency skills should have disable-model-invocation: true.

Output: bullet points only, state the detected tier at the top, grouped by: [CLAUDE.md issues] [rules/ issues] [skills description issues]
```

### Agent B — Control + Verification Layer Audit
Prompt:
```
Read: [project]/.claude/settings.local.json, [project]/CLAUDE.md, [project]/.claude/skills/**/SKILL.md
Also read conversation files: [batch of SCRATCH/*.txt]

This project is tier: [SIMPLE / STANDARD / COMPLEX] — apply only the checks appropriate for this tier.

Tier-adjusted hooks checks:
- SIMPLE: Hooks are optional. Only flag if a hook is broken (e.g., fires on wrong file types).
- STANDARD+: PostToolUse hooks expected for the primary language(s) of the project.
- COMPLEX: Hooks expected for all frequently-edited file types found in conversations.
- ALL tiers: If hooks exist, verify pattern field is present to avoid firing on all edits.

allowedTools hygiene (ALL tiers):
- Flag stale one-time commands (migrations, setup scripts, path-specific operations).
- Flag dangerous operations (rm -rf, force-push, brew uninstall, launchctl unload).

Tier-adjusted verification checks:
- SIMPLE: No formal verification section required. Only flag if Claude declared done without running any check.
- STANDARD+: CLAUDE.md should have a Verification section with per-task done-conditions.
- COMPLEX: Each task type in conversations should map to a verification command or skill.

Output: bullet points only, state the detected tier at the top, grouped by: [hooks issues] [allowedTools to remove] [verification gaps]
```

### Agent C — Behavior Pattern Audit
Prompt:
```
Read: ~/.claude/CLAUDE.md, [project]/CLAUDE.md
Read conversation files: [batch of SCRATCH/*.txt]

Analyze actual behavior against stated rules:

1. Rules violated: Find cases where CLAUDE.md says NEVER/ALWAYS but Claude did the opposite. Quote both the rule and the violation.
2. Repeated corrections: Find cases where the user corrected Claude's behavior more than once on the same issue. These are candidates for stronger rules.
3. Missing local patterns: Find project-specific behaviors the user reinforced in conversation but that aren't in local CLAUDE.md.
4. Missing global patterns: Find behaviors that would apply to any project (not just this one) that aren't in ~/.claude/CLAUDE.md.
5. Anti-patterns present: Check for these specific anti-patterns:
   - CLAUDE.md used as wiki/documentation
   - Skills covering too many unrelated tasks
   - Claude declaring done without running verification
   - Subagents used without tool/permission constraints
   - User having to re-explain the same context across sessions

Output: bullet points only, grouped by: [rules violated] [repeated corrections] [add to local CLAUDE.md] [add to global CLAUDE.md] [anti-patterns]
```

Assign conversation files to agents B and C based on size:
- Large files (>50KB): 1-2 per agent
- Medium (10-50KB): 3-5 per agent
- Small (<10KB): up to 10 per agent

## Step 4: Synthesize and present

Aggregate all agent outputs into a single report with these sections:

### 🔴 Critical (fix now)
Rules that were violated, missing verification definitions, dangerous allowedTools entries.

### 🟡 Structural (fix soon)
CLAUDE.md content that belongs elsewhere, missing hooks for frequently-edited file types, skill descriptions that are too long.

### 🟢 Incremental (nice to have)
New patterns to add, outdated items to remove, global vs local placement improvements.

---

**Stop condition:** After presenting the report, ask:
> "Should I draft the changes? I can handle each layer separately: global CLAUDE.md / local CLAUDE.md / hooks / skills."

Do not make any edits without explicit confirmation.
