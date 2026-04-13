# Echo

A Claude Code subagent that manages your project's coding rules. You teach it how you want things done — during code review, while explaining your architecture, whenever — and it maintains `.claude/rules/` files so Claude Code follows them next time.

No framework. No assumptions. Just a subagent.

## What it does

You're reviewing code. You see Claude used `FactoryBot.build` when it should have used `FactoryBot.create` for integration tests. You say:

```
@echo integration tests must use FactoryBot.create, not .build — we need records persisted to the database
```

Echo creates `.claude/rules/testing-factorybot-integration.md`:

```markdown
---
paths:
  - "spec/integration/**"
  - "spec/features/**"
---
# FactoryBot in integration tests

Use `FactoryBot.create`, not `.build`. Integration and feature specs hit the database — `.build` produces unpersisted objects that silently break assertions.
```

Next time Claude works on an integration test, that rule loads automatically.

## What it is

Three files:

```
.claude/
├── agents/
│   └── echo.md        # the subagent
├── skills/
│   └── review/
│       └── SKILL.md          # /review command
└── hooks/                    # (optional, see below)
```

**echo.md** — A subagent with `memory: project`. It knows where rules live, how to write them with proper glob scoping, and how to check for existing rules before creating duplicates. Its memory accumulates context about your project's conventions over time.

**review/SKILL.md** — A `/review` skill that reads the current diff, loads all existing rules, and flags anything that looks off. When you confirm a violation, it talks to echo to create or strengthen the rule.

That's it. Everything else is standard Claude Code primitives doing what they already do.

## Install

<TBD>

## Usage

### During code review

The main loop. You review what Claude did, and when something's wrong:

```
@echo we always use Sidekiq for background jobs in this project, never ActiveJob directly
```

```
@echo the ORM convention is: never call .save! in a controller — wrap mutations in a service object
```

```
@echo migrations must always be reversible. No `execute` without a matching `down` block.
```

Echo creates a scoped rule file each time. If a rule already exists for that topic, it updates it instead of creating a duplicate.

### Teaching it things

You don't have to wait for a violation. Explain things whenever:

```
@echo our BDD tests use Turnip with Capybara. Feature files go in spec/features/, step definitions in spec/steps/. We follow the given-when-then pattern strictly — no and/but keywords.
```

```
@echo when writing database queries, always use .includes() to prevent N+1s. Our CI runs bullet gem and will reject PRs with N+1 warnings.
```

### Browsing your rules

They're just markdown files:

```bash
ls .claude/rules/
# => api-validation.md
#    testing-factorybot-integration.md
#    db-no-save-in-controllers.md
#    migrations-reversible.md
#    sidekiq-over-activejob.md
```

Read them, edit them, delete them. They're your files.

## How rules get scoped

Echo writes glob-scoped frontmatter when the rule applies to specific files:

```yaml
---
paths:
  - "src/api/**"
---
```

Claude Code only loads that rule when working on matching files. Rules without `paths` frontmatter apply everywhere.

Echo decides the scope based on what you tell it. If you say "in tests," it scopes to your test directory. If you describe a general convention, it leaves the scope open.

## Specialized agents

Echo is your generalist. Over time, you might want domain-specific agents that own a slice of your codebase:

```
.claude/agents/
├── echo.md
├── bdd-specialist.md      # knows your test framework deeply
├── db-agent.md            # knows the ORM, query patterns, indexing
└── migration-agent.md     # knows deploy pipeline, rollback strategy
```

These are separate subagents with their own memory. Echo creates rules; specialists apply deep knowledge when Claude is working in their domain. They're independent — add them when you need them, not before.

## Design decisions

**Why `.claude/rules/` and not CLAUDE.md?** Rules files are modular. Each rule is independent, glob-scoped, and deletable without touching anything else. CLAUDE.md is for project-wide context that rarely changes.

**Why a subagent and not just a skill?** Echo needs persistent memory across sessions. It remembers what rules exist, what patterns it's seen, what you've told it. A skill is stateless — it loads when invoked and forgets. A subagent with `memory: project` accumulates knowledge.

**Why not a database / vector store / knowledge graph?** Because `.claude/rules/*.md` files are already the right primitive. Claude Code loads them natively, scopes them by path, and re-reads them after compaction. Adding infrastructure on top of what already works would be adding complexity for its own sake.

**Why not an MCP server?** Same reason. The filesystem is the interface. Rules are files. Claude Code already knows how to read files.

## Requirements

- Claude Code (any recent version with subagent and rules support)
- A project with a `.claude/` directory
