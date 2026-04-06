# Agentic Engineering Pipeline

An interactive visualization of four multi-agent AI workflows covering implementation, documentation, research, and security — each with independent cross-model verification via GPT-5.4 subagents.

## 🔗 Live Demo

**[View the Pipeline Visualization](https://kuldeeps48.github.io/agentic-enginerring/)**

## Overview

Four specialized pipelines, each led by a primary agent and verified by GPT-5.4 scrutinizer subagents. All scrutiny roles are model-pinned to GPT-5.4; primary agents run on whatever model is active.

### Pipelines

| Pipeline           | Lead Agent              | Flow                                                                       | GPT Scrutiny                                     |
| ------------------ | ----------------------- | -------------------------------------------------------------------------- | ------------------------------------------------ |
| **Implementation** | Architect → Implementer | Scope → Design → Scrutinize → Handoff → Implement → Review × 2 → Summary   | Architecture + Code review (2 rounds each)       |
| **Documentation**  | Documenter              | Backlog → Draft business doc → Draft technical doc → Verify → Fix → Update | GPT Doc Scrutinizer                              |
| **Research**       | Researcher              | Draft → Self-proofread → Proofread → Accuracy check → Reconcile → Save     | Proofreader + Accuracy Checker (separate passes) |
| **Security**       | Pen Tester              | Scope → Threat intelligence → OWASP assessment → Scrutinize × 2 → Report   | GPT Pen Test Scrutinizer (2 rounds)              |

### Implementation Pipeline

The implementation flow has two operating modes:

- **Mode A (architect-driven):** Used when `.architect-output/architecture.md` exists. Architect researches the codebase via Explore subagents, writes the architecture document, gets it scrutinized by GPT Architect Scrutinizer, then formally hands off to Implementer.
- **Mode B (direct):** For small, well-scoped tasks only. If work grows past roughly 20 budget units, Implementer stops and recommends routing through Architect.

After implementation, Code Reviewer runs **two full review cycles**, each with self-verification followed by two GPT Code Review Scrutinizer rounds. All findings at every severity level are fixed before cycle 2. If issues remain after cycle 2, they are escalated rather than looped.

### Documentation Pipeline

Documenter works one feature at a time from `documents/documentation-backlog.md`. For each feature it invokes Draft Documenter (Claude Opus 4.6) twice — once for the business document, once for the technical document — then verifies both against the source code using GPT Doc Scrutinizer, applies fixes, and updates the backlog status.

### Research Pipeline

Researcher drafts the document, self-proofreads before any external review, then runs two separate GPT-5.4 passes: one for writing quality (GPT Research Proofreader) and one for factual accuracy and technical correctness (GPT Research Accuracy Checker). Findings are deduplicated and independently verified before being applied. The final file is saved to disk under `documents/research/`.

### Security Pipeline

Pen Tester performs a read-only grey-box assessment. It starts with online threat intelligence research, then audits dependencies and configuration, then steps through the OWASP Top 10. GPT Pen Test Scrutinizer runs twice — round 1 verifies exploitability and hunts for missed attack paths; round 2 checks the merged report for blind spots, severity consistency, and remediation quality. The deliverable is a structured report, not a code change.

### Agents

**Primary agents (model not pinned):**

- **Architect** — Scopes, researches via Explore, writes and saves the architecture document, runs GPT scrutiny, and performs the formal handoff
- **Implementer** — Executes tasks from the architecture document or operates in direct mode; owns the fix loop, changed-files tracking, and implementation summary
- **Code Reviewer** — Full review of changed files, self-verifies, then runs two GPT scrutiny rounds per cycle
- **Documenter** — Manages the documentation backlog, delegates drafting and verification, applies fixes, updates backlog status
- **Researcher** — Writes the research document, self-proofreads, reconciles dual GPT reviews, and saves the final file
- **Pen Tester** — Scope, threat intelligence, OWASP assessment, two-round GPT scrutiny, final report

**Subagents (model-pinned to GPT-5.4 unless noted):**

- **Explore** — Lightweight read-only codebase research; spawned by Architect and Implementer to avoid consuming parent context budget _(model not pinned)_
- **Draft Documenter** — Reads source modules and drafts business and technical documents _(Claude Opus 4.6)_
- **GPT Architect Scrutinizer** — Checks architecture documents for requirements coverage, DB design, API consistency, permissions, migration sanity, and task split validity
- **GPT Code Review Scrutinizer** — Verifies code review findings for accuracy, false positives, and missed issues
- **GPT Doc Scrutinizer** — Checks business and technical documents against source code and each other
- **GPT Research Proofreader** — Checks prose quality: grammar, clarity, structure, consistency
- **GPT Research Accuracy Checker** — Checks claims: technical correctness, logical consistency, source validity, currency
- **GPT Pen Test Scrutinizer** — Used twice: round 1 for exploitability and missed vulnerabilities; round 2 for severity consistency and remediation quality

### Key Design Principles

- **Cross-model verification** — Every significant output is independently checked by a GPT-5.4 subagent
- **Separation of concerns** — Proofreading and accuracy checking are separate subagent passes so each check does one job well
- **Context isolation** — Subagents run in their own context windows and don't consume the parent's token budget
- **Auto-fix loop** — Code review findings are fixed in full before cycle 2; the second cycle is verification, not selective cleanup
- **Two-cycle maximum** — If issues remain after 2 review cycles, they are escalated with an assessment rather than looped indefinitely
- **Sequential documentation** — Documenter finishes one complete feature (both documents + verification + backlog update) before starting the next

## Features of the Visualization

- Tab-based navigation across all four pipelines (Overview, Research, Implementation, Documentation, Security)
- Executive flow summary for each pipeline showing the 4 major stages
- Expandable detail sections with full step-by-step flows, subagent cards, artifact listings, and model-pinning information
- Overview cards that deep-link directly into each pipeline's detail view
- Fully responsive, self-contained HTML — no build step or dependencies

## Usage

Open `index.html` in any browser, or visit the [GitHub Pages deployment](https://kuldeeps48.github.io/agentic-enginerring/).

## License

MIT
