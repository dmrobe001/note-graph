# Articulate

Version 4.0

Articulate is a single-file, local-first web app: a private **journal of
moments**, each one a past-tense sentence anchored at a timestamp — *drove to
work*, *promised John the invoices by Friday*, *closed the garage*. It is
built around one commitment: the moment you open it, you can type, and what
you type is committed with an honest timestamp — every other feature is an
optional layer applied later, or never. Lapses in organization produce
missing metadata, never wrong data.

The design idiom is **case grammar**. An entry is the head of a sentence; its
outgoing typed links are the sentence's clauses; link types are the
prepositions; category entries (Person, Place, Activity, the project map) are
the lexicons each clause selects from. The linking system exists to pin each
moment to locations in the keeper's mental map of their life, making entries
fast to write and easy to surface later — not to build a knowledge base about
objects. Arbitrary specifics live in entry bodies.

The entire application is one HTML file (`articulate.html`) with no build
step and no dependencies. Deploy it by serving the file from any static host;
it is designed to be added to a phone's home screen.

## Data model

Five tables, held in `localStorage` and mirrored to an NDJSON export format.

**Entries** are the single content type. An entry has an immutable `id` and
`created_at`, an editable display timestamp `ts`, and optional `title` and
`body`. Everything — journal jots, lexicon words, map categories, ghosts,
deadlines, interval markers — is an entry.

**Edges** are directed, typed links between entries, and every semantic edge
obeys one invariant: **it points outward from the sentence's subject** to the
thing mentioned. Rendering an entry reads only its outgoing edges; asking
"where is this mentioned" reads incoming ones.

### The structural types (the map)

- `is a` — taxonomy / lexicon membership, non-exclusive ("Driving is a
  Travel", "To-Do is a Status").
- `part of` — composition, non-exclusive ("Church is part of Personal").
- `of` — composition **with handle propagation**, for pieces too small or
  generic to be intelligible alone: "Treasurer of ASR" prints everywhere as
  `ASR/Treasurer`, recursively. Whether a super is displayed is stated in
  the link type, per entry.
- `complements` — the `of` of verbs. A noun declares its verb once, in the
  map ("Invoices complements Accounted", each game `complements → Played`);
  a moment then needs only `doing → Invoices` and the row reads
  *Accounted ASR/Annual Report/Invoices*. `object` remains for genuinely
  per-moment direct objects.
- `is` — state, exclusive *within a shared `is a` root*: setting a Status
  displaces the previous Status but touches no other lexicon's `is` links.
  Exclusivity is a property of the lexicon, not the type.

"A member of X" throughout the app means an entry with a live `is a` /
`part of` / `of` edge directly to X — never other types, never grandmembers.

### The clausal types (the sentence)

`by` (agent; **absent = first person**), `doing` (the verb — an Activity
member, a complement noun, or a token/ghost), `object`, `recipient`,
`beneficiary` ("for …"), `complement`, `with`, `at` (location of *this
moment*; an interval's origin is its begin entry's `at`, its destination the
end entry's), `until` (begin entry → end entry; produces the `12:00–14:30`
lead and is never printed as a word), `due` (→ a bare future moment whose
`ts` is the deadline), and `fulfills` (a fulfilling moment → the promise it
honors). New link types can be created freely.

### Handles and the first person

An entry's **handle** (its name in chips and clauses) is its title, else its
first body line, else its time. An `of` link prepends the target's handle
(`ASR/Treasurer`); a bare entry (no title, no body) with a `doing` link is
named as a piece of its deed (`Bins/8:45`). The seeded **I** entry is
positional: absent `by` means I did it and is never displayed; in oblique
slots it reads "me" ("John Promised me"); the keeper never writes about
themselves in the third person.

## Ghosts, promises, fulfillment

A **ghost** is a moment that hasn't happened yet: an ordinary entry marked
`is → To-Do`, its clauses describing the deed, its timestamp provisional —
the status link keeps a future-dated entry honest. A **promise** is a moment
that did happen: `doing → Promised` (or `Planned`, to oneself),
`recipient → person`, `object → ghost`, `due → deadline`. Work sessions are
intervals with `doing → ghost`. **Fulfillment** re-stamps the ghost to the
completion moment, removes `is → To-Do`, and adds `fulfills` — from the
concluding session's end entry, or from the deed itself — to every wrapper.
Cancelled / Missed / Deferred give ghosts honest endings other than
fulfillment. The due target stays a separate entry, so the deadline survives
the re-stamp and "was it on time?" remains answerable. Future-dated due
moments surface at their place in the time-sorted list: the top of explore is
quietly an agenda.

## Intervals

A begin entry with `doing → verb-or-ghost` and `is → Active`. Active ≡
carries that status link; there is no containment root, and a begin entry
never marked Active (an inauguration, a years-long role) clocks nowhere —
legitimately unended. The end entry may be pre-created (`until`-linked,
provisionally timestamped) to hold a planned destination without concluding
anything. "End now" re-stamps the end entry to the present — minting it if
absent — and removes `is → Active`.

## Tabs

- **＋ (create)** — a draft form: timestamp, title, body, pending links.
  Nothing exists until ＋ commits; drafts persist per tab in a device-local
  key so a killed page doesn't eat keystrokes.
- **❗ (inbox)** — the Noted dump: a field and ＋ create `{title, is → Noted}`
  in one gesture, asking nothing else. Below, the Noted backlog; tapping a
  row opens the task sheet to formalize it. Noted entries burn red in the
  runbar until triaged.
- **🕓 (active)** — active intervals as running clocks wearing their verb
  phrase, with "end now". The verb cloud below offers all and only: `is a`
  members of Activity, everything reaching one through a `complements`
  chain, every ghost, and every Noted entry — an open task or a standing
  complement is one tap from a running clock. Typing filters; the ancestor
  match walks `complements` / `of` / `is a` upward, so "asr" keeps
  `ASR/…/Invoices`. ＋ mints a titled Activity member from the field text
  and starts it at once. Tapping a row opens the **interval sheet**: With
  (Person cloud, any number), From / To (Place cloud, exactly-one by
  gesture — `at` on the begin and end entries respectively, offered for
  every interval; selecting To with no end yet pre-creates it), and a
  "task done" button when the interval's deed is a ghost.
- **☑ (to-do)** — the ghosts: all and only `is → To-Do` entries, due time
  on the right (overdue in red). The top field + ＋ mints a ghost and opens
  the **task sheet** — one surface serving triage and to-dos alike: inline
  title, Status chips (scoped-exclusive `is`), Doing (the verb cloud),
  Object and Parent (a drillable map cloud over Personal / Professional,
  ▸ to open a category, ⬑ to climb back, ＋ to mint-and-select; Parent
  carries a `part of` / `of` mode toggle and replaces existing composition
  links by gesture), Due (a date-time picker minting or moving the deadline
  moment; × removes only the link), Promised to (a Person cloud — selecting
  someone mints the wrapping promise, finding or minting the Promised verb
  by title; "plan" does the same with Planned and no recipient; nothing is
  created silently), and ✓ Done (the fulfillment gesture).
- **🧭 (explore)** — view field, search, and a flat time-sorted list (the
  default) or the entry tree (containment = the view's types; the seeded
  All view uses the structural three). In the flat list **every entry is
  one continuous line**: a time lead (the begin–end range for a concluded
  interval, whose end entry's own row is suppressed; a still-active
  interval never fakes a range), then the sentence read off its clauses in
  a fixed order — subject (`by`, unless first-person) · the title as
  subject noun when there is no verb · verb phrase · recipients · objects
  · complements · `is X` / `is a X` / `part of X` / `of X` /
  `complements X` · `with …` · `at …` (or `from X to Y` on interval rows)
  · `for …` · `due <t>` · `fulfills …` — then `: title / body`
  (`/ end title / end body` on interval rows). `:` and `/` print only
  between two non-empty pieces. So the seeded and hand-built records read
  back as the journal they encode:

      12:03 Cancelled is a Status
      1:09  Treasurer of ASR
      1:10  Annual Report part of ASR/Treasurer
      1:13  Accounted ASR/Annual Report/Invoices is To-Do
      1:14  Promised John ASR/Annual Report/Invoices/1:13 due fri 3:00pm
      5:00  Served as ASR/Treasurer
      5:01  John Served as ASR/Secretary
      1:00–2:30 Drafting with Bob at office for confined flow micelle
      9:00–9:40 Accounted ASR/Annual Report/Invoices from home to school
                fulfills Promised/1:14

  Tapping routes by kind: interval rows and active begins → the interval
  sheet; ghosts and Noted → the task sheet; everything else → the entry
  editor. Every sheet keeps the raw editor one tap inside.
- **⋯ (menu)** — synthetic, always last: Drive sync and settings,
  diagnostics, snapshots/restore, NDJSON export/import, and the two resets
  (wipe local, new realm). The views/links/tabs managers live here.

The entry picker used everywhere combines a search field with a collapsible,
view-filtered tree; while search text is present, every row shows a ＋ that
creates the typed text under that row (linked by the view's first
containment type). Each picker field remembers its own last-used view.

## Seed configuration

4.0 is a **reboot**: it carries no migrations from 3.x stores and assumes a
fresh dataset. On first run it creates: the sixteen link types above (all
locked — the tabs, sheets, and display grammar are hard-coded around them);
entries `Status`, `Activity`, `Person`, `Place`, `Personal`,
`Professional`, `I`, and the statuses `Active`, `To-Do`, `Noted`,
`Deferred`, `Cancelled`, `Missed` (each `is a → Status`); the locked `All`
view (containment `is a` / `part of` / `of`); and the five tabs. Nothing
else is seeded — Travel, Projects, Promised, verbs, people, and places are
the keeper's vocabulary, built with the structural types (the app finds
Promised/Planned by title, minting them on first use). Seed records carry
fixed ids identical on every device, so independently seeded stores merge as
no-ops; anchors and types materialize at boot on any store that predates
them. Everything seeded is ordinary data — retitle at will.

## Sync

Sync is organized around **realms**. A realm is one shared dataset, named by
a short random string; its state lives in a single Drive file,
`capturelog-sync-<realm>.ndjson`. A well-known pointer file,
`capturelog-realm.json`, records which realm the Drive account is currently
on as `{realm, generation}`, where `generation` is a monotonic counter
bumped each time a realm is minted.

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
  restorable snapshot (if it holds anything beyond the seed), wipes,
  re-seeds, and joins the pointer's realm.
- Pointer generation **below** the device's: the pointer is stale; the
  device rewrites it.
- Equal generations but different realms (two devices minted concurrently):
  the lexicographically greater realm string wins — the same verdict on
  every device, so all converge.
- No pointer at all: the device publishes its own realm, minting one if it
  has none.
- A device that has data but no realm yet (used offline before its first
  sync) joins the pointer's realm *without* a wipe; the ordinary merge
  unions its records in.

This makes global reset a first-class, single-tap operation. **⋯ tab → New
realm (reset all)** snapshots the device's data, wipes and re-seeds it,
mints a fresh realm at `generation+1`, and rewrites the pointer; every other
device follows suit on its own next sync, each saving its own snapshot
first. Restoring a snapshot works the same way: a new realm is minted whose
contents are exactly the snapshot, and the pointer bump carries the restored
state to every device. **Wipe local data** is the device-only variant: it
clears and re-seeds one device, which then pulls the realm's data straight
back on its next sync. Note that a 4.0 device joining a realm still carrying
3.x data will union the two vocabularies rather than translate between them;
start 4.0 on a fresh realm.

Devices or app builds that never read the pointer keep writing to whatever
filename they know; no realm member ever reads those files, so they are
harmless. Files of abandoned realms accumulate in Drive as inert clutter and
can be deleted there by hand at leisure.

IDs are globally unique strings — `{deviceId}-{letter}{n}` for user-created
records (the device id is per-install and never synced) and fixed `seed-…`
strings for seeded records — so records created offline on different devices
cannot collide.

### Drive setup

1. In the Google Cloud console, create a project and enable the
   **Google Drive API**.
2. Configure the OAuth consent screen: External, **Testing** status, and add
   your own Google account as a test user. Testing status avoids Google's
   app review; its cost is that grants expire after roughly seven days of
   disuse, so an occasional re-consent tap is expected behavior.
3. Create credentials → OAuth client ID → type **Web application**, and add
   the exact origin this file is served from (e.g. `https://you.github.io`)
   under **Authorized JavaScript origins**. No redirect URI is needed.
4. Paste the client ID into `DEFAULT_CLIENT_ID` near the top of
   `SEC:APP/SYNC` and commit. The client ID is a public identifier —
   security comes from the origin allowlist and the consent screen, not
   from secrecy — so committing it to a public repository is standard
   practice. The in-app sheet (⋯ tab → Drive settings) can override it per
   device.
5. On each device: ⋯ tab → Sync now → sign in once. Tokens are cached for
   about an hour; syncs run at load, every five minutes, and shortly after
   any edit, all silently while a token is live. Sign-in is only ever
   requested in response to a tap.

The synced files — the realm blob `capturelog-sync-<realm>.ndjson`, the
pointer `capturelog-realm.json`, and any `capturelog-snapshot-….ndjson` —
are visible in Drive as ordinary JSON/NDJSON, downloadable and importable by
hand, independent of the app.

## Data portability

⋯ tab → **Export NDJSON** downloads the full store (entries, edges, link
types, views, tabs, and a settings line). **Import & merge** unions a file
into the current store by the same rules as sync. **Import & replace** swaps
the store for the file's contents.

Wire-level identifiers are stable across releases and intentionally not
renamed with the app: the localStorage keys `capturelog.v2`,
`capturelog.device`, `capturelog.sync`, and `capturelog.draft`, the Drive
filenames `capturelog-realm.json`, `capturelog-sync-<realm>.ndjson`, and
`capturelog-snapshot-….ndjson`, and the NDJSON record tags. Data outlives
naming.

## Deferred by decision

**Repeats** (a standing plan with ghost occurrences; cancel-one vs
cancel-series is status-on-token vs status-on-plan) and **amounts** ("ran
5 km" lives in bodies until querying over quantities is a real want) are
deliberate bolt-ons awaiting experience with the core workflows. Verb tense
is the keeper's craft: titles are written to parse in the templates, and the
renderer never conjugates anything but I/me.

## Code map

The file is organized by grep-able section markers. Every section is
bracketed by `SEC:NAME (BEGIN)` and `SEC:NAME (END)` comments, nested
hierarchically (`SEC:APP/UI/TAB_ACTIVE`), with a full manifest in
`SEC:HEADER` at the top of the file. `grep -n "SEC:" articulate.html` prints
the skeleton.

## Versioning

Articulate uses a plain incremented version (this is 4.0), recorded here and
in the `SEC:HEADER` manifest. Storage identifiers do not change with the
version.
