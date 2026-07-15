# Articulate

Version 3.6

Articulate is a single-file, local-first web app for capturing atomic, timestamped
notes and organizing them after the fact. It is built around one commitment: the
moment you open it, you can type, and what you type is committed with an honest
timestamp — every other feature (linking, tasks, time intervals, views) is an
optional layer applied later, or never. Lapses in organization produce missing
metadata, never wrong data.

The entire application is one HTML file (`articulate.html`) with no build step
and no dependencies. Deploy it by serving the file from any static host; it is
designed to be added to a phone's home screen.

## Data model

Five tables, held in `localStorage` and mirrored to an NDJSON export format.

**Entries** are the single content type. An entry has an immutable `id` and
`created_at`, an editable display timestamp `ts`, and optional `title` and
`body`. Everything — journal jots, category headers, tasks, interval markers —
is an entry. There is no folder/file distinction: an entry acts as a "place"
in the hierarchy simply by being titled and linked to.

**Edges** are directed, typed links between entries (`from_id`, `to_id`,
`type_id`). The seeded **link types** are `child` (hierarchy; the edge
points child → parent), `ends` (the edge points an interval's end entry →
its start entry), and the relation vocabulary: `doing` (an interval's
start entry → its activity), `with` (an entry → a person present), `at`
(an entry → the place that moment happened at), `status` (an entry → the
status entry currently describing it; an interval is *active* while its
start entry carries one to `Active`), and `completed` (reserved, with the
rest of `status`'s wider role, for the coming rework of the ☑ tab).
Location hangs on
moments, so an interval's origin is its start entry's `at` link and its
destination its end entry's — there are no separate from/to types. New
link types can be created freely.

Throughout the app, "a child of X" means exactly one thing: an entry with a
live `child`-type edge pointing directly at X — never other link types,
never grandchildren. The intended norm is that link types, not depth, put
an entry in several categories at once: an entry keeps one organizing
hierarchy for finding it, while flattened memberships (an activity filed
under `Travel` that is *also* a direct child of `Activities`) decide which
menus and clauses it appears in. Descendant traversals are reserved for
view conditions, where they are explicit and chosen.

**Views** are saved queries: a name, a list of condition entries (a matching
entry must be a descendant of *all* of them — pure conjunction), and the set of
link types that count as containment for that view's traversal. A view with no
conditions matches everything; the seeded "All" view is the universal fallback.

**Tabs** are stored configuration, not code. Each tab row has a class and a
small config, and the tab bar — pinned to the top of the screen — renders
whatever rows exist. Tabs can be renamed, reordered, duplicated, and
configured from inside the app. One tab is synthetic rather than stored: the
⋯ tab, appended after the data tabs on every device, which carries sync,
export/import, snapshots, and the reset actions.

All cross-references are by ID, so retitling anything is always safe. Deletes
are soft (tombstones). Every record carries `updated_at` for merging.

## Tab classes

- **create** (seeded ＋) — a draft form: timestamp, Title field, focused
  body, Links field. Nothing exists in the store until the ＋ button in the
  top row commits the draft; committing creates the entry (with the draft's
  timestamp and its pending links) and clears the form back to the tab's
  defaults. Links are added through the entry picker; each chip shows its
  link type, tappable to change. Selecting the tab — arriving from elsewhere
  or re-tapping it — resets an untouched draft's timestamp to now, while a
  draft with content stays exactly as it was left. Drafts persist per tab in
  a device-local localStorage key (`capturelog.draft`, never synced), so a
  killed page doesn't eat keystrokes. Duplicated create tabs can
  carry default titles, bodies, and links, making them stencils for repeated
  entry shapes.
- **active** (seeded 🕓; class id `interval` on the wire) — there is no
  containment root: an interval is **active** while its start entry
  carries a `status` link to the `Active` status entry. Every active
  interval is a row with a running clock and an "end now" button.
  Starting one creates the start entry as a child of the activity, with a
  `doing` edge to it and the activating `status` edge. Tapping a row opens
  the **interval sheet** rather than the raw entry editor: a "With" field
  whose chip cloud is the children of `People` (any number of `with` edges
  from the start entry), and — when the interval's activity is a direct
  child of `Travel` — "From" and "To" fields whose chip clouds are the
  children of `Places`. Both are `at` edges, exactly-one enforced by the
  gesture: From sits on the start entry, To on the end entry. An active
  interval **may already own its end entry**: selecting a To place with
  none present pre-creates it (ends-linked, provisionally timestamped) to
  carry the planned destination, and the interval stays active. "End now"
  re-stamps that end entry to the present — minting it if none exists —
  and deletes the `status` link. Each cloud ends in a ＋ chip that
  mints a new child of the anchor and selects it at once; the underlying
  start and end entries stay one tap away at the bottom of the sheet.
  Because activeness is just an edge, intervals may overlap freely, and a
  clock started on one device can be ended on another.
- **explore** (seeded 🧭) — a view field, search, and a flat list sorted
  by timestamp (the default), or the entry tree via the sort toggle.
  In the flat list **every entry is one
  continuous line led by its time**. A concluded interval reports as one
  row: an entry with exactly one incoming `ends` link is a *beginning*,
  its end entry is dropped from the list, and the pair renders together
  (an *active* interval is never a beginning, even when it already owns a
  pre-created end entry — its provisional timestamp would fake a range) as

      8:30–8:47 Driving from home to school with Alice, Bob / cold / late

  — start–end range, then the `doing` target's title (when there is
  exactly one `doing` link *and* its target is a direct child of
  `Activities`), "from" the start entry's single `at` place and
  "to" the end entry's, a "with" clause gathering every `with` link, then
  the beginning entry's title, body, the end entry's title, and body.
  Every other entry renders as: time, its parent's name when it has
  exactly one parent, its title, a "with" clause, an "at"
  clause, its body, then `Children:` — the titles of its children plus a
  count of the untitled ones — and, when it has several parents,
  `Child of:` in the same style. Parent and children here are direct
  `child` edges, regardless of the view's containment types. Throughout, the `:` and `/` separators
  print only between two non-empty pieces. Tapping a combined row opens
  the interval sheet; tapping any other entry opens the editor.
- **exclusive** (seeded ☑) — a list view on top; tapping an entry opens a
  swap panel against a second view (seeded: "Statuses"). Choosing a member
  removes the entry's links to every member of that view and adds the chosen
  one — exactly-one membership, enforced by the gesture and self-healing.
- **now** (❗) — urgency capture. Nothing is created merely by opening the
  tab: ＋ creates a titled child of the `Now` anchor from the field text, and
  tapping an activity chip marks that activity itself urgent by linking it
  under `Now`. Children of `Now` appear as red chips in the runbar.

The engine's own configuration — view editing, link-type management, and tab
management — lives in the ⋯ tab (Views…, Link types…, Tabs…), not in the tab
bar. Tab rows of the retired editor classes lingering in older stores are
hidden everywhere and inert.

The now and active tabs and the interval sheet are hard-coded around
**anchor entries** with fixed seed ids — `Now` and `Active` (under
`Status`), `Activities`, `Travel` (under `Activities`), `People`, and
`Places` — rather than per-tab configuration. Anchors are materialized at boot if a store
predates them: because the ids are fixed, every device mints the identical
record and the merge unions the copies. Both tabs share the activity-capture
control: a focused field, ＋, and a chip cloud — every descendant of
`Activities` on the now tab; on the active tab the direct children of
`Activities`, `People`, `Active`, and `Now`, leaning on the flattening
convention above (an activity must be a direct child of `Activities` to be
startable, however deep it also sits under categories like `Travel`).
Chips are filtered so one survives when it or any
ancestor matches the typed text. On the active tab, tapping a chip starts
an interval on that activity — rows and runbar
bubbles wear the activity's title; typing a new name and ＋ mints a
parentless activity entry and starts an interval on it. Tab-bar buttons
are icons only.
- **⋯ (menu)** — synthetic, always last: Drive sync and settings,
  diagnostics, snapshots/restore, NDJSON export/import, and the two resets
  (wipe local, new realm).

The entry picker used everywhere combines a search field with a collapsible,
view-filtered tree. While search text is present, every row shows a ＋ that
creates the typed text as a child of that row. Each picker field remembers its
own last-used view.

## Seed configuration

On first run with an empty store, Articulate creates: link types `child`,
`ends`, `doing`, `with`, `at`, and — reserved for the coming rework of
status and completion tracking — `status` and `completed`; entries
`Status` (with children `Active`, `Someday`, `Dead`), `Activities` (with
child `Travel`), `People`, and `Places`; views `All`, `Active`, and `Statuses`
(`All` counts both `child` and `ends` links as containment); and the tabs
above. Seed records carry fixed ids (`seed-e-status`, `seed-v-all`, …) that
are identical on every device, so independently seeded stores contain
literally the same records and merging them is a no-op — the seed can never
duplicate. Stores seeded before a record existed materialize it lazily at
boot, exactly as anchors do. (3.2 briefly seeded `from`/`to` types; a boot
migration converts their edges to `at` — a from-edge retypes in place, a
to-edge moves to the interval's end entry — and retires the pair. `Done`
was seeded through 3.4; on upgrade it is tombstoned when nothing touches
it, and left in place — as ordinary, hand-deletable data — wherever
entries are filed under it. The legacy v1 importer still mints an
ordinary Done under Status when old done-tasks need a home. `Intervals`
was the interval containment root through 3.5; on upgrade every child
edge into it becomes a `status` edge to `Active` — running intervals stay
running — and the entry itself retires the way Done does, tombstoned only
when nothing else touches it.)
Everything seeded is ordinary data — retitle or
reconfigure at will. The `All` view and all seven seeded types are locked
against deletion: pickers and merges fall back to `child`/`ends`, and the
interval sheet and explore's interval rows are hard-coded around the rest.

## Sync

Sync is organized around **realms**. A realm is one shared dataset, named by a
short random string; its state lives in a single Drive file,
`capturelog-sync-<realm>.ndjson`. A well-known pointer file,
`capturelog-realm.json`, records which realm the Drive account is currently
on as `{realm, generation}`, where `generation` is a monotonic counter bumped
each time a realm is minted.

Within a realm, sync is a convergent merge over the realm's NDJSON blob.
Every record's `updated_at` is a last-write-wins clock: merging is union by
ID per table, newer record wins whole, tombstones ride the same clock. The
merge is idempotent and order-insensitive, so devices may overwrite each
other's uploads blindly — the next pull-merge-push restores the union.
Because seed ids are fixed, a freshly seeded device merges into a realm
without duplicating anything.

Every sync cycle begins by reading the pointer, then pulls, merges, and
pushes the realm file. The pointer comparison gives each device its marching
orders:

- Pointer generation **above** the device's: the dataset was reset from
  another device. The device preserves its current store to Drive as a
  restorable snapshot (if it holds anything beyond the seed), wipes, re-seeds,
  and joins the pointer's realm.
- Pointer generation **below** the device's: the pointer is stale; the device
  rewrites it.
- Equal generations but different realms (two devices minted concurrently):
  the lexicographically greater realm string wins — the same verdict on every
  device, so all converge.
- No pointer at all: the device publishes its own realm, minting one if it
  has none.
- A device that has data but no realm yet (used offline before its first
  sync) joins the pointer's realm *without* a wipe; the ordinary merge unions
  its records in.

This makes global reset a first-class, single-tap operation. **⋯ tab → New
realm (reset all)** snapshots the device's data, wipes and re-seeds it, mints
a fresh realm at `generation+1`, and rewrites the pointer; every other device
follows suit on its own next sync, each saving its own snapshot first.
Restoring a snapshot works the same way: a new realm is minted whose contents
are exactly the snapshot, and the pointer bump carries the restored state to
every device. **Wipe local data** is the device-only variant: it clears and
re-seeds one device, which then pulls the realm's data straight back on its
next sync.

Devices or app builds that never read the pointer keep writing to whatever
filename they know; no realm member ever reads those files, so they are
harmless. Files of abandoned realms and of pointer-unaware builds accumulate
in Drive as inert clutter and can be deleted there by hand at leisure.

IDs are globally unique strings — `{deviceId}-{letter}{n}` for user-created
records (the device id is per-install and never synced) and fixed `seed-…`
strings for seeded records — so records created offline on different devices
cannot collide.

### Drive setup

1. In the Google Cloud console, create a project and enable the
   **Google Drive API**.
2. Configure the OAuth consent screen: External, **Testing** status, and add
   your own Google account as a test user. Testing status avoids Google's app
   review; its cost is that grants expire after roughly seven days of disuse,
   so an occasional re-consent tap is expected behavior.
3. Create credentials → OAuth client ID → type **Web application**, and add
   the exact origin this file is served from (e.g. `https://you.github.io`)
   under **Authorized JavaScript origins**. No redirect URI is needed.
4. Paste the client ID into `DEFAULT_CLIENT_ID` near the top of
   `SEC:APP/SYNC` and commit. The client ID is a public identifier — security
   comes from the origin allowlist and the consent screen, not from secrecy —
   so committing it to a public repository is standard practice. The in-app
   sheet (⋯ tab → Drive settings) can override it per device.
5. On each device: ⋯ tab → Sync now → sign in once. Tokens are cached for
   about an hour; syncs run at load, every five minutes, and shortly after
   any edit, all silently while a token is live. Sign-in is only ever
   requested in response to a tap.

The synced files — the realm blob `capturelog-sync-<realm>.ndjson`, the
pointer `capturelog-realm.json`, and any `capturelog-snapshot-….ndjson` — are
visible in Drive as ordinary JSON/NDJSON, downloadable and importable by
hand, independent of the app.

## Data portability

⋯ tab → **Export NDJSON** downloads the full store (entries, edges, link
types, views, tabs, and a settings line). **Import & merge** unions a file
into the current store by the same rules as sync. **Import & replace** swaps
the store for the file's contents; it also accepts files in legacy formats
(numeric ids, or the `capturelog.v1` localStorage layout, which is also
imported automatically at boot if found) and migrates them on the way in.

Wire-level identifiers are stable across releases and intentionally not
renamed with the app: the localStorage keys `capturelog.v2`,
`capturelog.device`, `capturelog.sync`, and `capturelog.draft`, the Drive filenames
`capturelog-realm.json`, `capturelog-sync-<realm>.ndjson`, and
`capturelog-snapshot-….ndjson`, and the NDJSON record tags. Data outlives
naming.

## Code map

The file is organized by grep-able section markers. Every section is bracketed
by `SEC:NAME (BEGIN)` and `SEC:NAME (END)` comments, nested hierarchically
(`SEC:APP/UI/TAB_CREATE`), with a full manifest in `SEC:HEADER` at the top of
the file. `grep -n "SEC:" articulate.html` prints the skeleton;
`grep -n "SEC:APP/MERGE" articulate.html` finds both ends of a section.

## Versioning

Articulate uses a plain incremented version (this is 3.6), recorded here and
in the `SEC:HEADER` manifest. Storage identifiers do not change with the
version.
