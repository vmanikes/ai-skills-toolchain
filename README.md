# my-ai-skills

A collection of skills for AI coding agents such as Claude Code, Cursor, and others that support the [Agent Skills](https://agentskills.io) format. Each skill teaches the agent a specialized workflow -- giving it structured, repeatable instructions for tasks that benefit from consistency.

## Skills

| Skill | Description |
|-------|-------------|
| [git-smart-commit](git-smart-commit/) | Analyzes local git changes and generates conventional commit messages |
| [pr-review](pr-review/) | Performs structured pull request reviews across correctness, security, performance, and more |
| [pr-preflight](pr-preflight/) | Pre-push check that adds observability, tests, alerts, and performance fixes to local changes using repo conventions |

### git-smart-commit

Offloads the cognitive burden of writing commit messages. The agent inspects your staged/unstaged changes and produces messages following the [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/#specification) spec.

**What it does:**
- Checks branch safety (never commits to `main`/`master`)
- Auto-detects and runs project formatters (Prettier, Black, gofmt, etc.) on changed files only
- Generates structured `type(scope): subject` commit messages
- Suggests splitting unrelated changes into multiple commits
- Asks before pushing to remote

**Trigger phrases:** "commit my changes", "smart commit", "generate a commit message"

### pr-review

Runs a multi-dimensional review of a pull request, covering seven categories drawn from production engineering best practices.

**What it checks:**
- **Scope & Intent** -- is the PR focused and small enough to review confidently?
- **Correctness & Edge Cases** -- boundary conditions, failure modes, idempotency
- **Security & Privacy** -- secrets, input validation, PII handling, least privilege
- **Observability** -- structured logs, metrics, tracing, actionable alerts
- **Performance & Cost** -- N+1 queries, caching, cloud cost impact
- **API & Backward Compatibility** -- non-breaking changes, versioning, migration steps
- **Data & Migrations** -- backward-compatible migrations, rollback strategy, batching safety
- **Docs & Developer Ergonomics** -- README updates, local test instructions, operational callouts

Every finding cites exact file and line locations. Output is tiered into Critical / Suggestion / Nit so the author knows what blocks the merge.

**Trigger phrases:** "review this PR", "review pull request #123", "code review my branch"

### pr-preflight

An author-side check you run on local changes *before* opening a PR. It fills observability, testing, and performance gaps first, following the conventions already in your repo.

**What it does:**
- Discovers the repo's existing logging, metrics, tracing, test, and alerting conventions
- Suggests and (on approval) adds **metrics** (counter, gauge, histogram), **logs** (debug, info, warn, error), and **traces**, making sure every log carries enough context (correlation IDs, entity IDs, inputs, outcome, error) and checking label cardinality and PII
- Suggests **unit and integration tests** and writes them failing-first: each test is seen failing without the change before it is seen passing with it
- Proposes actionable **alerts** based on what the change does (error rate, latency, job absence, queue lag, saturation)
- Flags **performance** issues (N+1, missing pagination/timeouts, unbounded memory) and pre-push hygiene (debug prints, secrets, undocumented config)
- Ends with a readiness report: Ready / Ready with follow-ups / Not ready

**Trigger phrases:** "preflight", "PR sanity check", "is this ready for a PR", "check my changes before I push"

## Installation

These skills use the [Agent Skills](https://agentskills.io) format: a folder with a `SKILL.md` file. Any AI coding agent that supports it can load them, including Claude Code and Cursor.

Agents look for skills one level deep (`<skills-dir>/<skill-name>/SKILL.md`), so link or copy each skill folder into your agent's skills directory rather than cloning the whole repo into it.

| Agent | Personal (all projects) | Per-project (shared via the repo) |
|-------|-------------------------|-----------------------------------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| Other agents | See your agent's docs for its skills directory | |

### Personal use (available across all your projects)

Clone the repo once, then symlink each skill into your agent's skills directory so `git pull` keeps them up to date:

```bash
git clone git@github.com:vmanikes/my-ai-skills.git ~/my-ai-skills
```

```bash
SKILLS_DIR=~/.claude/skills   # or ~/.cursor/skills, etc.
mkdir -p "$SKILLS_DIR" && for d in ~/my-ai-skills/*/; do [ -f "$d/SKILL.md" ] && ln -sfn "${d%/}" "$SKILLS_DIR/$(basename "$d")"; done
```

### Per-project (shared with your team via the repo)

Copy the skill folders you want into your agent's project skills directory and commit them:

```bash
mkdir -p .claude/skills   # or .cursor/skills, etc.
cp -R ~/my-ai-skills/pr-preflight .claude/skills/
```

### Verify

Start a new agent session and ask:

> "What skills do you have available?"

## Skill Structure

Each skill follows the same layout:

```
skill-name/
├── SKILL.md          # Required -- main instructions (frontmatter + workflow)
├── CHECKLIST.md      # Optional -- detailed reference material
├── examples.md       # Optional -- usage examples
└── scripts/          # Optional -- utility scripts
```

The `SKILL.md` file contains YAML frontmatter (`name`, `description`) that agents use for skill discovery, and a markdown body with the agent's instructions. Some agents support extra frontmatter fields; for example, `disable-model-invocation: true` (used by `git-smart-commit`) tells Claude Code to run the skill only when invoked explicitly.

## Prerequisites

Some skills rely on external tools being available on your system:

| Skill | Requirements |
|-------|-------------|
| git-smart-commit | `git` |
| pr-review | `git`, [`gh` CLI](https://cli.github.com/) (for GitHub PR reviews) |
| pr-preflight | `git`; [`rg`](https://github.com/BurntSushi/ripgrep) and [`fd`](https://github.com/sharkdp/fd) optional (falls back to `grep`) |

## Contributing

To add a new skill:

1. Create a new directory with a descriptive, hyphenated name (e.g., `api-design-review`)
2. Add a `SKILL.md` with the required frontmatter:
   ```yaml
   ---
   name: your-skill-name
   description: What it does and when to use it. Written in third person.
   ---
   ```
3. Keep `SKILL.md` under 500 lines -- use separate reference files for detailed content
4. Include trigger terms in the description so agents can auto-discover the skill
5. Keep each skill self-contained -- don't reference other skills in this repo, since users may install only one
6. Test the skill by invoking it in at least one agent and verifying the output

## License

MIT
