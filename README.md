# personal-wiki

A Claude Code skill for maintaining a personal wiki in an Obsidian vault. Use it from any Claude Code session — including outside the vault — to ingest URLs, PDFs, and images; bookmark links; enrich existing notes; query the wiki; or lint the structure. All operations are plain filesystem reads and writes on the vault's markdown files (no MCP, no AppleScript).

See [`SKILL.md`](SKILL.md) for the full workflows, voice rules, and folder conventions Claude follows at runtime.

## Install

Clone into your Claude Code skills directory:

```bash
git clone https://github.com/robschmitt/personal-wiki-skill.git ~/.claude/skills/personal-wiki
```

Or scope it to a single project:

```bash
git clone https://github.com/robschmitt/personal-wiki-skill.git <project>/.claude/skills/personal-wiki
```

## Setup

Two things to set up: a vault-path config file, and a permission scope that includes the vault.

### 1. Point the skill at your vault

Create `~/.config/personal-wiki/vault-path` with the absolute path to your Obsidian vault:

```bash
mkdir -p ~/.config/personal-wiki
echo "/absolute/path/to/your/vault" > ~/.config/personal-wiki/vault-path
```

The skill reads this at session start. If the file is missing, Claude will ask once and write it for you.

### 2. Widen permission scope (recommended for cross-project use)

If you'll invoke the skill from projects other than the vault itself, add the vault root and the skill's config dir to `~/.claude/settings.json`:

```json
{
  "permissions": {
    "allow": [
      "Read(/absolute/path/to/your/vault/**)",
      "Write(/absolute/path/to/your/vault/**)",
      "Edit(/absolute/path/to/your/vault/**)",
      "Read(/Users/<you>/.config/personal-wiki/**)"
    ],
    "additionalDirectories": [
      "/absolute/path/to/your/vault",
      "/Users/<you>/.config/personal-wiki"
    ]
  }
}
```

Why both fields are needed: allow rules grant permission *within* the current project's scope. `additionalDirectories` widens that scope to cover paths outside the project's working directory. Without `additionalDirectories`, Claude Code prompts for vault paths every time the skill runs from another project, even with the allow rules in place. Together they make cross-project use friction-free.

### 3. Folder conventions (optional)

The skill reads `<vault-root>/Wiki Conventions.md` at session start if present, treating it as the authoritative source for your folder taxonomy. See [`SKILL.md`](SKILL.md#folder-taxonomy) for the role this file plays. The skill works without it — it'll propose folders directly — but a curated conventions note keeps suggestions consistent over time.

## Workflows

Once active, the skill greets with:

> Personal Wiki ready. Workflows: **ingest [source]** | **bookmark [url]** | **enrich [note]** | **query [topic]** | **discuss [topic]** | **lint**

See [`SKILL.md`](SKILL.md#workflows) for what each one does.
