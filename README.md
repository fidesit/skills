# utiska-skills

Reusable [Claude Code](https://claude.ai/code) skills for multi-agent workflows.

## Skills

### peer-review

A multi-agent review loop that generates documents through iterative AI peer review. Five parallel reviewer subagents (accuracy, completeness, overlap, probe, temporal) evaluate each draft, revise until all pass or max iterations are reached, then present the result for human approval.

**Usage**: copy `skills/peer-review/` into your project's `.claude/skills/` directory.

```
.claude/skills/peer-review/
├── SKILL.md                        # Orchestrator prompt
└── prompts/
    ├── source-fetcher.md           # Fetches and digests authoritative sources
    ├── draft-generator.md          # Generates the initial document
    ├── reviewer-accuracy.md        # Cross-references claims against sources
    ├── reviewer-completeness.md    # Audits against a required checklist
    ├── reviewer-overlap.md         # Detects rule overlaps and dead ends
    ├── reviewer-probe.md           # Runs test cases against the document
    ├── reviewer-temporal.md        # Verifies date-sensitive rules
    └── revision-agent.md           # Produces revised versions from feedback
```

**Invoke**: `/peer-review` in Claude Code, or ask Claude to "peer review" a document.

## License

MIT
