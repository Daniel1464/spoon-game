# 🥄 Spoon Game — Cary Academy

A secure web app for managing and playing the Cary Academy Senior Spoon Game, built for the Class of 2026. Supports player login, target tracking, elimination codes, admin management, a faculty fork view, and a public game board.

---

## How it works

Each senior is assigned a secret target and given a unique 6-character elimination code. To eliminate a target, a player must catch them outside a safe zone, touch them with their spoon, and say *"You've been spooned, friend."* The eliminated player gives their elimination code to the eliminator, who enters it into the app. The eliminator then inherits the eliminated player's target, and the chain continues until one player remains.

---

## Views

### Public board (pre-login)
- Visible to anyone before logging in
- SGGB title, tagline, and live game statistics
- Running eliminations log (who spooned whom, most recent first)
- Full official rules of play

### Player
- Log in with a unique 6-character access code
- See current target, game status, and % of players remaining
- Display personal elimination code to share if eliminated
- Enter a target's elimination code to record an elimination
- Eliminated players see a spectator view with live stats

### Admin
- Password-protected dashboard
- Player access toggle — enable or disable student logins
- Add, remove, restore, and manually eliminate players
- View full player list with targets, emails, elimination counts, and codes
- Manual target assignment via dropdown per player
- Reshuffle all targets randomly
- Set game status: Single target / Open season / Forks active
- Import players via CSV (requires Name and Email columns)
- Export players to CSV for mail merge
- Print player access cards with QR codes
- Manage faculty forks

### Fork (Faculty)
- Separate password-protected view
- Shows all players and their active/eliminated status
- Live count and percentage of players remaining

---

## Architecture & Security

### Security model
- The **anon key** is visible in the browser but restricted by Row Level Security — it can only read `id`, `name`, `active`, and `kills` from a public view. No codes, emails, or targets are accessible.
- All **admin writes** (eliminate, remove, reshuffle, reset codes, etc.) go through a Supabase **Edge Function** that verifies a SHA-256 password hash server-side before executing anything.
- All **admin reads** (full player list with sensitive data) also go through the Edge Function using the service role key, which bypasses RLS.
- **Passwords** are never stored in the HTML. The browser hashes the entered password using the built-in `SubtleCrypto` API and compares it against the hash stored in Supabase.
- Direct writes to the `players` table from the anon key are completely blocked.

### Changing passwords
Run this in the Supabase SQL Editor:
```sql
-- Admin password
update passwords set hash = encode(digest('yournewpassword', 'sha256'), 'hex') where key = 'admin';

-- Fork password
update passwords set hash = encode(digest('yournewpassword', 'sha256'), 'hex') where key = 'fork';
```

Then update the Edge Function secret to match the new admin password:
```powershell
supabase secrets set ADMIN_PASSWORD=yournewpassword
supabase functions deploy admin-action --no-verify-jwt
```

**Important:** Both must be updated together. If only one is changed, admin login will work but all admin actions will fail.

---

## Setup

### 1. Supabase

Create a free project at [supabase.com](https://supabase.com), selecting **East US (North Virginia)** as the region. Enable both **Data API** and **automatic RLS** during setup.

In the **SQL Editor**, run the following blocks in order:

**Tables:**
```sql
create table players (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  email text default '',
  code text unique not null,
  target_id uuid references players(id),
  kills integer default 0,
  active boolean default true,
  created_at timestamptz default now()
);

create table forks (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  created_at timestamptz default now()
);

create table game_state (
  id integer primary key default 1,
  status text default 'single',
  access_enabled boolean default false,
  constraint single_row check (id = 1)
);

create table eliminations (
  id uuid primary key default gen_random_uuid(),
  victim_name text not null,
  killer_name text not null,
  created_at timestamptz default now()
);

create table passwords (
  key text primary key,
  hash text not null
);

insert into game_state (id, status, access_enabled) values (1, 'single', false);
```

**Email uniqueness:**
```sql
update players set email = null where email = '';
alter table players alter column email drop not null;
alter table players add constraint players_email_unique unique (email);
```

**Row Level Security:**
```sql
alter table players enable row level security;
alter table forks enable row level security;
alter table game_state enable row level security;
alter table eliminations enable row level security;
alter table passwords enable row level security;

-- Revoke direct table access from anon
revoke select on players from anon;
revoke select on players from public;

-- Create restricted public view (no sensitive columns)
create view public_players as
  select id, name, active, kills from players;

create view player_self as
  select id, name, code, target_id, kills, active from players;

grant select on public_players to anon;
grant select on player_self to anon;
grant select on forks to anon;
grant select on game_state to anon;
grant select on eliminations to anon;
grant select on passwords to anon;
```

**Passwords** (replace values before running):
```sql
insert into passwords (key, hash) values
  ('admin', encode(digest('your_admin_password', 'sha256'), 'hex')),
  ('fork',  encode(digest('your_fork_password',  'sha256'), 'hex'));
```

---

### 2. Edge Function

The Edge Function handles all admin writes and reads securely using the service role key.

**Install Supabase CLI (Windows via Scoop):**
```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
Invoke-RestMethod -Uri https://get.scoop.sh | Invoke-Expression
scoop bucket add supabase https://github.com/supabase/scoop-bucket.git
scoop install supabase
```

**Login and link project:**
```powershell
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

**Set secrets:**
```powershell
supabase secrets set ADMIN_PASSWORD=your_admin_password
```

**Deploy:**
```powershell
supabase functions deploy admin-action --no-verify-jwt
```

**To redeploy after any changes to `index.ts`:**
```powershell
supabase functions deploy admin-action --no-verify-jwt
```

---

### 3. Deploy to GitHub Pages

1. Create a new **private** repository on GitHub
2. Upload `index.html` and `README.md` to the repository root
3. Go to **Settings → Pages**, set source to **main** branch, **/ (root)** folder
4. Your app will be live at `https://yourusername.github.io/your-repo-name`

To update the app, upload a new `index.html` — GitHub Pages redeploys automatically within about a minute.

---

## Game day checklist

- [ ] Import all players via CSV (Name + Email columns required)
- [ ] Reshuffle targets in admin dashboard
- [ ] Export CSV and send mail merge with access codes
- [ ] Print player access cards if distributing physically
- [ ] At noon on April 16th: flip **Enable access** in admin dashboard
- [ ] Monitor eliminations log on public board throughout the day

---

## Mail merge

The exported CSV has these columns: `Name`, `Email`, `Access Code`, `Status`, `Eliminations`.

Word mail merge fields: `«Name»`, `«Email»`, `«Access Code»`

---

## Clearing the eliminations log

Run in Supabase SQL Editor:
```sql
truncate table eliminations;
```

To also reset kill counts:
```sql
update players set kills = 0;
```

---

## Official Rules of Play

1. Above all else: this is a game for the well-mannered and polite. No tomfoolery, hoodwinkery, or — worst of all — shenanigans.
2. The game starts at **noon on Wednesday, April 16th, 2026**. Target assignments distributed that morning.
3. Eliminate your target by touching them with your spoon outside a safe zone and saying **"You've been spooned, friend."**
4. No elimination if your target's spoon is literally in hand — not attached to the wrist, in hand.
5. Safe zones: classrooms during class, bathrooms, safety drills, sports practices/games, and any moving motorized vehicle.
6. The real world is **not** a safe zone. Bring your spoon everywhere. *Do not trust anyone.*
7. No chasing, no running. This is a game of cunning and surprise — stride with confidence and purpose.
8. The more elaborate the spooning, the better. *#designthinking*
9. Upon elimination, give your elimination code to your eliminator.
10. The SGGB reserves the right to introduce complications. Watch out for forks.

---

## Governed by the Spoon Game Governing Body (SGGB)
*Cary Academy · Class of 2026*
