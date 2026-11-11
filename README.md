# DesignMakeathon

A design system guideline and a compliance audit for the Creao web app (agent.creao.ai).

## Files

| File | What it is |
|---|---|
| [`design-system-guideline.md`](design-system-guideline.md) | The rulebook (v1.1). It covers token, component and layout rules, eight screen patterns (P1–P8), rules for agent-generated UI, and four enforcement gates: lint, build scan, render matrix and review. The enforcement code is included in full. |
| [`compliance-audit.md`](compliance-audit.md) | The audit of today's production app against the rulebook. It has 21 findings (3 S1, 9 S2, 9 S3), each mapped to the rule it breaks and the gate that would have caught it, plus a fix order. |

## Reading order

1. **Audit, section 1 (Summary).** What's wrong today, in one page.
2. **Guideline, sections 0 and 5.** How to use the rules, and the screen patterns.
3. **Guideline, section 7.** How the rules are enforced.
4. **Audit, section 4.** The enforcement code running on real output.

## Evidence

- Findings F-01 to F-18 come from the public production build of agent.creao.ai, fetched 2026-11-09. No private source code was used.
- Findings F-19 to F-21 come from a short signed-in check of the live app on 2026-11-11: the chat home, one conversation, the Agents page and Discover.
- **What was run:** the lint rules, the build scan and its ratchet were taken from the guideline as written and run against that build. Their output in the audit is real.
- **What was not run:** the render matrix and the token contract test need the repo and a seeded test workspace. Guideline section 7.10 lists what has and hasn't been run.
