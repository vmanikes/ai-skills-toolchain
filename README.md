# ai-skills-toolchain

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

These skills use the [Agent Skills](https://agentskills.io) format: a folder with a `SKILL.md` file. Any AI coding agent that supports it can load them, including Claude Code, Cursor, Codex, GitHub Copilot, and Gemini CLI.

### With the GitHub CLI (recommended)

[`gh` CLI](https://cli.github.com/) 2.105+ installs skills into the right directory for your agent and can update them later:

```bash
gh skill install vmanikes/ai-skills-toolchain pr-preflight --agent claude-code --scope user
```

- `--agent`: `claude-code`, `cursor`, `codex`, `github-copilot`, `gemini-cli`, and more (`gh skill install --help` lists them all)
- `--scope`: `project` (default, inside the current repo) or `user` (all projects)
- `--all`: install every skill in this repo
- `pr-preflight@v1.0.0`: install a specific release

Update installed skills with:

```bash
gh skill update --all
```

### Manually

Agents look for skills one level deep (`<skills-dir>/<skill-name>/SKILL.md`), so link or copy each skill folder into your agent's skills directory rather than cloning the whole repo into it.

| Agent | Personal (all projects) | Per-project (shared via the repo) |
|-------|-------------------------|-----------------------------------|
| Claude Code | `~/.claude/skills/` | `.claude/skills/` |
| Cursor | `~/.cursor/skills/` | `.cursor/skills/` |
| Other agents | See your agent's docs for its skills directory | |

Clone the repo once, then symlink each skill into your agent's skills directory so `git pull` keeps them up to date:

```bash
git clone git@github.com:vmanikes/ai-skills-toolchain.git ~/ai-skills-toolchain
```

```bash
SKILLS_DIR=~/.claude/skills   # or ~/.cursor/skills, etc.
mkdir -p "$SKILLS_DIR" && for d in ~/ai-skills-toolchain/*/; do [ -f "$d/SKILL.md" ] && ln -sfn "${d%/}" "$SKILLS_DIR/$(basename "$d")"; done
```

To share a skill with your team, copy it into your project's skills directory and commit it:

```bash
mkdir -p .claude/skills   # or .cursor/skills, etc.
cp -R ~/ai-skills-toolchain/pr-preflight .claude/skills/
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
   license: MIT
   ---
   ```
3. Keep `SKILL.md` under 500 lines -- use separate reference files for detailed content
4. Include trigger terms in the description so agents can auto-discover the skill
5. Keep each skill self-contained -- don't reference other skills in this repo, since users may install only one
6. Test the skill by invoking it in at least one agent and verifying the output
7. Validate locally with `gh skill publish --dry-run` before opening a PR

## Releases

Releases are automated by [`.github/workflows/skills.yml`](.github/workflows/skills.yml):

- **Pull requests:** every PR runs `gh skill publish --dry-run` to validate all skills against the Agent Skills spec.
- **Merges to main** that change a skill: the workflow picks the next version from the commit messages since the last release and publishes a GitHub release with `gh skill publish`.

| Commits since the last release include | Version bump |
|----------------------------------------|--------------|
| `type!: ...` or `BREAKING CHANGE` | major (`v2.0.0`) |
| `feat: ...` / `feat(scope): ...` | minor (`v1.1.0`) |
| anything else | patch (`v1.0.1`) |

To force a bump, run the **Skills** workflow manually from the Actions tab and choose `patch`, `minor` or `major`. Changes to only the README, LICENSE or workflow files don't create a release. Release tags are protected by a ruleset, so they can't be moved or deleted once published.

## License

MIT
