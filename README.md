# Agent skills

Shared agent skills maintained independently from the NixOS and application repositories that install them. Each skill lives in `skills/<name>/` and can contain a `SKILL.md` plus related references, examples, scripts, or assets.

`catalog.nix` lists the shared skills for consumers that use Nix. It contains paths relative to this repository and can restrict an entry to specific frameworks with `frameworks`.

Downstream repositories may use the catalog as a default and add skill directories of their own through their agent configuration. Keep those local additions in the downstream repository.
