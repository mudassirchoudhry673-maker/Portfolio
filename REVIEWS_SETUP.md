# Reviews feature (Supabase)

Visitors can submit a review with **name**, **email**, optional **profile photo**, **1–5 stars**, and **text**. Rows are saved as **`pending`**. Only **`approved`** reviews appear on the site. **Email is stored for you** and is **not** selected in the app for public display.

**Status (Table Editor → `status` column):** **`pending`** (new), **`approved`** (live on site), **`rejected`** (hidden), **`deleted`** (soft delete — hidden, row kept). Run **`003_reviews_status_deleted.sql`** if your table was created before `deleted` existed.

---

## Step 1: Create a Supabase project

1. Go to [https://supabase.com](https://supabase.com) and sign in.
2. **New project** → choose organization, name, database password, region.
3. Wait until the project finishes provisioning.

---

## Step 2: Run the database SQL

1. In Supabase: **SQL Editor** → **New query**.
2. Open `supabase/migrations/001_reviews.sql`, copy the full file, paste, **Run**.
3. Open `supabase/migrations/002_reviews_enhancements.sql`, copy, paste, **Run**. It **removes duplicate emails** first (keeps the **newest** row per address), then adds the unique index. If you need to keep a specific duplicate row instead, delete the others manually in Table Editor, then run `002`.

4. Open `supabase/migrations/003_reviews_status_deleted.sql`, copy, paste, **Run** (adds **`deleted`** as a valid `status` for soft-remove).

**`001` creates:** table **`reviews`**, RLS, bucket **`review-avatars`**.

**`002` adds:** unique **`email`**, **`email_hash`**, **`attachment_url`**, trigger, bucket **`review-attachments`**.

**`003` updates:** `status` check so you can use **`pending`**, **`approved`**, **`rejected`**, **`deleted`** in Table Editor.

---

## Step 3: API keys in your app

1. Supabase: **Project Settings** → **API**.
2. Copy **Project URL** and **`anon` `public` key** (not the `service_role` key).
3. In the project root, create **`.env`** (same folder as `package.json`):

```env
VITE_SUPABASE_URL=https://YOUR_PROJECT_REF.supabase.co
VITE_SUPABASE_ANON_KEY=your_anon_key_here
```

4. Restart the dev server (`npm run dev`) after changing `.env`.

`.env` is gitignored; use `.env.example` as a template.

---

## Step 4: Approve or delete reviews (moderation)

1. Supabase → **Table Editor** → **`reviews`**.
2. New submissions appear with **`status` = `pending`** (you can sort by **`created_at`**).
3. To **publish** on the portfolio: set **`status`** to **`approved`**.
4. To **hide** (declined): set **`status`** to **`rejected`**.
5. To **soft delete** (hidden like removed, row kept): set **`status`** to **`deleted`**.
6. To **remove** the row permanently: use **Delete row** in Table Editor (hard delete).

There is no admin UI in this repo; the dashboard is your moderation panel.

---

## Step 5: Email when someone submits (optional)

Supabase does not email you automatically on insert. The site does **not** send mail unless you add one of the options below.

**“Approve / don’t show” buttons inside the email** need a server (e.g. Supabase Edge Function + signed links). This project uses **Supabase Table Editor** instead: you open the table, set **`approved`** to show on the portfolio or **`rejected`** / delete to hide.

### A) EmailJS (simple “new review” email to your inbox)

1. Create a free account at [https://www.emailjs.com](https://www.emailjs.com).
2. Add an **Email Service** (e.g. Gmail) and connect the inbox where you want alerts (**check spelling**: `gmail.com`, not `gamil.com`).
3. Create an **Email Template** with these **template variables** (names NAst match):

   - `{{reviewer_name}}`
   - `{{reviewer_email}}`
   - `{{rating}}`
   - `{{review_preview}}`
   - `{{instructions}}` — use this in the body for “how to approve” text (the app fills it).

   Example subject: `New portfolio review (pending)`.

4. In **Account → API Keys**, copy the **Public Key**.
5. In `.env` (and Vercel **Environment Variables**), add:

```env
VITE_EMAILJS_PUBLIC_KEY=your_public_key
VITE_EMAILJS_SERVICE_ID=your_service_id
VITE_EMAILJS_TEMPLATE_ID=your_template_id
```

6. Optional — one link in the email to your `reviews` table:

```env
VITE_REVIEW_SUPABASE_TABLE_URL=https://supabase.com/dashboard/project/YOUR_PROJECT_REF/editor/YOUR_TABLE_ROUTE
```

   In Supabase, open **Table Editor** → **`reviews`**, copy the URL from the browser bar, paste here.

7. **Redeploy** after changing env vars.

If EmailJS is not configured, reviews still save to Supabase; you just won’t get an email.

### B) Webhook URL (built into this app)

Set in `.env`:

```env
VITE_REVIEW_NOTIFY_WEBHOOK=https://hooks.zapier.com/hooks/catch/...
```

(or a **Make** webhook, **Discord** incoming webhook, etc.)

After a successful submit, the site sends a **POST** with JSON like:

```json
{
  "name": "...",
  "email": "...",
  "rating": 5,
  "body_preview": "first 280 chars...",
  "submitted_at": "ISO timestamp",
  "message": "New portfolio review pending approval"
}
```

Configure the automation to **email you**.  
Note: the webhook URL is visible in the built frontend bundle; use a **disposable / scoped** automation URL, not a master API key.

### C) Supabase Database Webhooks

In Supabase: **Database** → **Webhooks** → create a webhook on **`reviews`** **INSERT** to your server or automation URL.

### D) No automation

Open **Table Editor** periodically and filter **`status` = `pending`**.

---

## Step 6: Production (e.g. Vercel)

Add the same **`VITE_***`** variables in the host’s **Environment Variables**, redeploy.

---

## Troubleshooting

| Issue | What to check |
|--------|----------------|
| Submit fails | SQL migration ran; RLS policies exist; keys in `.env` are correct |
| Upload fails | Bucket **`review-avatars`** exists and is **public**; storage policies from SQL applied |
| Reviews don’t show | Row **`status`** NAst be **`approved`** |
| Yellow hint on site | Missing **`VITE_SUPABASE_URL`** or **`VITE_SUPABASE_ANON_KEY`** |
| No email on new review | Email was never wired by default; set **EmailJS** (Step 5A) or a **webhook** (Step 5B), then redeploy |
| **`23514` … `reviews_status_check`** when setting **`deleted`** | Run **`003_reviews_status_deleted.sql`** (or paste the fix from that file) so `status` allows **`deleted`**. |

---

## Changing footer text (“Portfolio review”)

Edit the label next to the globe icon in `src/components/Reviews.tsx` inside `ReviewCard`.
