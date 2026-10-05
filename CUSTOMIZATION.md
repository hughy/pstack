# How to grow a personal stack on top of this mirror

This file describes how to build a customized stack (working name `hstack`) on top of this mirror without losing the ability to pull from upstream. None of this work has started. The file records the plan so the first step can begin from it.

## Why a layer and not a fork

pstack's skills reference each other by name. At this commit, `/poteto-mode` appears 120 times across `skills/`, `agents/`, and `docs/`. Nine skills call Cursor's `Task` tool with `subagent_type: generalPurpose`. Eight `SKILL.md` files read `~/.cursor/rules/pstack-models.mdc`. Renaming or rewriting skills in place turns every upstream merge into a conflict. A second directory that shadows pstack by load order avoids that. The mirror stays a byte-exact input, and the personal layer holds every opinion.

Regenerate the counts with:

```bash
grep -rho '/poteto-mode\b' skills agents docs | wc -l
grep -rl 'subagent_type' skills | wc -l
grep -rl 'pstack-models' skills --include=SKILL.md | wc -l
```

## Target layout

```
hstack/
  .claude-plugin/marketplace.json    lists two plugins: pstack at ./pstack, hstack at ./
  .claude-plugin/plugin.json         name: hstack
  .codex-plugin/plugin.json
  pstack/                            git subtree of this repo's main branch. Never edited here.
  skills/
    hughy-mode/                      the entry point. Wraps poteto-mode, then applies deltas.
    setup-hstack/                    writes agent files and a host-neutral models file.
    <shadowed skill>/                same folder name as a pstack skill. Wins by load order.
    <own skill>/                     skills pstack lacks.
  agents/                            pi-subagents agents with model, skills, and tools pinned.
  adapters/
    pi.md                            Task -> subagent mapping, model slug table, config path.
    claude.md
    codex.md
  DIVERGENCE.md                      one entry per shadowed skill.
```

## How the mirror changes

The mirror becomes a dependency of hstack. Three things change in this repo.

The `main` branch keeps only portability fixes: the manifests, the folder-name frontmatter, and `MIRROR.md`. A change that `cursor/plugins` would not accept belongs in hstack instead. The sync recipe in `MIRROR.md` does not change.

`agents-pi/` moves to `hstack/agents/`. The copies exist because pi-subagents starts custom agents without the skill catalog. In hstack, agent files carry `model:`, `skills:`, and a tool allowlist. That replaces most of what `pstack-models.mdc` does, because pi's native per-role model setting is the agent file. Once hstack owns agents, the pi install section of `MIRROR.md` shrinks to the skills path.

`.claude-plugin/marketplace.json` in hstack lists two plugins. Today this repo's manifest lists one plugin with `source: "./"`. The hstack manifest adds `pstack` with `source: "./pstack"`. Codex reads the same file, so one `plugin marketplace add` installs both.

## How each host loads the two layers

pi keeps the first skill it finds for a name and warns about later ones. List hstack first and exclude each shadowed pstack skill to silence the warning:

```json
{
  "skills": [
    "~/dev/hstack/skills",
    "~/dev/hstack/pstack/skills",
    "!~/dev/hstack/pstack/skills/poteto-mode"
  ]
}
```

Replace the agents symlink:

```bash
rm ~/.pi/agent/agents/pstack
ln -s ~/dev/hstack/agents ~/.pi/agent/agents/hstack
```

Claude Code and Codex install both plugins from the one marketplace. Unverified: both hosts appear to namespace skills by plugin (`/pstack:how` and `/hstack:how`), which would make the exclusion list unnecessary there. Confirm before the hub skill names a namespace.

## Steps

### 1. Keep a friction log before writing code

Use pstack as installed for one to two weeks. Each time it fights your setup, add a line to `DIVERGENCE.md`. Known entries:

- Skills call the `Task` tool. pi has `subagent`.
- Default model names are Cursor slugs.
- `/setup-pstack` writes `~/.cursor/rules/pstack-models.mdc`.
- Your global `AGENTS.md` and pstack's `unslop`, `no-comments`, and `technical-writing` skills both set prose and comment rules. They disagree on details such as plan file locations. Pick one owner for each rule.

### 2. Scaffold hstack

Create the repo and vendor the mirror:

```bash
mkdir ~/dev/hstack && cd ~/dev/hstack && git init
git subtree add --prefix pstack ~/dev/pstack main
mkdir skills agents adapters .claude-plugin .codex-plugin
cp ~/dev/pstack/agents-pi/* agents/
```

Write the two manifests, repoint `~/.pi/agent/settings.json`, and replace the agents symlink. Run `/reload`. Confirm that `/skill:poteto-mode` still resolves through `~/dev/hstack/pstack/skills`.

### 3. Add the config layer

Write `adapters/pi.md`. It tells the agent how to read a pstack skill on pi. Where a skill names `Task`, use `subagent`. Where it names `pstack-models.mdc`, read `~/.config/hstack/models.md`. Where it names a Cursor slug, look up the pi id in the table at the end of the file.

Write `setup-hstack`. It writes `agents/*.md` with `model:` set to the chosen pi model ids and writes `~/.config/hstack/models.md` for the skills that still read a role table.

Shadow one of the eight `pstack-models.mdc` skills only if the adapter paragraph is not enough to redirect it.

### 4. Write the hub skill

`hughy-mode` starts as a wrapper. Its `SKILL.md` says: read `pstack/skills/poteto-mode/SKILL.md` in full, read `adapters/pi.md`, then apply the deltas below. The deltas are playbook additions and removals, and routing to skills pstack lacks.

Convert the wrapper to a full copy when the deltas exceed about one third of the original. Past that point an upstream merge costs more than it returns.

### 5. Shadow skills by churn

For each skill you want to change, measure how often upstream touches it:

```bash
git -C ~/dev/pstack log --oneline --since='6 months ago' upstream -- skills/<name> | wc -l
```

A skill with low churn and heavy edits gets copied and owned. A skill with high churn gets wrapped or left alone. Record the choice, the reason, and the upstream commit the copy was taken from in `DIVERGENCE.md`.

### 6. Fold in skills pstack lacks

Move personal skills from `~/.agents/skills/` into `hstack/skills/` so one install carries everything, and so `hughy-mode` playbooks can route to them by name.

### 7. Resync monthly

1. Sync the mirror. Follow the recipe in `MIRROR.md`.
2. Pull the subtree:

   ```bash
   git -C ~/dev/hstack subtree pull --prefix pstack ~/dev/pstack main
   ```

3. For each copied skill in `DIVERGENCE.md`, diff upstream between the recorded commit and the new head. Port what matters. Update the recorded commit.

## Decisions still open

- Same repo or a new repo. A new repo with a subtree keeps this mirror's `main` upstreamable and makes the two-plugin marketplace straightforward. A directory inside this repo needs less setup but reworks the manifests and the pi skills path here.
- Wrap or replace `poteto-mode` first. Wrapping keeps the 120 cross-references valid and merges cheap. Replacing gives a mode file you own from the first day.
- Whether `agents-pi/` stays in the mirror after hstack owns agents.
