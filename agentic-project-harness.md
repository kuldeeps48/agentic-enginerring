# Agentic Project Harness

This document defines the baseline harness every agent-friendly repository should carry. The goal: make the repo easy for humans and agents to understand, modify, verify, deploy, recover, and operate — without relying on hidden context.

The harness is not the product. It is the operating layer around the product: instructions, checks, scripts, docs, pipelines, observability, recovery, and feedback loops.

## Standards

Three requirement levels apply throughout this document.

| Standard | Meaning |
| --- | --- |
| **Required** | Every serious repo needs this. Missing it creates avoidable execution risk. |
| **Production-required** | Required before production traffic, sensitive data, payments, or customer commitments. |
| **Project-dependent** | Required only when stage, team size, compliance surface, or growth motion justifies it. |

## Recommended Repo Shape

Use this skeleton by default. Adjust names only when the ecosystem has a stronger convention.

```text
.
├── AGENTS.md or CLAUDE.md
├── README.md
├── .env.example
├── .gitignore
├── .dockerignore
├── docs/
├── scripts/
├── changelog/
├── scratch-pad/
├── infra/
├── migrations/
├── tests/
└── .github/workflows/   # or equivalent CI/CD folder
```

Folder names matter less than discoverability. A new maintainer or agent should find instructions, commands, docs, verification, release history, and runbooks within minutes.

## Readiness Levels

| Level | Meaning | Required when |
| --- | --- | --- |
| **0** | Context harness | The repo may be edited by an agent or collaborator. |
| **1** | Development harness | The repo has repeated work, multiple contributors, or meaningful complexity. |
| **2** | Production harness | The repo runs production workloads, stores sensitive data, handles money, or has customer commitments. |
| **3** | Product harness | The project needs customer learning, support, experiments, or go-to-market loops. |

---

## Level 0: Context Harness

Make the repo legible.

| Item | Standard | Purpose | Minimum bar |
| --- | --- | --- | --- |
| Main agent instructions | Required | Defines repo-specific behavior, guardrails, commands, architecture facts, and forbidden actions. | Root `AGENTS.md`, `CLAUDE.md`, or equivalent. Names dangerous actions, verification commands, deployment constraints, and source-of-truth docs. |
| README | Required | Fast entry point for humans. | Describes what the project is, how to run and test it, and where deeper docs live. |
| Skills | Required for agent-heavy repos | Encodes repeatable workflows agents should follow for specialized tasks. | Repo-local skills for major domains: backend, frontend, release, infra, security, payments, data. |
| MCPs | Required when external systems matter | Gives agents structured access to external tools and systems. | Documented MCP servers, auth assumptions, allowed actions, and prohibited actions. |
| Docs folder | Required | Stores durable product, architecture, operational, and decision context. | `docs/` exists with current entry points. Critical context never lives only in chat. |
| Scratch-pad folder | Required | Safe place for temporary investigation notes and throwaway analysis. | `scratch-pad/` exists, gitignored or clearly marked non-authoritative. |
| Scripts folder | Required when commands become non-trivial | Holds repeatable operational and development commands. | `scripts/` exists for setup, verification, data, deploy, or maintenance automation. |
| Decision records | Project-dependent | Preserves *why* important choices were made. | ADRs or equivalent for architecture, vendor, security, payment, data, and launch decisions. |

---

## Level 1: Development Harness

Make local development repeatable and reviewable.

| Item | Standard | Purpose | Minimum bar |
| --- | --- | --- | --- |
| `.env.example` | Required | Documents required environment variables without leaking secrets. | Every required variable listed with safe placeholders. Comments explain non-obvious values. |
| `.gitignore` | Required | Keeps build output, secrets, and generated noise out of git. | Covers language, framework, editor, OS, env, build, cache, and temp artifacts. |
| `.dockerignore` | Required when Docker is used | Keeps Docker build contexts small and prevents accidental secret inclusion. | Excludes git data, env files, dependency caches, build/test output, and local temp files. |
| Local bootstrap | Required | Lets a new machine become useful quickly. | One documented setup path: dependencies, env, local services, migrations, seed data. |
| Linting | Required | Catches style, correctness, and suspicious patterns before review. | One documented command that runs locally and in CI. |
| Formatting | Required | Keeps diffs focused on behavior. | One documented command or hook that formats supported languages. |
| Pre-commit hooks | Recommended | Moves cheap checks earlier than CI. | Cover formatting, linting, secret scanning, large files, and file hygiene where practical. |
| Tests | Required | Correctness oracle for agents and humans. | Unit tests for core logic, integration tests for risky boundaries, and one documented run command. |
| Type checks | Required when the stack supports it | Catches interface drift and unsafe assumptions. | One documented command, runs locally and in CI. |
| Automated migrations | Required when a database schema exists | Makes schema changes deterministic and reviewable. | Versioned migrations, local apply command, production apply path, rollback or recovery notes. |
| Changelog folder | Required for release-bearing repos | Captures deployable changes and verification evidence. | `changelog/` with a template and one entry per meaningful release, deploy, or operational change. |
| Dependency management | Required | Makes builds reproducible. | Lockfiles committed where appropriate. Upgrade process and supported runtime versions documented. |

---

## Level 2: Production Harness

Make the project shippable, observable, and recoverable.

| Item | Standard | Purpose | Minimum bar |
| --- | --- | --- | --- |
| CI pipeline | Required | Verifies every branch before merge. | Runs lint, format check, type checks, tests, builds, migration checks, and security checks appropriate to the repo. |
| CD pipeline | Production-required | Deploys repeatably with auditable inputs. | Manual or automated workflow with environment separation, changelog linkage, rollback notes, and explicit production safeguards. |
| Infra health checks | Production-required | Detects broken runtime dependencies and bad deploys. | Health endpoints, service checks, DB connectivity, worker checks, queue checks, external dependency checks. |
| DB backup automation | Production-required when persistent data exists | Protects production data from loss. | Scheduled backups, retention policy, offsite storage, encryption, alerting on backup failure. |
| DB restore automation | Production-required when backups exist | Proves backups are usable. | Documented restore procedure plus periodic drills against a safe target. |
| Monitoring | Production-required | Surfaces system health and user-impacting failures. | Metrics, logs, traces or structured events, dashboards, and SLIs. |
| Alerting | Production-required | Turns important failures into action. | Alerts for uptime, error rate, latency, queue backlog, disk usage, backup freshness, billing, certificate expiry, and critical third-party failures. |
| Runbooks | Production-required | Makes incidents and maintenance repeatable. | Runbooks for deploy, rollback, restore, incident triage, credential rotation, and dependency outages. |
| Security audits | Production-required | Catches risky configuration and code before attackers do. | Dependency scanning, secret scanning, static analysis, permission review, production config review. |
| Vulnerability scanning | Production-required | Tracks known CVEs in dependencies and images. | Automated scans in CI plus scheduled scans for deployed images or infrastructure. |
| Secrets management | Production-required | Prevents secret leakage and unclear credential ownership. | Secrets out of git, scoped by environment, rotated intentionally, documented by name and purpose only. |
| Performance testing | Production-required for high-traffic or latency-sensitive systems | Measures capacity before users do. | Load tests for critical paths with baseline latency, throughput, error rate, and resource usage. |
| Performance optimization | Project-dependent | Converts performance data into engineering work. | Documented bottlenecks, budgets, profiling method, and current known limits. |
| Incident history | Recommended | Preserves operational learning. | Incidents and near misses produce short writeups: cause, impact, fix, prevention. |

---

## Level 3: Product Harness

Help the project learn from users and operate as a real product.

| Item | Standard | Purpose | Minimum bar |
| --- | --- | --- | --- |
| User feedback collection | Required for user-facing products | Captures what users actually experience. | In-app feedback, support email, forms, surveys, or structured issue intake. |
| Feedback analysis | Required once feedback volume is non-trivial | Turns raw feedback into product decisions. | Tagging, prioritization, recurring review cadence, and links from decisions back to evidence. |
| License documentation | Required | Makes usage rights explicit. | `LICENSE` file, private/proprietary notice, or internal-use statement. |
| Legal documentation | Production-required for user-facing products | Covers user-facing obligations. | Privacy policy, terms, data deletion process, security contact, and compliance notes as relevant. |
| Feature flags | Project-dependent | Decouples deploy from release. | Flags have owner, default state, expiry plan, audit trail, and kill-switch behavior. |
| Experimentation framework | Project-dependent | Measures product changes without corrupting core metrics. | Clear ownership, assignment logic, success metrics, guardrail metrics, and cleanup plan. |
| Customer support system | Required for customer-facing products | Tracks user issues to resolution. | Support inbox or ticketing tool with triage rules, response expectations, and escalation paths. |
| Issue tracking system | Required for multi-step work | Keeps product and engineering work accountable. | Backlog with owners, priorities, states, and links to PRs, releases, incidents, or user reports. |
| Marketing campaigns | Project-dependent | Creates repeatable acquisition and launch work. | Campaign plan, target audience, channels, messaging, assets, tracking links, and result review. |
| Outreach campaigns | Project-dependent | Builds direct learning and distribution loops. | Prospect or user lists, outreach scripts, follow-up cadence, consent boundaries, and outcome tracking. |

---

## Minimum Baseline for a New Repo

Start here before serious implementation:

- Root agent instructions
- `README.md`
- `docs/`
- `scratch-pad/`
- `.env.example`
- `.gitignore`
- `.dockerignore` (when Docker is used)
- Lint, format, and test commands
- Local bootstrap command or doc
- CI pipeline
- Changelog template

## Minimum Baseline Before Production

Add before production traffic:

- Automated migrations
- Manual or automated CD pipeline
- Production health checks
- Monitoring dashboards
- Alerting for user-impacting failures
- Backup automation
- Restore drill documentation
- Runbooks for deploy, rollback, incident triage, and restore
- Security and vulnerability scanning
- Secrets management documentation
- Legal and license documentation
- Support and issue intake path

## Agent Readiness Checklist

Use this when deciding whether a repo is ready for agentic work.

- Can an agent find authoritative project instructions in under a minute?
- Can an agent discover the correct build, lint, test, migration, and deploy commands without asking?
- Can an agent run verification locally without hidden state?
- Are secrets represented by placeholders, not copied values?
- Are dangerous production actions guarded by explicit confirmation?
- Are generated files, scratch files, build output, and credentials excluded from git?
- Is there a documented place for temporary notes?
- Is there a documented place for durable decisions?
- Are schema changes automated and reviewable?
- Is CI a trustworthy merge gate?
- Is CD repeatable and auditable?
- Is rollback documented?
- Can production health be checked without reading private logs?
- Can the database be restored from backups?
- Are alerts mapped to an owner or response path?
- Would a new maintainer know where user feedback, support issues, and release history live?

## Anti-Patterns

Avoid these. They make agentic work brittle.

- Critical instructions live only in chat history.
- Agent instructions contradict the README, docs, or CI.
- Scripts require undocumented local state.
- `.env.example` omits required variables or contains real secrets.
- CI runs a weaker command set than local verification.
- Production deploys depend on a human remembering manual steps.
- Backups exist but restores are never tested.
- Alerts fire without an owner, severity, or action.
- Feature flags accumulate without owners or expiry dates.
- Scratch notes are treated as durable product decisions.

## Review Cadence

Review the harness at these moments:

- New repo creation
- First external collaborator or agent handoff
- Before first production deploy
- After each incident or failed deploy
- Before adding payments, sensitive data, or compliance-sensitive workflows
- Quarterly for active production systems

## Operating Principle

If a repo needs hidden memory to be changed safely, the harness is incomplete. Encode that memory in files, scripts, checks, dashboards, or documented workflows so the next human or agent can act from ground truth.
