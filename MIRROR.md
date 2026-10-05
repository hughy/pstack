# pstack mirror

This repo mirrors [`cursor/plugins/pstack`](https://github.com/cursor/plugins/tree/main/pstack) so pstack installs in pi, Claude Code, and Codex. `README.md` is upstream's, unchanged. Cursor users should install upstream with `/add-plugin pstack`.

## What differs from upstream

- `skills/poteto-mode/SKILL.md` and `skills/make-bot-ui/SKILL.md`: frontmatter `name` uses the folder name (`poteto-mode`, `make-bot-ui`). pi's `/skill:` command stops at the first space, so the upstream names `Poteto Mode` and `Make Bot UI` can't be invoked.
- `agents-pi/`: upstream's two agents plus a `skills:` line. pi-subagents starts custom agents without the skill catalog, and `poteto-mode`, `how`, and `why` are hidden from it (`disable-model-invocation: true`).
- `.claude-plugin/` and `.codex-plugin/`: plugin manifests. Codex also reads `.claude-plugin/marketplace.json`.
- `MIRROR.md`: this file.

Everything else matches upstream byte for byte. The `upstream` branch holds the exact copy.

## Before you install

Skills run bundled scripts, for example `skills/poteto-mode/scripts/`. Read them first.

pstack was written for Cursor. Outside Cursor, these parts are untested:

- Skills that spawn subagents call Cursor's `Task` tool with `subagent_type: generalPurpose`. Your agent has to map that to its own subagent tool.
- `/setup-pstack` writes `~/.cursor/rules/pstack-models.mdc`, which only Cursor loads automatically. `swarm`, `arena`, and `interrogate` name the full path, so an agent elsewhere can read it. `how`, `why`, `architect`, and `reflect` name only `pstack-models.mdc`, so outside Cursor they likely fall back to their defaults. `poteto-mode` never names the file and keeps its own defaults.
- Default model names (grok, opus 5.5, sol) are Cursor slugs.

## Install

Clone the repo first. The examples use `~/dev/pstack`.

### pi

pstack loads as a skills folder plus an agents folder, which also works where pi packages are blocked.

1. Add the skills folder to `~/.pi/agent/settings.json`:

   ```json
   { "skills": ["~/dev/pstack/skills"] }
   ```

2. Link the agents (needs pi-subagents):

   ```bash
   mkdir -p ~/.pi/agent/agents
   ln -s ~/dev/pstack/agents-pi ~/.pi/agent/agents/pstack
   ```

3. Run `/reload` or start a new session. Check with `/skill:poteto-mode`.

If another skill already uses a pstack name, pi keeps the first one it finds and warns. Rename yours, or exclude pstack's with `"skills": ["~/dev/pstack/skills", "!~/dev/pstack/skills/<name>"]`.

### Claude Code

```bash
claude plugin marketplace add ~/dev/pstack
claude plugin install pstack@pstack
```

### Codex

```bash
codex plugin marketplace add ~/dev/pstack
```

Then install pstack from `/plugins`.

### Any agent with Agent Skills

```bash
npx skills add ~/dev/pstack
```

This copies the skills only, without the agents.

## Sync with upstream

`upstream` holds Cursor's files exactly. `main` is `upstream` plus the edits above. Never copy Cursor's files onto `main` directly.

```bash
git clone --depth 1 --filter=blob:none --sparse https://github.com/cursor/plugins.git /tmp/cursor-plugins
git -C /tmp/cursor-plugins sparse-checkout set pstack
git switch upstream
rsync -a --delete --exclude .git --exclude MIRROR.md /tmp/cursor-plugins/pstack/ ./
git add -A
git commit -m "upstream: cursor/plugins/pstack @ $(git -C /tmp/cursor-plugins rev-parse --short HEAD)"
git switch main
git merge upstream
```

If `agents/` changed, copy each file into `agents-pi/` again and re-add its `skills:` line. If a merge conflict touches a skill `name`, keep upstream's change and restore the folder-name form.
