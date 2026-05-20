# VeloDB Best Practices Skill

An AI agent skill that teaches Claude, Antigravity, Cursor, Windsurf, and other AI coding assistants how to design optimal Apache Doris / VeloDB tables and size clusters.

## What It Does

Instead of guessing, the skill guides agents through an **interactive 4-step workflow**:

1. **Gather** — Ask 5 discovery questions (data, volume, queries, loading, environment)
2. **Classify** — Match workload to use case template(s)
3. **Design** — Produce `CREATE TABLE` + sizing together with inline comments explaining WHY
4. **Validate** — Check every decision against 37 rules

For troubleshooting, a parallel 3-step workflow (Gather Evidence → Diagnose → Fix) ensures agents never suggest fixes without seeing the table DDL, query, profile, and cluster info first.

## Install

### Via npx skills (recommended)
```bash
# Install all skills
npx skills add velodb/agent-skills

# Install to specific agents
npx skills add velodb/agent-skills -a claude-code -a antigravity

# List available skills first
npx skills add velodb/agent-skills --list
```

### Manual install
```bash
git clone https://github.com/velodb/agent-skills.git
cd agent-skills
./skills/velodb-best-practices/install.sh
```

## What's Included

```
skills/velodb-best-practices/
├── SKILL.md                    # Main instructions (4-step workflow)
├── AGENTS.md                   # Compiled reference (all rules inline)
├── install.sh                  # Multi-agent installer
└── references/                 # 50 reference files
    ├── schema-model-*.md       # Data model rules (4)
    ├── schema-partition-*.md   # Partition rules (4)
    ├── schema-bucket-*.md      # Bucket rules (5)
    ├── schema-keys-*.md        # Sort key rules (5)
    ├── schema-types-*.md       # Data type rules (5)
    ├── schema-index-*.md       # Index rules (7)
    ├── schema-mv-*.md          # Materialized view rules (3)
    ├── schema-props-*.md       # Table properties (2)
    ├── schema-cache-*.md       # Cache rules (2)
    ├── usecase-*.md            # Use case templates (7)
    ├── sizing-*.md             # Cluster sizing guides (4)
    ├── start-*.md              # Getting started (2)
    └── velocli-*.md            # CLI capabilities + manual handoffs (1)
```

## Supported Agents

| Agent | Config Dir |
|-------|-----------|
| Antigravity | `.agents/skills/` |
| Claude Code | `.claude/skills/` |
| Cursor | `.cursor/skills/` |
| Windsurf | `.windsurf/skills/` |
| Kimi Code | `.kiro/skills/` |
| Gemini CLI | `.gemini/skills/` |

## License

Apache-2.0
