# SAC Data Action Knowledge Base

A curated knowledge base for building, validating, and troubleshooting SAP Analytics Cloud (SAC) Data Actions.

Maintained by the Cognivion SAC practice team.

---

## Purpose

This repo gives Claude (and your developers) a structured, up-to-date reference for:
- Writing correct data action logic the first time
- Diagnosing blank output and context failures
- Capturing expert patterns from trusted community sources
- Verifying data action output against expected values

---

## Structure

```
sac-da-knowledge/
│
├── skill/
│   └── SKILL.md                  ← Claude skill instructions
│
├── references/                   ← SAP Help extracts, updated each QRC
│   ├── advanced-formulas.md
│   ├── data-action-steps.md
│   ├── version-management.md
│   ├── allocations.md
│   ├── data-locking.md
│   ├── multi-actions.md
│   ├── context-and-scope.md
│   └── whats-new-latest.md
│
├── patterns/                     ← Curated fixes for known issues
│   ├── blank-output-causes.md
│   ├── copy-step-issues.md
│   ├── allocation-gotchas.md
│   ├── formula-syntax-errors.md
│   ├── version-scope-issues.md
│   └── cross-model-tips.md
│
├── sources/                      ← Trusted expert contributors
│   ├── experts.md
│   └── reading-list.md
│
├── templates/                    ← Developer intake templates
│   ├── model-context-sheet.md
│   ├── data-action-requirement.md
│   └── verification-checklist.md
│
├── team-knowledge/               ← Internal learnings (anonymised)
│   ├── delivery-patterns.md
│   └── client-gotchas.md
│
└── changelog.md                  ← What changed and when
```

---

## Maintenance Schedule

| Trigger | Action | Owner |
|---|---|---|
| SAC QRC release | Update `references/whats-new-latest.md`, review changed steps/APIs | Lead consultant |
| Team hits a new issue | Add pattern entry to `patterns/` | Developer who fixed it |
| Expert post captured | Add to `patterns/` with attribution, update `sources/reading-list.md` | Any team member |
| Client go-live | Add anonymised learnings to `team-knowledge/` | Project lead |

---

## How to use with Claude

1. Add this repo to your Claude Code project
2. Claude reads the relevant reference and pattern files before generating any data action
3. Pass model context using the template in `templates/model-context-sheet.md`
4. After running the data action in SAC, export results and paste back for verification

---

## License

Internal use — Cognivion. Not for redistribution.
