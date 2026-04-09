# AI Agents Skills

This repository is a curated pack of reusable agent skills focused on Golang, Linux, performance, and infrastructure workflows.

## Credits

This repo includes skills copied and adapted from these community sources:

- [samber/cc-skills-golang](https://github.com/samber/cc-skills-golang)
- [sickn33/antigravity-awesome-skills](https://github.com/sickn33/antigravity-awesome-skills)

It also includes custom skills built for Linux performance analysis, network debugging, load-test interpretation, and infrastructure capacity planning.

## Included Skills

### From `samber/cc-skills-golang`

`golang-performance` · `golang-concurrency` · `golang-grpc` · `golang-observability` · `golang-error-handling` · `golang-context` · `golang-safety` · `golang-troubleshooting` · `golang-benchmark`

### From `sickn33/antigravity-awesome-skills`

`bash-linux` · `error-detective`

### Custom skills

`linux-perf-analysis` · `linux-network-debug` · `load-test-interpreter` · `infra-capacity-planning`

## What A Skill Looks Like

Each skill lives in its own folder:

```text
skill-name/
├── SKILL.md
├── scripts/        # optional
├── examples/       # optional
└── resources/      # optional
```

`SKILL.md` is the important file. That is the instruction set the agent loads when the context matches or when you invoke the skill explicitly.

## How To Use These Skills

Most agent tools support one of these patterns:

- Auto-load: the agent detects the relevant skill from your prompt or task context.
- Explicit invoke: you ask for a specific skill by name, often with `@skill-name`.

Examples:

```text
Why is my goroutine count growing over time?
```

```text
Use @linux-perf-analysis to investigate why p99 latency spiked at 2pm.
```

```text
Use @load-test-interpreter on these JMeter results, then use @infra-capacity-planning to size the deployment.
```

## Using One Shared Skills Repo

Yes, you can keep one central source of truth for your skills and point other agent tools to it with symlinks. This is usually the cleanest setup because:

- You clone once.
- You update once with `git pull`.
- Every tool sees the same skill versions.

Recommended layout:

```bash
git clone git@github.com:abhiabhi0/ai-agents-skills.git ~/ai-agents-skills
```

Then link the shared skills into each tool's expected path.

## Tool Setup

### Antigravity

```bash
mkdir -p ~/.gemini/antigravity
ln -s ~/ai-agents-skills ~/.gemini/antigravity/skills
```

### Claude Code

```bash
mkdir -p ~/.claude
ln -s ~/ai-agents-skills ~/.claude/skills
```

### Cursor

```bash
mkdir -p ~/.cursor
ln -s ~/ai-agents-skills ~/.cursor/skills
```

### OpenAI Codex

Codex setups vary a bit by environment. If your Codex workflow supports a skills directory, point it at this repo or symlink this repo into the directory your setup expects.

Common pattern:

```bash
mkdir -p ~/.codex
ln -s ~/ai-agents-skills ~/.codex/skills
```

If your Codex environment uses a project-local skills folder, you can also link it there:

```bash
mkdir -p .agent
ln -s ~/ai-agents-skills .agent/skills
```

### Project-local setup for any agent

```bash
mkdir -p /path/to/your-project/.agent
ln -s ~/ai-agents-skills /path/to/your-project/.agent/skills
```

## Windows Alternative

Windows does not use Unix symlinks as smoothly in every setup, but you still have good options.

### Option 1: Directory junction

This is usually the easiest Windows equivalent for folders:

```powershell
mklink /J $env:USERPROFILE\.cursor\skills $env:USERPROFILE\ai-agents-skills
```

You can use the same pattern for other tools by changing the destination path.

### Option 2: Real symlink

If Developer Mode is enabled or you have elevated permissions:

```powershell
New-Item -ItemType SymbolicLink -Path "$env:USERPROFILE\.claude\skills" -Target "$env:USERPROFILE\ai-agents-skills"
```

### Option 3: Copy instead of link

If a tool does not behave well with links on Windows, copy the folder into the tool-specific path and update it manually when needed.

## Updating All Tools At Once

If you use the shared-repo plus symlink approach:

```bash
cd ~/ai-agents-skills
git pull
```

That updates the source once, and every linked tool picks up the same changes.

## Notes For Public Use

- Keep upstream attribution in this README when copying or adapting skills.
- If you modify copied skills, it helps to document what changed and why.
- If you add new skills, keep one folder per skill with a clear `SKILL.md`.

## Contributing

Pull requests are welcome for:

- new infrastructure and performance skills
- better examples and helper scripts
- cleaner tool-specific installation notes
- fixes to copied or adapted upstream skills
