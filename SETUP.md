# Shared scores setup (Supabase)

The app works **right now** with no setup — it saves to the browser's localStorage,
exactly as before. Everything below is only needed to make scores **shared**, so
anyone opening the link sees the same live standings.

Takes about 5 minutes.

## 1. Create the project

1. Go to <https://supabase.com> and sign up (free tier is plenty).
2. **New project** → name it anything (e.g. `kobras`) → pick a region near the UK
   (`West EU (London)` or `West EU (Ireland)`) → set a database password
   (you won't need it again) → **Create**.
3. Wait ~2 minutes for it to provision.

## 2. Create the table

Left sidebar → **SQL Editor** → **New query** → paste this and hit **Run**:

```sql
create table tournament_state (
  id         text primary key,
  data       jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table tournament_state enable row level security;

-- Anyone with the link can read the scores.
create policy "public read"
  on tournament_state for select
  to anon using (true);

-- Anyone can write. The passcode in index.html is a convenience gate that
-- prevents accidental edits; it is NOT enforced here and cannot be, because
-- the browser has no secret to prove. See "Security" below.
create policy "public write"
  on tournament_state for insert
  to anon with check (true);

create policy "public update"
  on tournament_state for update
  to anon using (true) with check (true);

-- Seed the single row this tournament uses.
insert into tournament_state (id, data) values ('kobras-july26', '{}'::jsonb);
```

## 3. Get your keys

Left sidebar → **Project Settings** → **API**. Copy:

- **Project URL** — looks like `https://abcdefgh.supabase.co`
- **anon / public** key — a long `eyJ...` string

## 4. Paste them into index.html

Near the top of the `<script>` block, find the config section and fill in the
first two lines:

```js
const SUPABASE_URL      = "https://abcdefgh.supabase.co";
const SUPABASE_ANON_KEY = "eyJhbGciOi...";
const SCORER_PASSCODE   = "kobras21";      // change this to whatever you like
const TOURNAMENT_ID     = "kobras-july26"; // must match the seeded row above
```

Save, refresh, and the footer should change from
"Saved on this device" to **"Shared · everyone sees this"**.

## 5. Test it properly before the night

Open the page on your laptop **and** your phone. Unlock scoring on one, enter a
score, and within ~10 seconds it should appear on the other.

---

## How it behaves

- **Viewing is open.** Anyone with the link sees live standings, no passcode.
- **Scoring needs the passcode.** Tap **Scorer login** at the bottom. The unlock
  is remembered on that device, so a scorer enters it once for the night.
- **Wifi drops are survivable.** Every entry is written to localStorage first and
  the backend second. If the connection dies mid-event, scoring keeps working and
  re-syncs automatically when it returns. The footer shows
  "Saved locally · reconnecting to share…" while it catches up.
- **Two scorers won't clobber each other.** Every entry carries a timestamp and
  the newest edit per match wins, so two people scoring different courts on
  different phones merge cleanly.
- **Refreshing never loses data** — this was true before and still is.

## Security — read this

The passcode is checked **in the browser**, and the Supabase anon key ships in the
page source of a public repo. That means:

- Anyone who views source can read the passcode.
- Anyone who finds the key can write to the table directly.

For a club tournament that's a reasonable trade — it stops a spectator's mis-tap,
which is the actual risk. It is **not** protection against someone who wants to
mess with it. If you ever want that, the fix is a Cloudflare Worker sitting in
front of the database holding the write secret server-side; the page would call
the Worker instead of Supabase directly.

## Backing out

Blank out `SUPABASE_URL` and the app reverts to local-only mode. Nothing else
needs changing.
