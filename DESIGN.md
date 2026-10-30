# noteql

## what it is

noteql is a search engine over my notes from every app i keep them in.
it pulls notes from obsidian (via github), notion, onedrive, and google
drive into one cloud database, figures out when the same note lives in
more than one app, and lets me search all of it from a search page on
my notes website.

it exists because my notes are fragmented. a class note might start as
a word doc in onedrive, get copied into notion, then get rewritten in
obsidian. searching any one app gives me a partial or outdated view,
and nothing tells me which copy is the current one.

## principles

- **notes stay where they are.** the database holds a searchable copy;
  nothing is moved or edited in the source apps.
- **cross-app matching is the core**, not an add-on.
- **everything is public** (tentatively). they're just notes, so no
  login.

## differentiator

it knows when the same note lives in several apps, groups the copies,
and shows the newest one.

search across apps already exists. duplicate finders already exist,
but each works inside a single app (drive, onedrive, notion, obsidian
each have their own). i haven't found anything that matches copies
*across* apps.

## final product shape

```
github (obsidian)  notion  onedrive  google drive
        \            |        |          /
         sync jobs (only process what changed)
                        ↓
          normalize into relational schema
                        ↓
          match copies of the same note
                        ↓
               cloud postgres
                        ↓
   search page on notes website  +  mcp server (later)
```

1. **search page** on my notes website: type a query, get ranked notes
   with snippets, see every app each note lives in, newest copy first.
2. **mcp server** (later): claude can search my notes directly.

## schema (draft)

```sql
sources (
  id          SERIAL PRIMARY KEY,
  app         TEXT,        -- 'github', 'notion', 'onedrive', 'gdrive'
  account     TEXT,        -- 'personal', 'school'
  folder      TEXT,        -- only sync this folder, not the whole account
  visibility  TEXT DEFAULT 'public'
);

notes (
  id                 SERIAL PRIMARY KEY,
  title              TEXT,
  canonical_copy_id  INTEGER   -- newest copy; the one search shows first
);

copies (
  id            SERIAL PRIMARY KEY,
  note_id       INTEGER REFERENCES notes(id),
  source_id     INTEGER REFERENCES sources(id),
  external_id   TEXT,          -- id/path inside the source app
  path          TEXT,
  title         TEXT,
  body          TEXT,
  content_hash  TEXT,
  modified_at   TIMESTAMP,
  UNIQUE (source_id, external_id)
);

links (from_note INTEGER, to_note INTEGER);
tags  (note_id INTEGER, tag TEXT);
```

key decision: **`notes` vs `copies` are separate tables.** three files
in three apps = 3 rows in `copies` pointing to 1 row in `notes`.

## sync

one command per source, plus `--all`:

```bash
noteql sync github
noteql sync notion
noteql sync onedrive --folder "School/Courses"
noteql sync --all
```

- incremental: skip a copy if its `content_hash` hasn't changed.
- re-syncing the same file updates its row instead of adding a new one
  (`UNIQUE (source_id, external_id)`).
- only sync chosen folders, so random files in drive/onedrive don't
  become publicly searchable.
- can run by hand, on a cron, or as a github action.

## duplicates

two different problems:

| kind | example | how |
|---|---|---|
| same file synced twice | re-running `sync notion` | unique key → upsert |
| same note in different apps | docx → notion page → obsidian file | matching (below) |

matching, roughly:

| case | detection |
|---|---|
| exact copy | same `content_hash` |
| edited copy | similar text + similar title |
| not a copy | neither |

matched copies share one `note_id`. the newest becomes canonical:

```sql
UPDATE notes SET canonical_copy_id = (
  SELECT id FROM copies WHERE note_id = :note_id
  ORDER BY modified_at DESC LIMIT 1
) WHERE id = :note_id;
```

## evaluation

hand-label [50–100] pairs of my own notes as "same note" / "not same"
and measure the matcher:

```
correct:          [X] / [N]
false matches:    [Y]
missed matches:   [Z]
```

this number is what makes the project measurable. without it, "it
matches duplicates" is just a claim.

## build order

each step has to work end-to-end before the next.

- v1: sync github (obsidian notes) into cloud postgres. basic search.
- v2: search page on my notes website.
- v3: second source (notion or one onedrive folder).
- v4: cross-app matching + canonical copy. hand-label pairs, get the
  number.
- v5: ranking that prefers canonical copies and uses metadata (tags,
  links, recency).
- later: more sources, mcp server, semantic search, qa with citations.

## open design questions

- **search: build my own index or use postgres full-text search?**
  own index = more work, stronger database-internals story (overlaps
  with cse 444 material: pages on disk, tombstones, caching, crash
  safety). postgres fts = faster to ship, more time for matching.
- **hosting:** which managed postgres. tbd.
- **matching threshold:** how similar is "same note"? decide after
  looking at real pairs.
- **canonical rule:** newest by `modified_at` is the default. is
  "most edited" or "longest" sometimes better?
- **granularity:** whole notes for now. chunks later if search needs it.

## non-goals

- being a note-taking app (the apps stay the apps)
- moving or deleting anything in the source apps
- multi-user
- real-time sync (batch is fine)
- private notes / login (for now)
