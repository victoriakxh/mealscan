# NomNom — Cloud sync setup (Supabase)

The app works **local-only** until you fill in two values. Follow these steps once to turn on accounts + cloud sync with per-user data isolation.

Estimated time: ~15 minutes. Cost: free tier.

---

## 1. Create a Supabase project
1. Go to https://supabase.com, sign up, and create a **New project** (pick any name, set a database password, choose a region near you).
2. Wait ~2 minutes for it to provision.

## 2. Get your two keys
In the project: **Settings → API**. Copy:
- **Project URL** (looks like `https://abcdefgh.supabase.co`)
- **anon public** key (a long string under "Project API keys")

These two are *safe to put in the app's public code* — they're designed to be public. Your data is protected by the database rules in step 4, not by hiding the key.

## 3. Paste them into the app
Open `mealscan.html`, find these two lines near the top of the `<script>`:

```js
const SUPABASE_URL='';       // e.g. https://abcdefgh.supabase.co
const SUPABASE_ANON_KEY='';  // your project's anon (public) key
```

Fill them in:

```js
const SUPABASE_URL='https://abcdefgh.supabase.co';
const SUPABASE_ANON_KEY='eyJhbGciOi...your-anon-key...';
```

(Leaving them blank keeps the app local-only with no login.)

## 4. Create the table + security rules
In Supabase: **SQL Editor → New query**, paste this, and click **Run**:

```sql
create table if not exists public.profiles (
  id uuid references auth.users on delete cascade primary key,
  data jsonb,
  updated_at timestamptz default now()
);

alter table public.profiles enable row level security;

create policy "read own profile"   on public.profiles
  for select using (auth.uid() = id);
create policy "insert own profile" on public.profiles
  for insert with check (auth.uid() = id);
create policy "update own profile" on public.profiles
  for update using (auth.uid() = id) with check (auth.uid() = id);
```

This is what guarantees separation: each row is one user's data, and the rules let a signed-in user touch **only their own row**. Even though the app's key is public, nobody can read anyone else's data.

## 5. Turn on sign-in methods
**Authentication → Providers:**
- **Email** is on by default — this powers the magic link. Nothing to do. (You can leave "Confirm email" on.)
- **Google** (optional): toggle it on, then follow Supabase's link to create Google OAuth credentials in the Google Cloud Console. In Google, add this as an authorized redirect URI:
  `https://YOUR-PROJECT.supabase.co/auth/v1/callback`
  Paste the Google Client ID + Secret back into Supabase and save. *(Skip this if you only want magic-link email sign-in.)*

## 6. Allow your app's web address
**Authentication → URL Configuration:**
- **Site URL:** your app's address, e.g. `https://YOURNAME.github.io/YOURREPO/`
- **Redirect URLs:** add the same address (with a trailing `*` is fine, e.g. `https://YOURNAME.github.io/YOURREPO/**`).

This lets the magic link and Google redirect land back on your app.

## 7. Deploy and test
1. Upload the edited `mealscan.html` (as `index.html`) to GitHub Pages, alongside `opennutrition.json.gz`.
2. Open the app — you'll now see a **sign-in screen**.
3. Sign in with your email (magic link) or Google.
4. On your **first** sign-in, the app uploads your existing on-device data into your account automatically. After that it syncs to the cloud, with the device copy kept as an offline cache.
5. Have each family member open the app and sign in with *their own* email/Google — they each get a separate, private profile.

---

## Notes
- **Free-tier pause:** a free Supabase project pauses after **7 days with no activity** (data is kept; you click to resume). If your group uses it regularly it stays awake. To be safe, set up a free keep-alive ping (Uptime Robot or a GitHub Actions cron hitting your project) — optional.
- **Migrate from the right device:** the first sign-in uploads whatever data is in *that* browser. Sign in first from the device/browser that holds your real history (remember a Safari tab and the home-screen app can have separate copies).
- **Sign out** is in **Settings → Account**.
- **Capacity:** the 500 MB free database holds personal logs only (the food database stays on GitHub Pages), so it's effectively unlimited for friends-and-family scale.
