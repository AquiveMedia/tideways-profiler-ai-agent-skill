# Tideways Profiler — Claude Code Skill

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill that autonomously controls a browser to log in to Tideways, navigate the dashboard, and analyze APM data to diagnose PHP performance bottlenecks, generate root-cause hypotheses, and prescribe actionable fixes.

## What it does

When you mention **Tideways** in a Claude Code conversation, this skill activates and guides the analysis through:

1. **Performance Overview** — Identify high-impact transactions by sorting on Impact%
2. **Observations Triage** — Review automated performance hints from Tideways
3. **Slow SQL Analysis** — Trace slow queries back to code, run EXPLAIN locally
4. **Trace Deep-Dive** — Break down wall time vs CPU time, identify IO-bound vs CPU-bound
5. **Deploy Correlation** — Compare before/after release metrics
6. **Root-Cause Hypotheses** — Structured findings with confidence scores and concrete fixes

Every finding follows the format: **Symptom → Mechanism → Root Cause → Fix → Confidence**

## Framework support

- **Generic PHP** — Works with any PHP application profiled by Tideways
- **Magento 2** — Automatically loads Magento-specific patterns (EAV queries, FPC analysis, plugin chains, layout rendering) when a Magento 2 project is detected

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI
- [playwright-cli](https://www.npmjs.com/package/@anthropic-ai/playwright-cli) skill for browser automation (Tideways interaction via snapshots)
- A Tideways account with profiling data

## Installation

Copy the skill to your Claude Code skills directory:

```bash
# Clone
git clone https://github.com/AquiveMedia/claude-skill-tideways-profiler.git

# Copy to Claude Code skills directory
cp -r claude-skill-tideways-profiler ~/.claude/skills/tideways-profiler
```

Or symlink for easy updates:

```bash
ln -s /path/to/claude-skill-tideways-profiler ~/.claude/skills/tideways-profiler
```

## File structure

```
tideways-profiler/
├── SKILL.md                        # Main skill (loaded on trigger)
├── references/
│   ├── navigation-map.md           # Tideways UI navigation & commands
│   ├── bottleneck-detection.md     # Generic PHP bottleneck patterns
│   ├── root-cause-analysis.md      # Root cause analysis framework
│   └── magento2.md                 # Magento 2 extensions (loaded when detected)
├── LICENSE
└── README.md
```

## Usage

In any Claude Code conversation with a PHP project:

```
> My category pages are slow, can you check tideways?

> What happened after the last deploy on tideways?

> Analyze the slow SQLs in tideways
```

The skill uses `playwright-cli` to navigate Tideways in a browser, take snapshots, and cross-reference findings with your local codebase and database.

## Built with

This skill was created following Anthropic's official skill authoring guidelines:

- [How to create custom skills](https://support.claude.com/en/articles/12512198-how-to-create-custom-skills)
- [The Complete Guide to Building Skills for Claude](https://resources.anthropic.com/hubfs/The-Complete-Guide-to-Building-Skill-for-Claude.pdf) (PDF)

## License

MIT
