# The Unstandup: team settings (next build)

**Goal:** any team can run The Unstandup by setting its own team name and places in Host notes. No separate build, and no changes to the portrait's look.

This document is the spec for that update. It lists what to build, every spot in today's code that has to change, the guardrails that protect the layout, and how to check the result. Function and constant names refer to `index.html` as described in `README.md`.

---

## 1. What the host gets

A new card at the top of **Host notes**, above "0 · host controls", called **Your team**. It has:

- **Team name**: one text field, 28 characters at most. Placeholder: `Team VeriFidity`.
- **Places**: a list of 2 to 7 rows. Each row has three parts:
  - **Chip label**: what shows on the time chip in the header, e.g. `Topsham · New Hampshire · Georgia · Ontario`. 48 characters at most.
  - **Time zone**: picked from a dropdown, never typed (see §4).
  - **Short name**: what the portrait's time tile and the invite use, e.g. `Topsham`. 16 characters at most.
  - Each row can be removed with ×. *Add a place* adds a row.
- **Home place**: a choice of which row the meeting start time is entered in. It replaces the fixed "start, Topsham time" label.
- **Reset to Team VeriFidity**: a two-tap button, like *Clear all answers*, that restores the defaults.

Changes save as you type and redraw the header chips immediately, the same way the date and time fields work today.

## 2. Data

One new storage key holds the team settings:

```js
// key: 'unstandup-team'
{
  name: "Team VeriFidity",
  home: 2,                       // index into places
  places: [
    { label: "Seattle", tz: "America/Los_Angeles", short: "Seattle" },
    { label: "Texas", tz: "America/Chicago", short: "Texas" },
    { label: "Topsham · New Hampshire · Georgia · Ontario", tz: "America/New_York", short: "Topsham" },
    { label: "Halifax", tz: "America/Halifax", short: "Halifax" },
    { label: "India", tz: "Asia/Kolkata", short: "India" }
  ]
}
```

- **Defaults:** if the key is missing or unreadable, use the object above. Wrap every read in try/catch, as `store.get` already does.
- **Validation on load:**
  - Drop any place whose `tz` isn't in the dropdown list.
  - Trim every string and cut it to its limit.
  - If fewer than 2 places remain, fall back to the defaults.
  - If `home` is out of range, set it to the first place.

### Rename the storage keys (same build)

GitHub Pages shares browser storage across every repo under `<username>.github.io`, so the current `cg-` and `cg2-` keys can collide with other apps. Move everything to an `unstandup-` prefix:

| Old | New |
|---|---|
| `cg-when` | `unstandup-when` |
| `cg2-sel` | `unstandup-sel` |
| `cg2-ans` | `unstandup-ans` |
| `cg2-ignore` | `unstandup-ignore` |
| `cg2-people` | `unstandup-people` |
| `cg2-tab` | `unstandup-tab` |
| `cg2-qi` | `unstandup-qi` |
| (new) | `unstandup-team` |

On first load, read the old key when the new one is missing, write it under the new name, and leave the old one alone. That way an existing setup survives the move.

## 3. Every spot that changes

Each spot that's written for Team VeriFidity today, and what it becomes.

| Where | Today | Becomes |
|---|---|---|
| `const ZONES` / `const HOME` | Fixed arrays | Built from `team.places` and `team.home`. Keep the same `[label, tz, short]` shape, so `drawZones()`, `portraitHTML()` and `gapNote()` keep working. |
| Header label (HTML, above the `<h1>`) | `Team VeriFidity · twenty minutes, zero story points` | `${team.name} · twenty minutes, zero story points`. Give the span an id and fill it from the script. |
| `renderPlay()` intro, "why" line | "across five time zones" | Count the distinct `tz` values: "across five time zones" or "across 3 time zones". Spell out one to nine. If there's only one zone, say "across one office, more or less". |
| `portraitHTML()` headline | `This is <b>Team VeriFidity</b>.` | `This is <b>${team name}</b>.` (escaped) |
| `portraitHTML()` meta line | `Across ${ZONES.length} time zones` | Count distinct `tz` values, not rows. Two rows can share a zone. |
| `portraitHTML()` time tile | Earliest → latest place | No change in logic. It already uses `short`. |
| `snapPortrait()` image alt text | `Team VeriFidity portrait` | `${team name} portrait` |
| `setupHTML()` time label | "start, Topsham time" | `start, ${home place short name} time` |
| `renderHost()` invite, first line | `Team VeriFidity, you're invited to The Unstandup.` | `${team name}, you're invited to The Unstandup.` |
| `renderHost()` invite, time line | Hard-wired ET, India, Halifax, Texas and Seattle | The home time first, then every other place's `short` name and local time, in list order, joined with commas. |
| `renderHost()` closing message | "thank you, India, for staying late with us" | Thank the latest place, by `short` name, only if its local start time is 5 PM or later. If none qualifies, end at "Thanks for playing." |
| `renderHost()` closing message | `This is Team VeriFidity.` | `This is ${team name}.` |
| `gapNote()` | Always compares the home zone to India, with India's shift and commute advice | **If a place uses `Asia/Kolkata`:** keep today's note exactly. **Otherwise:** name the latest place and its gap, and add "That's after 6 PM for them. Worth moving earlier." when its local time is 6 PM or later. |
| `const EXAMPLE` | Team VeriFidity flavor: biryani, Guntur, chai | Keep it as is. It's always marked *Example*, and it shows what a finished portrait looks like. Optionally, swap `home.items` to the team's own `short` names, so the example lists familiar places. |

Nothing else in `renderPlay()`, `renderControls()` or the question bank refers to the team, so those stay as they are.

## 4. Guardrails that protect the layout

The portrait looks sharp because its type sizes and grid are fixed. Every guardrail below keeps new text inside what that layout was designed for.

- **Team name: 28 characters.** At the portrait headline's size (`clamp(34px, 6vw, 60px)`), that's one line on a laptop and at most two at phone width. Use `maxlength` on the input and also cut it on load.
- **Chip label: 48 characters.** Chips already wrap. 48 matches today's longest label ("Topsham · New Hampshire · Georgia · Ontario" is 43).
- **Short name: 16 characters.** The time tile reads `one call, from ${early} to ${late}` at 16px, and two 16-character names still fit a tile at its 190px minimum width.
- **2 to 7 places.** Fewer than 2 makes the time tile meaningless. More than 7 chips wrap to a third row in the header.
- **Time zones come from a dropdown, never free text.** A typo would make `Intl.DateTimeFormat` throw and break every time on the page. Suggested list: grouped by region, showing a friendly city with its UTC offset on the meeting date.
  - **Americas:** Los_Angeles, Denver, Phoenix, Chicago, New_York, Halifax, St_Johns, Sao_Paulo, Mexico_City
  - **Europe and Africa:** London, Dublin, Lisbon, Paris, Berlin, Warsaw, Athens, Johannesburg, Lagos, Cairo
  - **Asia and the Pacific:** Dubai, Karachi, Kolkata, Kathmandu, Dhaka, Singapore, Manila, Shanghai, Tokyo, Seoul, Sydney, Auckland
- **Escape everything.** Every user-entered string passes through `esc()` before it goes into HTML. That includes the invite `<pre>` and the image alt text.
- **No new colors or fonts.** The card reuses `.msg`, `.when`, `.btn` and the input styles already in the sheet.

## 5. Build order

1. Add the `unstandup-team` key, its defaults and validation, and the old-key migration.
2. Build `ZONES` and `HOME` from settings. Every existing time display should still work with the defaults. This is the regression check.
3. Replace each fixed string in the §3 table.
4. Add the **Your team** card to `renderHost()` and wire it the way `wireSetup()` wires the date and time fields. Redraw the header chips and the gap note on change.
5. Rewrite `gapNote()` for the non-India case.

## 6. Checks before pushing

**With the defaults (Team VeriFidity), nothing should look different:**
- The header chips, gap note, invite, closing message and portrait match today, apart from the closing message's new rule. With India at 7:30 PM, it still thanks India.

**With a test team: "Night Shift Crew" with London, New York and Sydney, home New York:**
- [ ] The header chips show three places. The intro and portrait both say "three time zones" and "3 time zones".
- [ ] The gap note names Sydney and warns that it's late there.
- [ ] The invite lists New York first, then London and Sydney, with correct local times.
- [ ] The closing message thanks Sydney.
- [ ] The portrait headline reads "This is Night Shift Crew." and the time tile reads from New York to Sydney.

**Guardrails:**
- [ ] A 28-character team name stays at two lines or fewer on the portrait at 400px wide.
- [ ] Seven places wrap to at most two rows of chips at 1280px.
- [ ] The time zone can't be typed in, only picked from the list.
- [ ] Typing `<b>hi</b>` as the team name shows the literal text everywhere, never bold.
- [ ] Reset restores Team VeriFidity exactly.

**Storage:**
- [ ] A browser with old `cg2-` data keeps its questions and answers after the update.
- [ ] *Clear all answers* clears answers but not team settings.

## 7. Out of scope for this build

- Editing the question bank from the page. Questions still live in `const BANK`.
- Sharing settings between computers. Each host's settings stay in their own browser, which is the point: one team's changes never touch another's.
- Renaming the game itself. "The Unstandup" stays fixed.
