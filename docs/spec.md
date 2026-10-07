# SourceJump Plugin — Specification & Discussion

This document describes the new, unified SourceJump SourceMod plugin: what each part of it is for, which questions are still open, and what the community has agreed on so far.

It is a living document. It changes as discussions conclude.

## How to take part

Discussion happens on the **SourceJump Discord**, in the [#plugin-topics](https://discord.gg/RYP5G4tuwX) forum. Each section below links to its own topic there. If a topic link doesn't open, join through the invite link first.

- Discuss in the topic for that section.
- **Missing a question?** Ask it in that section's topic. It gets added to the end of the list, so question numbers never change.
- **Missing a whole section, or think something here is wrong?** Post in the [💡 Missing something?](https://discord.com/channels/333865962568941568/1557356307835650139) topic. That includes decisions already marked as agreed.
- 👍 the messages you agree with. Thumbs-up is how we measure support.
- When a topic settles, the outcome is written into **Current consensus** below and added to the [Decision log](#decision-log).
- If it isn't written here, it hasn't been decided.

Each section has a short **direction**, the **open questions**, and the **current consensus**.

**Status legend**

| Status | Meaning |
|---|---|
| **Agreed** | Settled. Reopening it needs a good reason. |
| **In discussion** | Open. Some parts may already be agreed. |
| **Not discussed** | Open, nobody has weighed in yet. |

---

## 1. Goals

**Status:** Agreed

**Direction**
- One plugin replaces the current separate plugins (database, anti-cheat logs, WR display).
- Built from scratch with maintainability and performance in mind. The old plugins are kept for reference only, and their behavior isn't ported as-is.
- Counter-Strike: Source only.
- Anyone can install and run it. Features that write data to SourceJump require an API key issued by SourceJump.
- The plugin is the source of the raw data (players, times, replays, etc.) that the new backend and website are built on.

---

## 2. Access & API keys

**Status:** In discussion · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353440575885312)

**Direction**
Keyless servers get a limited, read-only feature set. Submitting records and anti-cheat reports requires a key. The current setup, with a separate key per plugin, goes away.

**Open questions**
1. One key per server covering every feature, or separate keys/permissions per feature?
2. Exactly which features are available without a key?
3. How are keys requested, issued and revoked?

**Current consensus**
- Records and anti-cheat reports require a key issued by SourceJump.

---

## 3. Timer support

**Status:** Not discussed · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353437761634314)

**Direction**
Every server currently runs shavit's timer. The plugin needs to read times, styles, tracks and zones from the timer.

**Open questions**
1. Is shavit the only supported timer, or should others be supported?
2. If only shavit: which version(s) are supported?

**Current consensus**
_None yet._

---

## 4. Record & time submission

**Status:** In discussion · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353434233970749)

**Direction**
Authorized servers send finished runs to SourceJump. The old plugin only sends world records (normal style, main track), plus every existing record when it first runs.

**Open questions**
1. Send only records, or every completed run? Sending every run would allow tracking individual player progress.
2. Should a server's existing times be sent when it first connects, given they may have been set before the server followed SourceJump's rules?
3. What data does a submission include (time, jumps, strafes, sync, date, tickrate, etc.)?

**Current consensus**
- Submitting requires an API key.
- Submitting is disabled if the server doesn't meet the anti-cheat requirement (see [Anti-cheat](#11-anti-cheat-reporting)).

---

## 5. Styles & tracks

**Status:** In discussion · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353430971060304)

**Direction**
All styles should be supported, not just autobhop/normal on the main track.

**Open questions**
1. How do we make sure a time submitted as one style was actually run on that style? For example, a sideways run must never count as normal.
2. Who defines the list of supported styles and their settings?
3. Are bonus tracks supported?

**Current consensus**
- Support all styles, not only normal.

---

## 6. Server settings enforcement

**Status:** Not discussed · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353427061702792)

**Direction**
Every server that submits times should play on a level playing field. This section is about *how* that is checked. *Which* settings are required is decided in [Gameplay rules](#7-gameplay-rules).

**Open questions**
1. How do we enforce or verify that a server uses the agreed settings, zones and plugins?
2. What happens when a server is out of compliance: submissions blocked, flagged, or something else?

**Current consensus**
_None yet._

---

## 7. Gameplay rules

**Status:** Not discussed · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353422284390491)

**Direction**
Defines the gameplay behaviors, cvars and plugins a server must (or must not) run to submit times. The first step is collecting the list of rules that need deciding. Any rule that needs a longer debate gets its own topic.

**Open questions**
1. Which gameplay behaviors, cvars and plugins need a decision?
2. For each: is it required, forbidden, or left to the server?

**Rules**

Add a row for each rule that comes up, and record its outcome here.

| Rule | Description | Status | Decision |
|---|---|---|---|
| Landfix | _TBD_ | Not discussed | — |
| _…_ | | | |

**Current consensus**
_None yet._

---

## 8. Maps

**Status:** In discussion · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353418178302018)

**Direction**
Times are accepted from any map. This section is about which maps count toward rankings and leaderboards, and how that is decided. There are many low-effort maps that players may not want to grind to rank.

**Open questions**
1. Should every map be ranked, or only some?
2. If only some: what makes a map eligible, and who decides?
3. How are new maps added or reviewed?

**Current consensus**
- Times are accepted from any map.

---

## 9. Map tiers & points

**Status:** Not discussed · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353408518692924)

**Direction**
Defines how maps are tiered and how points are calculated, so that rankings reflect how hard and how relevant a map is.

**Open questions**
1. Should maps be tiered by difficulty? If so, how many tiers, and who assigns them?
2. How are points calculated (map tier, placement, time relative to the WR, style, etc.)?
3. How do rankings handle different styles and tracks?

**Current consensus**
_None yet._

---

## 10. Zones

**Status:** In discussion · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353404915785799)

**Direction**
SourceJump stores zones for every map (start, end, bonuses, and possibly others). Servers that submit times must use these zones, so everybody runs the same zones.

**Open questions**
1. How do zones get submitted, reviewed and accepted into the database?
2. Which zone types are covered?
3. What happens on a map that has no approved zones yet?
4. Can keyless servers download zones too?

**Current consensus**
- SourceJump is the source of zones for submitting servers.

---

## 11. Anti-cheat reporting

**Status:** In discussion · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353398695628830)

**Direction**
The plugin forwards anti-cheat detections to SourceJump. [BASH 2.0](https://github.com/hermansimensen/bash2) is the primary supported anti-cheat.

The old anti-cheat plugin also depends on [REST in Pawn](https://forums.alliedmods.net/showthread.php?t=298024) and [SteamWorks](https://forums.alliedmods.net/showthread.php?t=229556).

**Open questions**
1. Should other anti-cheats be supported besides BASH?
2. What data is sent with a detection?
3. How are detection reports used on the SourceJump side (flagging runs, reviewing players, etc.)?

**Current consensus**
- Requires an API key.
- If the required anti-cheat isn't installed, the server doesn't submit records.

---

## 12. Replays

**Status:** Not discussed · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353393356275852)

**Direction**
The old plugin uploads a replay file with each world record, but SourceJump doesn't currently use them for anything. Some server owners have reported performance drops while a replay is being sent.

**Open questions**
1. Should the plugin keep sending replays? If so, for which runs (WRs only, top times, everything)?
2. What would replays be used for: a web replay viewer, run verification, something else?
3. How can uploading be made lighter on the server (compression, uploading in the background, etc.)?
4. Should servers be able to download WR replays to play back in-game?

**Current consensus**
_None yet._

---

## 13. World record display

**Status:** In discussion · **Discuss:** [Discord topic](https://discord.com/channels/333865962568941568/1557353390185652244)

**Direction**
Show the current SourceJump world records for the map the server is on, similar to the current SJWR plugin ([wrsj.sp](https://github.com/rtldg/wrsj/blob/main/wrsj.sp)).

**Open questions**
1. What's shown in-game, and through which commands or menus?
2. Is this available to keyless servers?

**Current consensus**
- Supports every style, not only normal.

---

## Reference: old plugins

Kept for reference only. They are not a specification.

| Plugin | Source | Purpose |
|---|---|---|
| Database | [sourcejump/plugin-database](https://github.com/sourcejump/plugin-database) | Sends records (and replays) to SourceJump. API key required. |
| Anti-cheat logs | [sourcejump/plugin-ac-logs](https://github.com/sourcejump/plugin-ac-logs) | Forwards BASH detections to SourceJump. Separate API key. |
| SJWR | [rtldg/wrsj](https://github.com/rtldg/wrsj/blob/main/wrsj.sp) | Shows SourceJump WRs for the current map. |

---

## Decision log

Settled decisions, newest first. Entries marked *initial direction* were set by the project lead before community discussion started.

| Date | Section | Decision | Source |
|---|---|---|---|
| 2026-10-07 | Goals | Counter-Strike: Source only. | Initial direction |
| 2026-10-07 | Maps | Times are accepted from any map. | Initial direction |
| 2026-10-07 | Goals | One unified plugin, rewritten from scratch; old plugins are reference only. | Initial direction |
| 2026-10-07 | Access & API keys | Anyone can run the plugin; records and AC reports require a SourceJump-issued key. | Initial direction |
| 2026-10-07 | Anti-cheat | No supported anti-cheat installed → no record submission. BASH 2.0 is primary. | Initial direction |
| 2026-10-07 | Zones | SourceJump stores zones; submitting servers must use them. | Initial direction |
| 2026-10-07 | Styles / WR display | All styles are supported, not only normal. | Initial direction |
