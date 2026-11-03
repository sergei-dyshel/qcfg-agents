# Notes for agents

This repository is an agent plugin named `qcfg-agents` for Claude Code, Codex
and GitHub Copilot CLI. The repository root is the plugin directory. Each skill
lives in `skills/<name>/SKILL.md`. A file that several skills share lives in
`context/` and is linked from a skill by a relative path, such as
`../../context/handling-ambiguity.md`.

## Writing content

- **Read [writing-style](skills/writing-style/SKILL.md) before writing or
  editing any prose here**: skills, this file, manifest descriptions and commit
  messages.
- **This repository is public. Never mention a private repository or anything
  inside it**: no names, paths, commands or symbols. Check a skill copied from
  elsewhere for such references before committing it. See
  `Public content is self-contained` in
  [writing-style](skills/writing-style/SKILL.md).
- **Keep a skill's `name` frontmatter identical to its directory name**, as the
  Agent Skills specification requires.

## Manifests

- **Keep the plugin name and version identical across the manifests.** The name
  is in all four: `.claude-plugin/plugin.json` (Claude Code),
  `.codex-plugin/plugin.json` (Codex), `.github/plugin/marketplace.json` (GitHub
  Copilot CLI, as both the marketplace and the plugin name) and `plugin.json`
  (the agent-plugins.org manifest). The version is in the first two.
- **Run `claude plugin validate .` after changing a manifest.** Its plugin
  directory checks (e.g. a missing README) only matter for a submission to the
  Claude Code plugin directory and do not fail the validation.

## Installing and discovery

- **Claude Code: symlink the repository as `~/.claude/skills/qcfg-agents`.**
  Claude Code loads it as a full plugin named after
  `.claude-plugin/plugin.json`, not after the symlink. It discovers both
  `skills/` and `agents/`, as `qcfg-agents:<name>`.
- **Codex: symlink the repository as `~/.codex/plugins/qcfg-agents`.** Codex
  reads only `skills/`, as `.codex-plugin/plugin.json` declares. It does not
  load agents, so a skill that spawns a plugin agent does not work in Codex.
- **GitHub Copilot CLI: register the repository as a local marketplace** with
  `copilot plugin marketplace add <repository dir>`, then run
  `copilot plugin install qcfg-agents@qcfg-agents`. Copilot loads the plugin
  from the directory without copying it.
- **New or renamed skills and agents appear only in a new session**, or after
  `/reload-plugins` in Claude Code. To check discovery without leaving the
  current session, ask a fresh one:
  `cd /tmp && claude -p "List every skill available to your Skill tool, one per line" --model haiku --max-turns 1`.
  For agents, ask for "every subagent_type value available to your Agent tool"
  instead.

## Writing plugin agents

- **List a skill in the frontmatter `skills:` by its scoped name**
  (`qcfg-agents:<skill>`). It preloads the skill only when the agent runs as a
  subagent. Under `claude --agent <name>` it is not preloaded, so also tell the
  agent in its body to load the skill with the Skill tool (verified with Claude
  Code 2.1.302).
- **Take tools away with `disallowedTools`.** Plugin agents ignore
  `permissionMode`, `hooks` and `mcpServers`.
- **Make an agent that may need input stop and state what it needs in its final
  message.** Subagents never get `AskUserQuestion`.
