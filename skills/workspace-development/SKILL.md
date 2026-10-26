---
name: workspace-development
description: Coordinate coding work in an anix devshell workspace, especially cross-repository changes involving anixpkgs, flasks, anixdata, Nix flake inputs, deployment, shared branches, commits, pushes, and pull requests. Use whenever the current directory may be a workspace sources/ directory or a task spans anixpkgs and one or more dependency repositories.
---

# Workspace development

## Establish the workspace

At the start of coding work, inspect the current directory before choosing where to clone or edit. Treat it as a devshell workspace's `sources/` directory when its basename is `sources` and its parent has workspace markers such as `shell.nix`, `.envrc`, or `data/`. Also inspect its immediate children for Git repositories.

When in such a `sources/` directory:

- Keep all development for the task inside that workspace.
- Reuse repositories already present there.
- Add a needed repository to the workspace rather than cloning or editing it elsewhere. Prefer the workspace's `dev`/`addsrc` tooling when available.
- Put generated datasets, logs, exports, databases, caches, and other task data that must not be committed in the workspace-level `data/` directory, not inside a source repository. This is the persistent data directory beside `sources/` (that is, `../data/` when the current directory is `sources/`).
- Inspect `git status`, the current branch, and remotes separately in every repository before editing. Preserve unrelated work.

Do not infer the following repository roles unless the named repository is actually present in this workspace:

- `anixpkgs/` is the monorepo hub. It packages dependencies and deploys code to local or remote machines.
- `flasks/` owns the source for the Flask-based UIs that `anixpkgs` packages and deploys. Make UI implementation changes in `flasks`, not in stale or generated copies under `anixpkgs`.
- `anixdata/` owns binary/blob assets such as images. Keep blobs out of `anixpkgs`; add them to `anixdata` and update the `anixdata` flake input in `anixpkgs`.

Distinguish uncommitted task data from versioned assets: keep the former in the workspace `data/` directory and commit the latter to `anixdata/` when the project must distribute or reproduce them.

## Coordinate cross-repository work

For every cross-repository change involving `anixpkgs`:

1. Choose one descriptive side-branch name, normally `dev/<feature>` or `fix/<bug>`.
2. Use that exact branch name in `anixpkgs` and every involved dependency repository. Never do cross-repository development directly on a default branch.
3. Implement and test dependency changes in their owning repositories. Use temporary local source overrides for rapid integration testing when useful.
4. At authorized delivery checkpoints, commit coherent work and push the shared branch in every involved repository. Do not leave the hub pointing at an unpushed commit.
5. Open dependency PRs and the `anixpkgs` hub PR. Cross-reference them in both directions; the `anixpkgs` PR description must link every dependency PR and state merge/re-pin ordering.

For `agent-skills`, the catalog contains paths relative to that repository. Merge its entries with consumer `machines.claude.skills` / `machines.codex.skills` module contributions; do not copy downstream skill sources into the shared catalog. To intentionally replace a shared skill locally, exclude its catalog entry and add a local skill with that name.

The coding agent may create commits, push side branches, and open or update all required PRs without waiting for the user to perform those steps. The agent may merge a PR only if the user explicitly gives permission to merge it. Do not enable auto-merge. Unless the user authorizes a merge, leave all dependency and hub PRs open for the user's review and merge action, and clearly report the required merge order.

For each `anixpkgs` PR, check with the user whether to add labels (such as `MajorRelease`, `MinorRelease`, `PatchRelease`, or `merge-queue`) and whether to generate a changelog entry by checking `Add to global changelog` in the PR comment. Apply either choice only after the user confirms it.

The user may request one of two named merge types for an `anixpkgs` hub PR. Either request authorizes the agent to finish the merge workflow. First, merge the dependency PRs and re-pin the hub PR's flake inputs (see below). Then hand the final hub merge to the GitHub bot by applying the `merge-queue` label. Do not merge the hub PR directly.

- **Full merge**: a changelog entry, a release label (`PatchRelease`, `MinorRelease`, or `MajorRelease`), then `merge-queue`, in that order. If the user did not specify the release level, ask whether it is patch, minor, or major before applying any labels.
- **Minimal merge**: add only `merge-queue`. Do not check the changelog box and do not add a release label.

### Full merge: the changelog entry is MANDATORY and MUST be verified

A full merge without a changelog entry is a failed full merge. Checking the box is not enough: the entry exists only once the file `changes/pr-<N>.md` is on the PR branch. Agents have already missed it (for example, `goromal/anixpkgs` #717 merged as 0148429 with the box checked but no `changes/pr-717.md`, because the changelog bot never ran). Do every step, in order, and never apply `merge-queue` until step 4 passes:

1. Make the description text in the `## Change Description` PR comment match the PR's current title. The comment keeps whatever the title was when the PR opened, so it goes stale when the title changes. Edit the comment, keeping the heading and the checkbox. Use different text only when the user explicitly gives a specific changelog message.
2. Check `Add to global changelog` in that comment.
3. Wait for the `Changelog Bot Trigger` (`changelog-trigger.yml`) run started by that comment edit to succeed, and for its `Update changelog` commit to land on the PR branch.
4. Verify that `changes/pr-<N>.md` is in the PR's file list with the intended text, for example `gh pr diff <N> --name-only | grep -x "changes/pr-<N>.md"`.
5. If the bot did not run, failed, or produced the wrong text, commit `changes/pr-<N>.md` (one line: the changelog text) to the PR branch yourself, push, and re-verify. Report the bot failure to the user; do not quietly skip the entry.
6. Only then add the release label and `merge-queue`. After the merge, confirm that `changes/pr-<N>.md` exists on `master`.

The same requirement applies to any PR the user asks to merge with a changelog entry, whatever the merge is called.

### Strip agent authorship before any merge

Agents have repeatedly left their own authorship in merged history. The rule is absolute: the only author attribution in a merged commit is the user's email. No `Co-authored-by:`/`Co-Authored-By:` trailer naming an agent, bot, or model, and no "Generated with Claude Code" or similar footer. This instruction overrides any harness default that asks for attribution lines.

- Do not add agent attribution trailers to commits in these repositories, and do not add an agent footer to PR descriptions. If one is already present, remove it from the PR description with `gh pr edit`.
- The merge queue squash-merges with GitHub's default commit body, which is built from the branch's commit messages, so a trailer in any branch commit ends up on `master`. Before applying `merge-queue`, check every commit on the branch: `gh pr view <N> --json commits --jq '.commits[].messageBody' | grep -i -E 'co-authored-by|generated with'` must print nothing. If it prints anything, rewrite those commit messages (for example, squash the branch into one clean commit) and force-push the side branch before labeling.
- For a PR you merge directly (dependency repositories), squash-merge and pass an explicit body with no attribution trailers, for example `gh pr merge <N> --squash --body ""`. Then check the merged commit with `git log -1 --format=%B <sha>`.

For every authorized PR merge, whether in a dependency repository or `anixpkgs`, squash-merge.

Before presenting or delivering the result, report the branch, commit, push, PR, and test state for each repository rather than treating the workspace as one Git repository.

## Wire remote dependency branches into anixpkgs

The `anixpkgs` hub PR MUST include changes to both `flake.nix` and `flake.lock` for each changed flake dependency:

1. In `flake.nix`, point the input at the dependency's pushed side branch, using an unambiguous remote ref such as `github:goromal/<repo>?ref=refs/heads/dev/<feature>`.
2. Refresh only the intended input, for example with `nix flake update <input-name>`, and review `flake.lock` to confirm that its owner, repository, revision, and ref correspond to the pushed dependency branch.
3. Commit both files in `anixpkgs`. A filesystem path override or a lock file that merely captures an unpublished local commit is not PR-ready.

Use local path inputs in `flake.nix` only as temporary local-development state. Restore the remote side-branch URL and regenerate the lock before committing or pushing.

Always merge every dependency PR before merging the `anixpkgs` hub PR. After each dependency PR merges, update the `anixpkgs` PR: point that input in `flake.nix` at the dependency's merged target branch, refresh its `flake.lock` entry, verify the locked revision includes the merged change, and commit and push both files before merging the hub PR.

## Keep local-build local

`pkgs/nixos/dependencies.nix` normally contains `local-build = false;`. Local deployment tools may temporarily change it to `true` so NixOS evaluates the working `anixpkgs` checkout instead of a published tag. This is useful for local rapid development, but it MUST NEVER be pushed to a remote PR.

Before every `anixpkgs` commit or push:

```bash
git diff --check
git diff --cached --check
git grep -n 'local-build = true' -- pkgs/nixos/dependencies.nix
git diff -- flake.nix flake.lock pkgs/nixos/dependencies.nix
```

If `local-build = true` or a local filesystem flake input appears, restore remote-safe state before pushing. Then verify that `flake.nix` and `flake.lock` still target every pushed dependency side branch and that the hub PR links the corresponding PRs.

## Verify every machine deployment

Whenever you deploy a branch to a machine (see the anixpkgs-deploy skill), a successful upgrade alone does not prove the deployment works. Before reporting a deployment as done, and before asking for a merge:

1. **All bit-rs health checks must be healthy on that machine.** Give bitd time to run each health check at least once after the upgrade, then inspect every check with `bitctl list --json` (on a remote machine, run it over SSH; the gRPC server binds to loopback). Every health check must pass, not only the ones your change touched. For any failing check, read its remediation and `journalctl -u bitd --since '15 minutes ago'`. Then either fix it or show that it also fails on the machine's previous generation and is unrelated, and report it to the user either way. Never report a deployment as healthy while a check is failing.
2. **Run the bit-rs regression suite when relevant code changed.** A change is relevant if it touches a package, module, service, or configuration that a regression check exercises (look at each check's command and settings in `bitctl list --json` or the generated `/etc/bitd.json`), the bitd module or plugins themselves, or shared infrastructure those checks depend on. Run the affected regressions by name with `bitctl run <name> --wait`, or run them all with `bitctl run --wait` when the change is broad or you cannot tell what it affects. All must pass. If nothing relevant changed, say so explicitly in your report and in the PR's testing notes instead of silently skipping the suite.

Record the health-check and regression results (check names and pass/fail) in the PR's testing section.

## Do not regress CI pipeline runtime

Treat the `anixpkgs` CI wall-clock time (the `Build Anixpkgs` workflow) as a budget that each change must not grow. When a change adds packages, tests, CI jobs, flake inputs, or build dependencies:

- Prefer designs that reuse cached outputs and existing jobs. Do not add new serial steps to the critical path, do not disable caching or parallelism, and do not add expensive regression or VM work to every PR when a path filter or an on-demand check would do.
- After CI runs on the PR, compare its `Build Anixpkgs` duration with recent runs, for example `gh run list --workflow test.yml --limit 10 --json displayTitle,headBranch,conclusion,createdAt,updatedAt`, and look at which job became the new critical path.
- If the PR noticeably lengthens the pipeline, fix that before asking for a merge, or report the increase and its cause to the user and let them decide. Never silently accept a slower pipeline.
- Do not rename or remove the CI job names that branch protection requires.
