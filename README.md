# AIMAVA · AI Sprint Workbook

A single-page web app where each participant fills in the four-canvas AI Sprint
framework, **saves** progress against their email, **exports** a copy to their
device, and **submits** the finished workbook to the facilitators.

```
index.html    the whole frontend (no build step, no framework)
config.js     your two Supabase values go here
schema.sql    run once in Supabase to create the backend
Aimava.png    logo used in the header
```

The app works with **no backend at all** (local-only mode: Save / Load / Export
use the browser). Add Supabase to get cross-device Save and Submit.

---

## Part 1 — Review it now (no setup)

1. Serve the folder locally (a plain double-click on `file://` also works for
   review, but a local server is closer to production and avoids browser quirks):

   ```powershell
   cd C:\Users\Denis\ClaudeCode_Prjs\ai_workbook
   python -m http.server 5173
   ```

2. Open <http://localhost:5173>. You will see the periwinkle Aimava header, the
   four canvases, and a "Local-only mode" banner.
3. Try **Save**, reload the page, then **Load my saved workbook** — your text
   comes back from the browser. Try **Export (.json)** and **Print / PDF**.

---

## Part 2 — Stand up the Supabase backend

### Step 1 — Create the project
1. Sign in at <https://supabase.com> → **New project**.
2. Name it (e.g. `aimava-sprint`), set a database password (save it in your
   password manager), pick the region closest to your participants.
3. Wait ~2 minutes for it to provision. **Free tier is enough** — see notes below.

### Step 2 — Create the tables and functions
1. Left sidebar → **SQL Editor** → **New query**.
2. Open `schema.sql` from this folder, copy everything, paste it in, click **Run**.
3. You should see `Success. No rows returned`. Under **Table Editor** you now have
   `workbooks` and `submissions`.

### Step 3 — Get your API values
1. Left sidebar → **Project Settings** (gear icon) → **API**.
2. Copy **Project URL** (looks like `https://abcdefgh.supabase.co`).
3. Copy the **`anon` / `public`** key (a long JWT). **Not** the `service_role`
   key — that one is a secret and must never go in frontend code.

### Step 4 — Wire up the frontend
Open `config.js` and paste the two values:

```js
window.AIMAVA_CONFIG = {
  SUPABASE_URL:      "https://abcdefgh.supabase.co",
  SUPABASE_ANON_KEY: "eyJhbGciOi...your-anon-key..."
};
```

Reload the app. The banner should now read **"Connected to the server."**

### Step 5 — Test the round trip
1. Fill in your email + a few fields → **Save** → status shows "Saved to the server".
2. In Supabase → **Table Editor → workbooks**: your row is there, `data` holds the JSON.
3. Open the app in a different browser, enter the same email → **Load** → your
   work appears (this is the cross-device pick-up).
4. **Submit to facilitators** → a row lands in `submissions` and a `.json` copy
   downloads to your device.

### Step 6 — Deploy (static hosting, free)
The app is three static files. Any of these work — drag-and-drop the folder:

| Host | How |
|------|-----|
| **Netlify Drop** | <https://app.netlify.com/drop> — drop the folder, done. |
| **Cloudflare Pages** | New project → Direct Upload → upload folder. |
| **Vercel** | `vercel` in the folder, or drag-drop in the dashboard. |
| **GitHub Pages** | Push the folder to a repo, enable Pages on the branch. |

No server, no environment variables — `config.js` ships with the site. (The
`anon` key is designed to be public; your protection is that it can only call the
three functions in `schema.sql`.)

### Step 7 — Read the results
- **Table Editor → submissions** for a grid view, or
- **SQL Editor** and run the facilitator queries at the bottom of `schema.sql`
  (all submissions; who has a draft but hasn't submitted).
- Export from Supabase: any table view has a **CSV download**.

---

## How the pieces fit

```
Browser (index.html + Supabase JS)
   |  supabase.rpc('save_draft'  , ...)   on Save
   |  supabase.rpc('load_draft'  , email) on Load
   |  supabase.rpc('submit_workbook', ...) on Submit
   v
Supabase PostgREST  --> 3 SECURITY DEFINER functions --> workbooks / submissions
                        (tables have RLS on + no policies, so the anon key
                         cannot touch them except through these functions)
```

- **Save** = upsert the whole workbook JSON into `workbooks`, keyed by lower-cased email.
- **Load** = fetch that row back by email.
- **Submit** = save + set `submitted_at` + append a row to `submissions`.
- **Export** and **Print/PDF** are 100% client-side; no backend involved.
- The browser also keeps an autosaved copy in `localStorage` as a safety net; on
  Load, the newer of (server copy, browser copy) wins.

---

## Free tier notes

| Limit (free) | Impact for a cohort |
|---|---|
| 500 MB database | A workbook is a few KB. Non-issue. |
| Project **pauses after 7 days idle** | During a programme with sessions every few days it won't pause. Between cohorts: click **Restore** in the dashboard (data is intact), or hit any function weekly to keep it warm. |
| No automated backups | Do your own: after each session run the "all submissions" query and CSV-download it, or `pg_dump` via the connection string. |

Upgrade to Pro ($25/mo) only if you want always-on + daily backups.

---

## Optional hardening (later)

The current model: anyone with the site URL can load/overwrite a workbook if they
know the email. Fine for the sanitised material this programme uses. If you want more:

1. **Cohort passcode** — add a `p_passcode text` argument to the three functions
   and `raise exception` unless it matches a value you share with the cohort.
2. **Magic-link auth** — Supabase Auth → enable Email; switch the functions to
   `auth.email()` instead of a passed-in `p_email`, and add RLS policies
   `using (email = auth.email())`. One extra click for participants, real
   per-user isolation.
3. **Rate-limit** `load_draft` (e.g. via `pg_cron` + a counter table) to blunt
   email enumeration.
4. **Custom SMTP** (Auth → SMTP settings) if you go the magic-link route, so
   invite mails come from your domain rather than Supabase's shared sender.
