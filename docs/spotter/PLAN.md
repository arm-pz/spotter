# Spotter: rebuild plan

Written 2026-11-15. Status: **plan only, nothing below is built yet.** This file is meant to be
picked up cold, by a person or by a new Claude session, without the conversation it came from.

## 1. What we are making

Spotter is a gym tracker for ordinary gym members, built on the open-source
[openGym](https://github.com/DuarteSantos8/openGym). Three things set it apart from upstream:

1. **Its own look.** Ivory Porcelain, Jade Lacquer and Scarlet Signal, with rearranged controls.
2. **Three plans: Lite, Pro, Plus.** Lite hides the lifter's jargon (RIR, PR, 1RM and so on).
3. **An admin screen** where the owner decides which features each plan shows.

The first audience is gym owners, who will be shown **Lite only**, as an app in development.

## 2. Where things stand

| Thing | State |
|---|---|
| `arm-pz/spotter` `main` | openGym **v1.4.1** plus a rebrand that broke the build. Do not build on it. |
| Draft PR #6 (`ccr-5fa25fd0-az9j7q`) | Repairs `main` and adds a graphite and brass theme. **Superseded by this plan.** Keep it only as reference; close it once the new branch exists. |
| Upstream openGym | **v1.4.5** (2026-11-14), four releases ahead. Much larger than v1.4.1. |
| Mockups | Six boards at <https://claude.ai/artifact/RVNFvNPgC9M2kM4iRg49CG> (private to the owner). Sources are copied in `docs/spotter/mockups/`. |
| Approved so far | The palette, the "Features by plan" admin mockup, starting again from v1.4.5. |

## 3. What v1.4.5 already gives us

Reading v1.4.5 changed the plan. Three things we were going to build already exist upstream.

**Modules (since v1.4.4).** Settings → Modules switches whole feature areas on and off per
profile. The single gate is `isOn(S, id)` in `frontend/src/lib/modules.js`; views never read
`S.modules` directly. Switching a module off hides it everywhere and deletes nothing. There are
13 modules and three presets ("Just training", "Training and body", "Everything"). Upstream's
guide is `docs/MODULES.md`. **Our plans are built on this gate, not beside it.**

**A build-level off switch.** `setUnavailableModules(ids)` in the same file makes a module not
exist for a build; `lib/build-caps.js` uses it for the offline app. This is the hook a plan uses
to lock a module.

**Gyms, gym leads and trainers.** With `TRAINERS=on` the server has gyms, join codes, trainer
dashboards and member consent (`docs/TRAINERS.md`, `api/trainers/`). This is what a gym owner
would actually use, so a plan should be assignable to a gym, not only to a person.

**Focus view.** The workout screen already has a one-exercise-at-a-time layout
(`frontend/src/views/FocusView.jsx`) next to Cards and List. The Lite workout mockup is a
restyle and simplification of Focus, not a new screen.

## 4. Feature map

### 4.1 Always on (upstream calls this the core)

Logging sets, weight and reps; plans and routines; the weekly schedule; history; the exercise
library; basic stats (workout count, activity heatmap, exercise progress, muscles); rest timer;
imports and CSV export; backup.

### 4.2 Existing modules and the plan each lands in

| Module id | What it covers | Lite | Pro | Plus |
|---|---|---|---|---|
| `body` | Weigh-ins, weight goal, measurements, progress photos | on | on | on |
| `checkin` | Membership codes on Home, NFC stickers | on | on | on |
| `motivation` | Streaks, challenges, badges, recaps | on | on | on |
| `effort` | RIR / RPE column, effort in Stats | off | on | on |
| `plates` | Plate line under each set, automatic warm-ups | off | on | on |
| `strength` | Strength levels (Beginner to Elite) | off | on | on |
| `health` | Apple Health, Health Connect, heart rate | off | on | on |
| `watch` | Apple Watch, Wear OS, Garmin, Zepp | off | off | on |
| `videos` | Filming a set, form videos | off | off | on |
| `coach` | AI Coach | off | off | on |
| `integrations` | Webhooks, MCP server | off | off | on |
| `community` | Shared exercises, plans and sounds | off | off | on |
| `seasonal` | Halloween, Christmas themes | off | off | on |

`motivation` in Lite is a recommendation, not yet confirmed: streaks are what casual members
respond to. The approved mockup did not list it.

### 4.3 Jargon that is still core upstream, so needs new modules

These cannot be hidden today. Each becomes a new module, added the way `docs/MODULES.md`
"For developers" describes (entry in `MODULES`, words in `modules-copy.js` and every locale,
search keywords, then `isOn` gates).

| New module id | What it hides when off | Lite | Pro | Plus |
|---|---|---|---|---|
| `records` | PR badges, record trophies and cheers, record lists, "best" tags | off | on | on |
| `advsets` | Set types (warm-up, drop, rest-pause, cluster, failure), supersets, tempo, bands | off | on | on |
| `progression` | Progression rules and their guidance line; Lite keeps "same as last time" | off | on | on |
| `analysis` | Estimated 1RM, muscle balance, fatigue and recovery, structural balance | off | off | on |
| `planplus` | Rotation schedule, deload weeks, programmes, routine folders | off | off | on |

Rule for all five: **off means hidden, never changed behaviour of saved data.** A Lite member
who inherits a routine with a drop set still logs it; they just cannot create one. Progression
is the delicate one, because it decides the next weight: it needs unit tests beside the code
(`CONTRIBUTING.md` requires this for training logic).

### 4.4 Server features outside the module system

Trainers and gyms (`TRAINERS=on`), invites, OIDC sign-in, API tokens, the admin dashboard.
These stay server settings. Only "which plan is this gym on" is new.

## 5. How plans work

### 5.1 Data

In the server's `db.json`, next to users and invites:

```
plans: {
  lite: { name: 'Lite', modules: ['body', 'checkin', 'motivation'] },
  pro:  { name: 'Pro',  modules: [ ...lite, 'effort', 'plates', 'strength', 'health',
                                   'records', 'advsets', 'progression' ] },
  plus: { name: 'Plus', modules: [ every module ] },
}
defaultPlan: 'lite'
users[uid].plan        optional, overrides the default
gyms[gid].plan         optional, applies to the gym's members (org.json, when TRAINERS=on)
```

Order of precedence for a member: their own plan, else their gym's, else `defaultPlan`.

### 5.2 Enforcement

- The server sends the member's allowed module list with the session.
- The app passes the modules **not** on that list to the existing `setUnavailableModules`.
  From there `isOn` returns false and every existing gate does the hiding. No per-view work for
  the 13 existing modules.
- Settings → Modules lists only what the plan allows. The member can still switch an allowed
  module off for themselves.
- The three server-side readers of modules (`api/wrapped.js`, `api/coach/core/due.js`,
  `api/integrations/index.js`) also check the plan, so an off feature sends no notifications.

### 5.3 Admin screen

A new page in the existing admin dashboard (`frontend/src/views/Admin.jsx`), as in the
"Admin, features by plan" mockup: one row per module, one switch per plan, core rows shown
locked on, a selector for the default plan. Plus a plan picker on each user row and, with
`TRAINERS=on`, on each gym. New routes: `GET` and `POST /api/admin/plans`,
`POST /api/admin/user/plan`, `POST /api/admin/gyms/plan`. Every change goes to the audit log.

### 5.4 Standalone app (no server)

A build variable, `VITE_PLAN=lite|pro|plus`, read in `lib/build-caps.js` beside the offline
switch, fixes the plan at build time. Same gate, no second mechanism. The demo for gym owners
is `VITE_DEMO=1 VITE_PLAN=lite`.

### 5.5 What a plan is not

It is a display setting, not a paywall. The code is AGPL and self-hostable: anyone running
their own server can turn everything on. Do not describe plans to gym owners as access control.

## 6. Look

| Role | Share | Hex | Used for |
|---|---|---|---|
| Ivory Porcelain | 60% | `#F4EFE4` (cards `#FBF8F1`) | Page ground, cards, tab bar |
| Jade Lacquer | 30% | `#0E4F40` | Text, icons, today's-workout slab, rest timer, done states, charts |
| Scarlet Signal | 10% | `#FF3A24` | Start button, the set to tick next, today's dot, record flags |

- Only the Scarlet hex was supplied. **Ivory and Jade are estimates; replace when the real
  values are known.**
- Text on Scarlet is near-black jade `#062019` (about 4.7:1). White on Scarlet is about 3.4:1
  and must not be used for small text.
- Dark theme swaps the first two: jade ground `#082E26`, ivory slab.
- Type: Archivo, condensed bold for titles and numbers. Bundle the font files in the app (it
  loads nothing from a CDN) and add its OFL notice to `NOTICE.md`.
- Tokens live at the top of `frontend/src/index.css`; accent presets in `lib/format.js` and
  `lib/accent.js`. Add a `scarlet` preset and make it the default for new profiles only.
- Lite workout = Focus view, simplified: one set on screen, large plus and minus for weight and
  reps, one "Set done" button at thumb height, plain words ("times", not "reps").

## 7. Rebrand, as a script

One script, `scripts/spotter-rebrand.mjs`, so it can be re-run after every upstream release:

1. Identity files: `frontend/index.html`, `public/manifest.json`, `capacitor.config.json`,
   Android `strings.xml`, iOS `Info.plist` and project file.
2. App id `com.caffeinity.spotter`: move the Java package (main and test) and fix the tests
   that read those paths.
3. Display strings: `openGym` → `Spotter` in `frontend/src` and all locale files (25 in
   v1.4.5), skipping URLs. Fix "an Spotter". Update the pt-BR fingerprint test.
4. Links: update check and "Source code" → `arm-pz/spotter`.

**Never rename:** localStorage keys (`gym_*`), `opengym-media`, the `opengym_plan` wire value,
crypto salts, the native keyboard event name, and the Gym visual attribution line in Settings
(the licence requires it word for word, and it names openGym).

## 8. Steps

Each step ends with something that can be checked. Do them in order.

| # | Step | Done when |
|---|---|---|
| 0 | In `arm-pz/spotter`, create branch `spotter` from upstream tag `v1.4.5`. Leave `main` and PR #6 untouched. | Branch exists; `npm test` passes in `frontend`, `api`, `mcp` with no changes. |
| 1 | Write and run the rebrand script (section 7). | Build passes; all three test suites pass; locale checks pass; app says Spotter. |
| 2 | Theme tokens, font, scarlet accent (section 6). No layout changes yet. | Screenshots of Home, Workout, Stats, Plan, Settings in light and dark match the palette. |
| 3 | Plans on the client: `VITE_PLAN`, Lite / Pro / Plus presets over the 13 existing modules. | Lite demo build shows no effort column, plates, strength levels, coach, watch. |
| 4 | The five new modules (4.3), one at a time: `records`, `advsets`, `progression`, `analysis`, `planplus`. | Each has tests; with it off, a search of the running app finds no trace of the feature. |
| 5 | Lite screens: Focus-based workout, trimmed Home and Stats. | Matches the "Workout, Lite" and Home mockups; a full workout can be logged and read back. |
| 6 | **Gym-owner demo checkpoint.** Deploy the Lite demo build. | Owner has walked through it on a phone. |
| 7 | Plans on the server and the admin screen (5.1 to 5.3). | Changing a switch in admin hides the feature for a signed-in member on next load. |
| 8 | Plan per gym (needs `TRAINERS=on`). | A member who joins a gym by code gets that gym's plan. |
| 9 | Icons and splash in the new palette; README, docs and website renamed. | No "openGym" left outside licence, attribution and credit lines. |
| 10 | Make `spotter` the default branch on GitHub; rename old `main` to `legacy-1.4.1`; close PR #6. | Owner's decision, done by the owner or on their explicit say-so. |

Steps 0 to 6 are what the gym-owner demo needs. Steps 7 to 10 can follow it.

### Commands to resume from step 0

```bash
git remote add upstream https://github.com/DuarteSantos8/openGym
git fetch upstream tag v1.4.5
git switch -c spotter v1.4.5
(cd frontend && npm ci && npm test && npm run build)
(cd api && npm ci && npm test)
(cd mcp && npm ci && npm test)
# demo: cd frontend && VITE_DEMO=1 npx vite
```

## 9. Risks and things the owner must decide

| Item | Why it matters | Recommendation |
|---|---|---|
| **Exercise media licence** | `NOTICE.md`: the animations are licensed to openGym; a non-commercial fork may keep the 180 px media, "a commercial fork, or any other use, needs its own licence" from gymvisual.com. Selling to gyms is commercial. | Contact Gym visual before any paid pilot, or plan for different media. |
| **AGPL** | Members who use a hosted Spotter must be offered its source, modifications included. | Keep `arm-pz/spotter` public and the "Source code" link working. |
| **CLA workflow** | The inherited `cla` check asks contributors to sign openGym's agreement and fails every PR here. | Remove `.github/workflows/cla.yml` on the `spotter` branch. |
| **API tests and the calendar** | On v1.4.1, 11 `api` tests fail on any date after about 12 Oct 2026 (a 12-week window against a fixed fixture date). | Check at step 0 whether v1.4.5 fixed this; if not, pin the clock in those tests. |
| **Keeping up with upstream** | Upstream ships about every two weeks and is large (v1.4.5 Workout view alone is 2,900 lines). | Keep Spotter changes in few places: the rebrand script, tokens, `modules.js`, the plans code. Avoid rewriting upstream views. |
| **Plan order** | The owner said Lite, Pro, Plus, with Plus on top. Most products put Pro on top. | Confirm before the names reach gym owners. |
| **Who sets a plan** | Assumed: the admin, per gym or per member, with a default. | Confirm. |
| **Default theme** | The palette is light-first; openGym defaults to dark. | Light by default, dark available. Confirm. |
| **Accent picker** | Members can recolour the app today. | Keep it, scarlet as the default. Confirm. |

## 10. Not in this plan

Payments or subscriptions, app-store submission, a new backend, changes to how sync works, and
any feature upstream does not already have other than plans and the admin page for them.
