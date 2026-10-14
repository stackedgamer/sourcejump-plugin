# SourceJump Plugin — Blueprint

This document describes **how the plugin is built**: its parts, how they fit together, and what the first version (the *foundation*) includes.

*What* the plugin should do is decided in [spec.md](spec.md), together with the community. When the two disagree, the spec wins and this document is updated.

---

## In one paragraph

`sourcejump.smx` is a single SourceMod plugin that replaces today's three SourceJump plugins (database, anti-cheat logs, SJWR). Any server can install it. Without an API key, the server gets SourceJump's zones, records display and rules. With a key, it also sends every finished run, its replay and anti-cheat detections to SourceJump. Before sending anything, the plugin checks that the server plays by SourceJump's rules. If it doesn't, nothing is sent until it's fixed.

---

## Guiding principles

1. **Build what's decided. Don't block on what isn't.** Most open discussions (landfix choice, ranked styles, which cvars) are about *values*, not *code*. Those values live in a **ruleset** that SourceJump serves. When a discussion concludes, SourceJump updates the ruleset, and servers pick it up on the next map change with no plugin update.
2. **Never lag the server.** All network traffic runs in the background. Uploads happen between runs, never during them.
3. **Never lose a run.** Finished runs are saved on the server first, then sent. A network outage or map change just means they're sent later.
4. **Use what already exists.** shavit's timer already records replays and can load zones from a URL. The plugin configures and checks those features instead of rebuilding them.
5. **Trust is verified, not assumed.** The plugin keeps honest servers consistent and makes mistakes visible. It can't make a server owner's machine tamper-proof, and doesn't pretend to. SourceJump verifies runs on its own side, mainly from replays (see [Limits](#what-the-plugin-can-and-cant-guarantee)).

---

## How the pieces fit together

```
                      ┌──────────────────────── SourceJump API ───────────────────────┐
                      │  public: ruleset · zones · records · replays · latest version │
                      │  keyed:  handshake · runs · replay upload · AC detections     │
                      └───────────────▲───────────────────────────────┬───────────────┘
                                      │ background HTTPS              │
 ┌─────────────────── game server ────┼───────────────────────────────┼───────────────┐
 │                                    │                               ▼               │
 │  ┌──────────────── sourcejump.smx ─┴────────────────────────────────────────────┐  │
 │  │  core · api client · ruleset · compliance · runs + local queue · replays     │  │
 │  │  records (!wrsj) · zones · anti-cheat bridge · landfix (built in)            │  │
 │  └───────▲──────────────────────────▲─────────────────────────────▲───────────┘  │
 │          │ forwards & natives       │ detections                   │ checks       │
 │  ┌───────┴──────────────┐   ┌───────┴──────┐   ┌───────────────────┴─────────┐   │
 │  │ shavit's bhoptimer   │   │ BASH 2.0     │   │ cvars · loaded plugins ·    │   │
 │  │ (timer, zones-json,  │   │              │   │ style config                │   │
 │  │  replay recorder)    │   └──────────────┘   └─────────────────────────────┘   │
 │  └──────────────────────┘                                                         │
 └───────────────────────────────────────────────────────────────────────────────────┘
```

### What's built in, what's required, what's checked

Not everything should be crammed into one plugin, and not everything can be. Components fall into four groups:

| Group | Meaning | Members |
|---|---|---|
| **Built in** | Compiled into `sourcejump.smx`. Ships with it, updates with it, and can't be swapped out | All SourceJump modules (below). **Landfix**: always on, no toggle (poll: 87–27 to remove toggling). Which variant is still being polled: Haze vs. no-time-loss |
| **Required alongside** | Separate plugins or extensions that must be loaded, in a version SourceJump accepts. The plugin checks them; it doesn't contain them | shavit's bhoptimer (incl. `shavit-zones-json`, `shavit-replay-recorder`) · BASH 2.0 · REST in Pawn extension |
| **Forbidden** | Plugins that conflict with the rules. Loaded → no submissions | e.g. standalone landfix or landfix-toggle plugins (landfix is built in now) |
| **Free** | Everything else: admin tools, chat, visual aids like paint and showtriggers | Not restricted. Reported to SourceJump so it can audit if needed |

Each module is its own source file, and `sourcejump.sp` just includes them, so each part can be worked on separately while the server only installs one file. A small fix plugin that SourceJump decides to standardize later can be added the same way.

shavit's timer stays separate on purpose: it's large, every server already runs it, and server owners configure it. The plugin *configures and checks* the timer instead of containing it.

---

## Modules

Each module is one file under `scripting/sourcejump/`. Only `timer_shavit.sp` talks to shavit's timer directly; everything else uses the plugin's own data structures. If another timer ever needs supporting, only that file changes.

| Module | Job | Needs key |
|---|---|---|
| `core` | Startup, config file, API key, logging, admin commands, status | — |
| `api` | Background HTTPS client, retries, versioned endpoints | — |
| `ruleset` | Fetch the ruleset, cache it on disk, work from the cache if SJ is unreachable | No |
| `compliance` | Apply and check the ruleset: cvars, plugins, styles, zone setup | No |
| `timer_shavit` | Read finished runs, styles, tracks and zones from shavit | — |
| `styles` | Recognize which server styles are SourceJump's ranked styles | No |
| `runs` | Capture finished runs, keep them in a local queue, send them | **Yes** |
| `replays` | Upload the replay for each sent run, in the background | **Yes** |
| `zones` | Point shavit at SourceJump's zones; fingerprint the zones in use | No |
| `records` | Show SourceJump records in-game (today's SJWR) | No |
| `anticheat` | Forward BASH detections | **Yes** |
| `landfix` | The one standard landfix, always on | No |

### Styles: recognized by their settings, not their number

Servers number their styles differently, which is why the old plugin only ever sent normal. The plugin solves this by **comparing each server style's settings** (read from shavit) **against SourceJump's ranked style definitions** in the ruleset:

- A server style that matches a ranked definition exactly is sent as that SourceJump style, whatever number the server gave it.
- A style that matches nothing is a fun style and is never sent. Servers keep their fun styles (leaning in [spec §5](spec.md#5-styles--tracks)).
- A style that *claims* to be ranked (same name) but has different settings is a compliance problem and gets reported.

This answers the spec's question of how a "sideways" time can never be sent as normal.

### Runs

- Every completed run on a ranked style is captured when shavit reports a finish (leaning: every run, not just records, [spec §4](spec.md#4-record--time-submission)).
- Each run gets a unique ID on the server, so a retried send is never counted twice.
- Runs are written to a local SQLite queue (built into SourceMod) and sent in the background. A keyless server can keep its queue too (the "approved later" idea in [spec §2](spec.md#2-access--api-keys), not decided).
- Practice-mode and paused runs are never sent.

### Replays

shavit already records a replay for every run and tells the plugin where it saved the file. The plugin uploads that file, from the queue, between runs. Agreed so far: replays are sent for every run, and the upload must not lag the server ([spec §10](spec.md#10-replays)). Large files are compressed by SourceJump when they arrive, not on the game server.

Watching SourceJump replays in-game (`!wrosr`-style) comes after the foundation. Replays are then fetched **when someone asks to watch one**, not on every map load: a map can have a WR per style and track, which adds up to a lot of downloading for replays nobody may watch.

### Zones

shavit already has a module (`shavit-zones-json`) that loads a map's zones from a URL when the map starts. Its documentation even lists a SourceJump URL as an example. So:

- The plugin points `shavit-zones-json` at SourceJump's zone endpoint and enforces that setting.
- It turns off other zone sources for the zone types SourceJump controls. Leaning from discussion: start, end, bonus and glitch zones are SourceJump's, while freestyle and personal zones are left to the server.
- It records a fingerprint of the zones actually loaded and sends it with every run, so a zone changed mid-map can't slip through.
- Zone-trigger behaviour that shavit controls with cvars (e.g. `shavit_zones_box_offset`) is enforced through the ruleset.
- On a map without SourceJump zones, submissions are off for that map until it's decided otherwise ([spec §8](spec.md#8-zones), open).

No new zone plugin is needed for the foundation. Whether SourceJump later wants its own zone logic (Eric's idea of SJ controlling exactly when the timer starts) can be decided without changing anything else.

### Anti-cheat

BASH 2.0 stays a separate plugin the server installs itself; `sourcejump.smx` checks that an accepted version is loaded. The plugin is compiled against BASH's include file (a small header that describes BASH's events, not BASH itself) so it can hear detections and forward them to SourceJump, as the old AC plugin did. Detections aren't shown publicly (leaning, [spec §9](spec.md#9-anti-cheat-reporting)). This module stays thin on purpose: the community has openly questioned whether BASH is enough, and the most-supported alternative (replay analysis) needs no extra plugin work, since replays are already sent.

### Landfix

Built in, always on, no toggle command. That follows the poll (87–27). Which landfix is still being polled (Haze vs. no-time-loss). The module is built so the chosen variant drops in; nothing else depends on it.

### Records display

Today's SJWR, built in. Works without a key, covers every style and track. Shows the map's SourceJump records through a command and menu. Records from before 2.0 carry a legacy flag from the API, and the display marks them as such. Exact commands are open ([spec §11](spec.md#11-world-record-display)); the foundation starts with `!wrsj` plus a menu.

---

## The ruleset

A versioned JSON document served publicly by SourceJump. The plugin applies it on every map start.

| Section | Contains |
|---|---|
| `version` | Ruleset version. Sent with every run, so SourceJump knows which rules a run was played under |
| `plugin` | Minimum accepted `sourcejump.smx` version |
| `cvars` | Required values. The plugin sets them, and re-sets them if they change |
| `plugins.required` | Name + accepted versions (shavit, BASH, …) |
| `plugins.forbidden` | Names or file hashes that block submissions |
| `styles` | Ranked styles: SourceJump key, display name, required settings |
| `zones` | Zone source URL, zone types SourceJump controls |
| `features` | Switches, e.g. which styles get replay uploads |

### Starting values (draft)

The first ruleset, with each value's status. Anything not **Agreed** is a proposal, and changing it later is a data change, not a code change.

| Setting | Value | Status |
|---|---|---|
| Landfix toggling | Not allowed | **Agreed** (poll 87–27) |
| Landfix variant | Haze or no-time-loss | Poll running |
| `sv_allow_wait_command` | `0` | Leaning (nobody against) |
| `sv_cheats` | `0` | Proposed |
| Toggling fix plugins mid-run | Not allowed | Leaning |
| `shavit_zones_json_enable` / `_url` | `1` / SourceJump zone endpoint | Follows from "SJ is the zone source" (initial direction) |
| `shavit_zones_box_offset` | `0` | Suggested |
| Replay recorder | shavit's own, same everywhere | Suggested |
| rngfix settings | Fixed values | Suggested, values open |
| Ranked styles | normal, scroll, hsw, sw, a/d-only, w-only, bw (+ lowgrav, autostrafe discussed) | Proposed list, open |
| Tickrate | Reported with every run | Value not discussed yet |

---

## Plugin ↔ SourceJump API (proposed)

The API doesn't exist yet. The plugin is built against this contract and a small **mock server** that ships in the repository, so it can be developed and tested now. The contract then becomes the first specification for the backend.

All endpoints are under `/v1/`, so the API can change later without breaking servers on older plugin versions. Keyed requests send `Authorization: Bearer <key>`.

| Method | Endpoint | Key | Purpose |
|---|---|---|---|
| `GET` | `/v1/ruleset` | No | Current ruleset |
| `GET` | `/v1/maps/{map}/zones` | No | Zones in the format `shavit-zones-json` reads directly |
| `GET` | `/v1/maps/{map}/records` | No | Top times per style and track, each with a legacy (pre-2.0) flag |
| `GET` | `/v1/replays/{run_id}` | No | Download a replay (for in-game playback, later) |
| `GET` | `/v1/plugin/latest` | No | Latest plugin version + file hash |
| `POST` | `/v1/servers/handshake` | Optional | On map start: versions, loaded plugins, compliance state → key status |
| `POST` | `/v1/runs` | Yes | Submit a finished run |
| `PUT` | `/v1/runs/{run_id}/replay` | Yes | Upload its replay |
| `POST` | `/v1/anticheat/detections` | Yes | Forward a BASH detection |

**A run submission includes (draft):** run ID, player SteamID64 and name, map, SourceJump style key, track, time, jumps, strafes, sync, perfs, average/max velocity, finish timestamp, tickrate, plugin version, ruleset version, zone fingerprint, compliance state. What a submission should contain is still open in [spec §4.3](spec.md#4-record--time-submission), so the format is easy to extend.

---

## Updates

- **Foundation:** on every map start, the plugin compares its version with `/v1/plugin/latest` and the ruleset's minimum version. Too old → no submissions, and admins are told.
- **Later:** automatic updates. When a new version exists, the plugin downloads it, checks its hash, and swaps it in on the next map change. Server owners get notice before updates. A server that switches auto-update off still works, but stops submitting once it falls below the minimum version.

---

## What the plugin can and can't guarantee

- **It can** make sure every honest server runs the same settings, zones, styles and landfix. It can stop accidental mistakes from reaching the database, and tell players *before* they start a run that the server isn't eligible.
- **It can't** stop a determined server owner from modifying the plugin itself. A modified build can report whatever it likes, including a correct hash.
- **So** the real safeguards are on SourceJump's side: replays (and the movement data inside them) are checked there, every run carries the ruleset and zone fingerprint it was played under, and a server's key, along with every time sent with it, can be revoked. Hosted servers have also been discussed as a way to close this gap; they're outside the plugin's scope.

---

## Foundation: what gets built first

The foundation is built in phases. Each phase is reviewed before the next one starts.

| Phase | Delivers | Waits on a discussion? |
|---|---|---|
| 0 | Repository layout, automatic compile on every change, mock API server, build instructions | No |
| 1 | Core, config and key, API client, handshake, ruleset fetch with offline cache, version check | No |
| 2 | Records display (`!wrsj`), keyless | No |
| 3 | Timer adapter, style recognition, run capture, local queue, run submission | No (submission fields extendable) |
| 4 | Compliance: apply/check cvars, required and forbidden plugins, styles; player and admin notices; pause submissions | No (values come from the ruleset) |
| 5 | Zones via `shavit-zones-json` + zone fingerprint | Partly (maps without zones) |
| 6 | Replay upload in the background | No |
| 7 | BASH detection forwarding | No |
| 8 | Built-in landfix | Which variant (poll running) |

**After the foundation:** automatic updates, in-game SourceJump replay playback, and rules that need gameplay code (e.g. blocking mid-run lowjumps) once they're decided.

---

## Dependencies

| Dependency | Version |
|---|---|
| SourceMod | 1.12 or newer |
| shavit's bhoptimer | 4.x (latest release is 4.0.1). Exact minimum chosen during the build |
| REST in Pawn (ripext) | Latest. Used for background HTTPS, JSON and file transfer |
| BASH 2.0 | Version pinned by SourceJump |

SourceMod plugins are licensed GPLv3, so the plugin and everything built into it is open source.
