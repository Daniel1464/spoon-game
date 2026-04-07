# 🥄 Spoon Game — Cary Academy

A web app for managing and playing the Cary Academy Senior Spoon Game, built for the Class of 2025. Supports player login, target tracking, elimination codes, admin management, and a faculty fork view.

---

## How it works

Each senior is assigned a secret target and given a unique 6-character elimination code. To eliminate a target, a player must catch them outside a safe zone, touch them with their spoon, and say *"You've been spooned, friend."* The eliminated player gives their code to the eliminator, who enters it into the app. The eliminator then inherits the eliminated player's target, and the chain continues until one player remains.

---

## Views

### Player
- Log in with a unique 6-character access code
- See current target, game status, and % of players remaining
- Display personal elimination code to share if eliminated
- Enter a target's elimination code to record an elimination

### Admin
- Password-protected dashboard
- Add, remove, restore, and manually eliminate players
- View full player list with targets, elimination counts, and codes
- Reshuffle all targets randomly
- Set game status: Single target / Open season / Forks active
- Print player access cards with QR codes for distribution
- Manage faculty forks

### Fork (Faculty)
- Separate password-protected view
- Shows all players and their active/eliminated status
- Live count of players remaining

### Public board (pre-login)
- Visible to anyone before logging in
- Running eliminations log (who spooned whom)
- Full official rules of play

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
  constraint single_row check (id = 1)
);

create table eliminations (
  id uuid primary key default gen_random_uuid(),
  victim_id uuid references players(id),
  victim_name text not null,
  killer_id uuid references players(id),
  killer_name text not null,
  created_at timestamptz default now()
);

create table passwords (
  key text primary key,
  hash text not null
);

insert into game_state (id, status) values (1, 'single');
```

**Row Level Security:**
```sql
alter table players enable row level security;
alter table forks enable row level security;
alter table game_state enable row level security;
alter table eliminations enable row level security;
alter table passwords enable row level security;

create policy "read players"      on players     for select using (true);
create policy "insert players"    on players     for insert with check (true);
create policy "update players"    on players     for update using (true);
create policy "delete players"    on players     for delete using (true);

create policy "read forks"        on forks       for select using (true);
create policy "insert forks"      on forks       for insert with check (true);
create policy "update forks"      on forks       for update using (true);
create policy "delete forks"      on forks       for delete using (true);

create policy "read state"        on game_state  for select using (true);
create policy "update state"      on game_state  for update using (true);

create policy "read eliminations" on eliminations for select using (true);
create policy "write eliminations" on eliminations for insert with check (true);

create policy "read passwords"    on passwords   for select using (true);
create policy "write passwords"   on passwords   for all using (true);
```

**Passwords** (replace values before running):
```sql
insert into passwords (key, hash) values
  ('admin', encode(digest('your_admin_password', 'sha256'), 'hex')),
  ('fork',  encode(digest('your_fork_password',  'sha256'), 'hex'));
```

### 2. Deploy to GitHub Pages

1. Create a new **public** repository on GitHub
2. Upload `index.html` to the repository root
3. Go to **Settings → Pages**, set source to **main** branch, **/ (root)** folder
4. Your app will be live at `https://yourusername.github.io/your-repo-name`

To update the app, upload a new `index.html` — GitHub Pages redeploys automatically within about a minute.

---

## Changing passwords

Run this in the Supabase SQL Editor, replacing the value with your new password:

```sql
-- Admin password
update passwords set hash = encode(digest('newpassword', 'sha256'), 'hex') where key = 'admin';

-- Fork password
update passwords set hash = encode(digest('newpassword', 'sha256'), 'hex') where key = 'fork';
```

---

## Security model

- **Passwords** are never stored in the HTML file. The browser hashes the entered password using the built-in `SubtleCrypto` API and compares it against the hash in Supabase. A student reading source code sees only hashing logic, never the password itself.
- **Player data** is partitioned by role. Players only receive their own row plus a minimal list of names for target display — no other codes or targets are ever sent to their browser.
- **Target IDs** are fetched only as needed (during elimination processing) and only the fields required for reassignment are requested.
- **Admin and fork access** is gated client-side by password and server-side by the anonymous Supabase key — all writes go through RLS policies.

---

## Official Rules of Play

1. Above all else: this is a game for the well-mannered and polite. No tomfoolery, hoodwinkery, or — worst of all — shenanigans.
2. The game starts at **noon on Wednesday, April 16th**. Target assignments distributed that morning.
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
*Cary Academy · Class of 2025*
