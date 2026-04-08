# Project Standards

## Documentation Requirements

- **Decision trail**: All decision points must be recorded in project docs with date and rationale. Use the format: `**Decision (YYYY-MM-DD):** [decision text]`
- **Research sourcing**: All research findings must include linked sources. Never state facts without attribution.
- **Outdated info**: When information becomes outdated, mark it with `[OUTDATED as of YYYY-MM-DD: reason]` rather than deleting. This preserves the decision history.
- **Open questions**: Unresolved items go in `docs/open-questions.md` with context on why they're open and what's needed to resolve them.
- **Tech stack notes**: Record technology choices inline with the decisions they support, not in separate files.

## Cross-Linking

- Do NOT duplicate information that exists in another repo. Link to it instead.
- Master strategy: [business-idea-automation/docs/master-strategy.md](https://github.com/monikaalmasy/business-idea-automation/blob/main/docs/master-strategy.md)
- Related repos: [spawni](https://github.com/monikaalmasy/spawni), [niche-seo-saas](https://github.com/monikaalmasy/niche-seo-saas), [spawni-growth](https://github.com/monikaalmasy/spawni-growth), [business-idea-automation](https://github.com/monikaalmasy/business-idea-automation)

## Handoff Readiness

Any agent or person picking up this project should be able to understand:
1. What has been decided and why
2. What is still open
3. What information is current vs outdated
4. Where to find related context in other repos

## Feedback

**Feedback (2026-04-08):** Don't keep strategic insights to yourself. If research data contradicts a current decision or suggests a significantly better approach, raise it IMMEDIATELY — don't wait for the user to discover it. The user prefers strong business cases over comfort. Challenge decisions when evidence supports a different path.

**Why:** Reddit research supported niche-specific branding from the start, but this was only surfaced after the user independently proposed it. Should have been recommended proactively.

**How to apply:** When research reveals a better approach than the current plan, present it as a recommendation with evidence, even if it contradicts a recent decision.
