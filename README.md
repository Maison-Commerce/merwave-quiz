# AI Skills

A shared collection of [Claude Code Skills](https://docs.claude.com/en/docs/claude-code/skills) maintained by the Maison Commerce team.

Each subfolder in this repo is a self-contained skill (with a `SKILL.md` at its root) that you install into your own Claude Code environment.

## Available skills

| Skill | What it does |
|---|---|
| [`styleguide-from-figma`](./styleguide-from-figma) | Inspects a Figma URL via the Figma MCP and writes a replicatable `design.md` + `styleguide.html`. |
| [`styleguide-from-url`](./styleguide-from-url) | Opens a live URL via chrome-devtools MCP and extracts design tokens into a `design.md` styleguide. |

## Install

Skills live in `~/.claude/skills/` on your machine. Each skill is its own subfolder containing a `SKILL.md`.

### Option 1 — Clone the whole repo into your skills folder (recommended)

This keeps every skill in sync. When the team adds a new skill, you just `git pull`.

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/Maison-Commerce/ai-skills.git
```

To update later:

```bash
cd ~/.claude/skills/ai-skills
git pull
```

Claude Code discovers skills recursively, so nesting them under `ai-skills/` works fine.

### Option 2 — Install a single skill

If you only want one skill, copy just that folder into `~/.claude/skills/`:

```bash
mkdir -p ~/.claude/skills
cd ~/.claude/skills
git clone https://github.com/Maison-Commerce/ai-skills.git _tmp
cp -R _tmp/styleguide-from-figma .
rm -rf _tmp
```

The final layout should look like:

```
~/.claude/skills/
├── styleguide-from-figma/
│   └── SKILL.md
└── styleguide-from-url/
    └── SKILL.md
```

### Verify the install

Restart Claude Code, then run:

```
/help
```

You should see the skills listed in the available skills section. You can also confirm with:

```bash
ls ~/.claude/skills
```

## Invoke a skill

Skills auto-trigger when your prompt matches their description, but you can also invoke them explicitly.

**Auto-trigger** — just describe the task:

> "Build a styleguide from https://stripe.com"

Claude will load `styleguide-from-url` automatically.

**Explicit invocation** — type `/` followed by the skill name:

```
/styleguide-from-url https://stripe.com
/styleguide-from-figma <figma-url>
```

## MCP server requirements

Some skills depend on MCP servers. Make sure these are connected before invoking:

- `styleguide-from-figma` → **Figma MCP** (connect via Claude Code's MCP settings)
- `styleguide-from-url` → **chrome-devtools MCP** (connect via Claude Code's MCP settings)

If a skill fails on first run, check that its MCP server is connected with `/mcp`.

## Contributing a new skill

1. Create a new folder at the repo root with a kebab-case name.
2. Add a `SKILL.md` with YAML frontmatter (`name`, `description`, `compatibility`, `metadata`).
3. Open a PR — once merged, teammates pick it up with `git pull`.
