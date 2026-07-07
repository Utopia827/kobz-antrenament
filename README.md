# Antrenament Kobz — server-backed workout app

Single static `index.html` (Render static site, live at
`https://kobz-antrenament.onrender.com`). Kobz's personal workout log.

Data is **server-backed** via Supabase so it syncs across all his devices and so
**Jarvis (the brain) can pre-fill his workouts**. `localStorage` is kept as an
offline fallback and is migrated up to the server on first load.

## Storage

- **Supabase project:** `xflfbphbklbntrspkwsv` (the IGP project — has a direct PG
  URL for DDL + an anon key we can embed).
- **REST base:** `https://xflfbphbklbntrspkwsv.supabase.co/rest/v1`
- **Client auth:** the page embeds the project **anon** key (public by design;
  a `NEXT_PUBLIC`-style key). The **service_role** key is NEVER in the page.
- **Table:** `public.workout_logs`

```sql
create table public.workout_logs (
  id         bigserial primary key,
  log_date   date        not null,          -- Europe/Bucharest date, 'YYYY-MM-DD'
  day        text,                          -- RO day name: Luni|Marți|Miercuri|Joi|Vineri
  exercise   text        not null,          -- EXACT name from the app PLAN (see below)
  set_index  int         not null,          -- 0-based set number
  weight_kg  numeric,                       -- null = light/bodyweight (cell shows blank)
  reps       int,
  note       text,
  updated_at timestamptz not null default now(),
  unique (log_date, exercise, set_index)
);
-- RLS on, permissive (personal single-user app): anon + authenticated get full access.
```

The app matches a stored row to an input cell by **`exercise` name** (and `day`),
so `exercise` **must exactly equal** the plan's exercise name for that day, and
`day` should be the Romanian day name. The app opens on the current weekday's tab.

### Exercise names per day (must match exactly)

- **Luni / PUSH:** Împins înclinat cu gantere · Împins orizontal cu gantere ·
  Fluturări la cablu (piept) · Împins umeri cu gantere, priză neutră ·
  Ridicări laterale la cablu · Extensii triceps la cablu cu funie
- **Marți / PULL:** Tracțiuni la helcometru (lat pulldown) · Ramat cu bara aplecat ·
  Ramat la cablu așezat · Face pull la cablu (umeri sănătoși) ·
  Flexii biceps cu bara · Flexii biceps înclinat cu gantere
- **Miercuri / LEGS:** Genuflexiuni cu bara · Îndreptări românești (RDL) ·
  Presa de picioare · Flexii picioare așezat (ischio) ·
  Extensii picioare (cvadriceps) · Ridicări pe vârfuri (gambe)
- **Joi / UPPER:** Împins orizontal cu gantere · Ramat cu sprijin pe piept ·
  Landmine press (împins blând cu umărul) · Tracțiuni la helcometru priză neutră ·
  Ridicări laterale la cablu · Flexii ciocan cu gantere
- **Vineri / ARMS:** Împins la piept cu priză îngustă · Flexii biceps cu bara ·
  Extensii triceps deasupra capului la cablu · Flexii biceps înclinat cu gantere ·
  Extensii triceps la cablu cu funie · Flexii biceps la cablu

## How Jarvis fills a workout (INSERT)

Insert one row **per set** (upsert on `log_date, exercise, set_index`). Two ways:

### 1. Direct Postgres (service, from the brain repo — for DDL/bulk)

```python
# authored_tools pattern; IGP_PG_URL is in the brain .env
import os, psycopg2
from psycopg2.extras import execute_values
from dotenv import load_dotenv; load_dotenv()
rows = [  # (log_date, day, exercise, set_index, weight_kg, reps, note)
  ("2026-07-07","Marți","Tracțiuni la helcometru (lat pulldown)",0,40,12,None),
  ("2026-07-07","Marți","Tracțiuni la helcometru (lat pulldown)",1,40,10,None),
]
conn = psycopg2.connect(os.environ["IGP_PG_URL"]); conn.autocommit=True
execute_values(conn.cursor(), """
  insert into public.workout_logs (log_date,day,exercise,set_index,weight_kg,reps,note)
  values %s on conflict (log_date,exercise,set_index)
  do update set weight_kg=excluded.weight_kg, reps=excluded.reps,
                day=excluded.day, note=excluded.note, updated_at=now();
""", rows)
```

### 2. REST (service_role key, upsert)

```bash
curl -s -X POST \
  "https://xflfbphbklbntrspkwsv.supabase.co/rest/v1/workout_logs?on_conflict=log_date,exercise,set_index" \
  -H "apikey: $IGP_SUPABASE_SERVICE_ROLE_KEY" \
  -H "Authorization: Bearer $IGP_SUPABASE_SERVICE_ROLE_KEY" \
  -H "Content-Type: application/json" \
  -H "Prefer: resolution=merge-duplicates,return=minimal" \
  -d '[{"log_date":"2026-07-07","day":"Marți",
        "exercise":"Ramat la cablu așezat","set_index":0,
        "weight_kg":40,"reps":12,"note":"working set"}]'
```

Kobz refreshes the app → the sets appear pre-filled in the right cells. He can edit
and save (the page upserts his edits back with the embedded anon key).

## Deploy

Push `index.html` to `main` → Render static site auto-deploys.
Push with `GH_UNIGIRLIES_TOKEN` (the `GITHUB_PAT` is denied for this repo).
