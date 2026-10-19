---
name: editing-skills
description: Use when about to edit, create, or delete an agent skill. Shared skills are sourced from the agent-skills repository; downstream-local skills stay with their owning repository.
---

## Where installed skills come from

Agent skills are installed by home-manager under `~/.claude/skills/`, `~/.agents/skills/`, and `~/.codex/skills/`. These managed paths link to the source skill directory. Edit the owning repository rather than an installed link.

Shared skill sources and resources live in the workspace's `agent-skills/skills/<name>/` directory. The Nix catalog at `agent-skills/catalog.nix` selects shared skills for consumers. Downstream overlays may add local skill paths through their own `machines.claude.skills` and `machines.codex.skills` configuration; keep those files in the downstream repository.

## Edit a shared skill

1. Edit its directory in the workspace `agent-skills` checkout.
2. If the change also updates a consumer, use the same feature branch in both repositories. Add `agent-skills` to the workspace with `addsrc agent-skills git@github.com:goromal/agent-skills.git` if it is not present.
3. Test anixpkgs integration using a temporary local flake input if needed. Before pushing, restore the remote feature-branch URL and lock it with `nix flake update agent-skills`.
4. Open the `agent-skills` PR and link it from the consumer PR. Merge the skill source PR first, then repin consumers to the merged `main` revision.

## Create a shared skill

1. Add `skills/<skill-name>/SKILL.md` in `agent-skills`; keep references, examples, scripts and assets alongside it.
2. Add `{ name = "<skill-name>"; file = ./skills/<skill-name>/SKILL.md; }` to `catalog.nix`. Entries apply to Claude and Codex by default; set `frameworks` when needed.
3. Link supporting resources relative to the skill folder and update downstream locks.

## Edit a downstream-local skill

Change it in the repository that owns the overlay or module contributing the skill path. Keep the directory self-contained, including references. Its configuration can add a `{ name, file }` entry to the framework's skills option.

## Delete or replace a shared skill

Remove its entry from `catalog.nix` and delete its source folder, then update downstream locks. To replace a shared skill on one machine, list its name in `machines.agents.excludedSkills` and add the local skill path with that name in the downstream module.
