# Build prompt: plugin foundation

This is the prompt used to start building the plugin with an AI coding agent (e.g. Claude Code) working in this repository. It's published so anyone can see exactly what the agent is asked to do, and suggest changes before the work starts.

It deliberately stays short. The details live in [blueprint.md](blueprint.md) (how it's built) and [spec.md](spec.md) (what the community decided). The prompt points the agent at those instead of repeating them, so the documents stay the single source of truth.

---

```text
You are building the foundation of the new SourceJump plugin: a single SourceMod
plugin (sourcejump.smx) for Counter-Strike: Source bhop servers. It replaces
SourceJump's three current plugins (records database, anti-cheat logs, SJWR).
This repository is where it lives.

## Read first

1. docs/blueprint.md  How the plugin is built. This is your design. Follow it.
2. docs/spec.md       What the community has decided. "Agreed" items are fixed.
                      Anything open is not yours to decide.

If the two disagree, the spec wins. Stop and ask.

## What you are building

The foundation phases listed in blueprint.md ("Foundation: what gets built
first"), in order, starting with phase 0. Work on one phase at a time. Finish
each phase with a pull request, then stop and wait for review before starting
the next.

## Ground rules

- Don't settle open questions. If you need a value that the spec hasn't
  decided (a cvar value, a style list, a landfix variant), read it from the
  ruleset, or put a clearly marked placeholder in the mock ruleset. Never
  hardcode it.
- Don't guess APIs. Before you use any shavit/bhoptimer, BASH or REST in Pawn
  forward or native, look it up in that project's source at the version we
  target, and note the version in a comment. If something you need doesn't
  exist, say so.
- Never block the game server. All HTTP goes through REST in Pawn's async
  callbacks. No file or network work during a player's run where avoidable.
  Uploads happen between runs.
- Never lose a run. Runs go into the local SQLite queue before any network
  call. Only a confirmed response from the API removes them.
- Keyless servers must work: ruleset, zones and records display need no key.
- Only timer_shavit.sp may call shavit directly.
- The old plugins (old URLs, old endpoints) are reference only. Build against
  the /v1 API in blueprint.md, served by the mock server.

## Code conventions

- SourcePawn with `#pragma semicolon 1` and `#pragma newdecls required`;
  methodmaps and enum structs over parallel arrays.
- sourcejump.sp is the entry point and includes one file per module from
  scripting/sourcejump/. Shared types and the public API for other plugins go in
  scripting/include/sourcejump.inc.
- Compiles with zero warnings on SourceMod 1.12. Third-party includes are
  vendored at pinned versions so the build is reproducible.
- Prefix: cvars sj_*, commands sm_sj*, chat tag [SJ]. Player-facing text goes
  in a translations file.
- Comment *why*, not *what*. Each module starts with a short header saying what
  it owns.

## Phase 0 specifics

- Repository layout:
    addons/sourcemod/scripting/sourcejump.sp
    addons/sourcemod/scripting/sourcejump/*.sp
    addons/sourcemod/scripting/include/   (sourcejump.inc + vendored includes)
    addons/sourcemod/configs/sourcejump/  (config template)
    mock-api/                             (mock SourceJump API)
    .github/workflows/                    (compile on every push and PR)
- A GitHub Actions workflow that compiles the plugin and publishes the .smx as
  a build artifact.
- mock-api/: a small dependency-free server implementing every /v1 endpoint in
  blueprint.md. It serves a sample ruleset, zones and records from JSON files
  and logs every request, so submissions and uploads can be inspected.
- docs/development.md: how to build, run the mock API and point a local test
  server at it.

## How to verify each phase

- The plugin compiles in CI with no warnings.
- Every network feature is exercised against the mock API, and the PR shows the
  requests it produced.
- You probably can't run a CS:S server. For anything that needs one, add a
  short step-by-step test checklist to docs/testing.md (what to do in-game,
  what should happen) so a server owner can confirm it.

## Each pull request should include

- What was built, in plain language for non-programmers, then the technical
  detail.
- Anything you couldn't verify, and why.
- Questions that need a decision, marked as questions, not answered by you.
```
