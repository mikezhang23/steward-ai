# Steward AI — Launch Playbook

Everything you need to go from HTML file to live landing page with email signup tracking. Built so every piece carries forward into the full product.

---

## Step 1: Buy the Domain (10 minutes)

Go to **Namecheap** or **Google Domains** (now Squarespace Domains).

First choice: `stewardai.com`  
Fallbacks: `getsteward.ai`, `steward.ai`, `usesteward.com`, `trysteward.com`

The `.ai` TLD is strong for this product if available. If you go `.com`, `stewardai.com` is clean.

**Why this matters for later:** This domain becomes your production domain. The Next.js app, API, and transactional emails all live here eventually. Don't use a throwaway.

---

## Step 2: Set Up Email (15 minutes)

**Use Google Workspace** — $7.20/month for Business Starter.

1. Go to `workspace.google.com` → Get Started
2. Use the domain you just bought
3. Verify domain ownership (Namecheap: add the TXT record Google gives you to your DNS)
4. Create your first mailbox: `hello@stewardai.com`
5. Create a second alias or mailbox: `mike@stewardai.com` (your personal one)

**Set up these aliases** (free, unlimited in Google Workspace):
- `hello@` — public-facing, goes on the landing page and waitlist confirmation emails
- `support@` — for when the product launches, route to same inbox for now
- `mike@` — your direct address for outreach, investor conversations, etc.

**Why Google Workspace over free alternatives:** You already use Gmail, so the workflow is seamless. Every email you send from `@stewardai.com` during manual validation builds brand credibility. When you're texting homeowners vendor recommendations, forwarding from `mike@stewardai.com` looks professional. And Google Workspace scales — add team members later without migrating.

**DNS records you'll add** (Google walks you through this):
- MX records (for receiving email)
- TXT record (SPF — prevents spoofing)
- CNAME record (DKIM — email authentication)
- TXT record (DMARC — deliverability)

Do all four. Takes 10 minutes and means your emails don't land in spam. This is especially important later when you're sending transactional emails (waitlist confirmations, appointment notifications).

**Once email is live:** Update the landing page footer `mailto:` link from `hello@stewardai.com` to your actual domain if different.

---

## Step 3: Set Up Supabase for Waitlist Tracking (20 minutes)

You're already using Supabase in the spec's tech stack (Next.js + Supabase + Vercel). This isn't throwaway infrastructure — this is your production database from day one.

### Create the Project

1. Go to `supabase.com` → Start Project (free tier is fine)
2. Name it `steward-ai` (or `steward-prod` — you'll keep this one)
3. Choose the region closest to Vegas: **West US**
4. Save your database password somewhere secure

### Create the Waitlist Table

Go to SQL Editor in Supabase dashboard and run:

```sql
-- Waitlist signups from landing page
CREATE TABLE waitlist (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    email TEXT NOT NULL,
    created_at TIMESTAMPTZ DEFAULT now(),
    source TEXT,          -- which page they signed up from
    referrer TEXT,        -- where they came from before landing page
    utm_source TEXT,      -- for ad tracking
    utm_medium TEXT,
    utm_campaign TEXT,
    notes TEXT,           -- manual notes (e.g., "met at open house")
    converted_at TIMESTAMPTZ,  -- when they become a real user later
    CONSTRAINT waitlist_email_unique UNIQUE (email)
);

-- Index for quick lookups
CREATE INDEX idx_waitlist_created ON waitlist (created_at DESC);
CREATE INDEX idx_waitlist_email ON waitlist (email);

-- Enable Row Level Security
ALTER TABLE waitlist ENABLE ROW LEVEL SECURITY;

-- Policy: allow anonymous inserts (landing page visitors)
CREATE POLICY "Allow anonymous inserts" ON waitlist
    FOR INSERT
    TO anon
    WITH CHECK (true);

-- Policy: only authenticated users can read (you, in dashboard)
CREATE POLICY "Allow authenticated reads" ON waitlist
    FOR SELECT
    TO authenticated
    USING (true);
```

### Why This Table Design

The `source`, `referrer`, and `utm_*` columns track where signups come from — critical when you run the $200 Facebook ad test. You'll be able to see exactly which ad creative drove which signups.

The `converted_at` column lets you track the full funnel later: waitlist signup → beta user → paying customer. Don't delete this table when you launch — it becomes your acquisition analytics.

The `notes` column is for when you manually add people (from the validation conversations, networking events, etc.). Not everything comes through the form.

### Get Your Keys

1. Go to Settings → API in Supabase dashboard
2. Copy **Project URL** — looks like `https://abc123.supabase.co`
3. Copy **anon/public key** — this is safe for client-side use
4. **Never** expose the service_role key in frontend code

### Update the Landing Page

Open `steward-landing.html` and replace the two config lines near the bottom:

```javascript
const SUPABASE_URL = 'https://your-actual-project.supabase.co';
const SUPABASE_ANON_KEY = 'your-actual-anon-key';
```

Test locally by opening the HTML file and submitting a test email. Check the Supabase Table Editor to confirm the row appeared.

---

## Step 4: Deploy on Vercel (15 minutes)

Vercel is in your spec's tech stack. When you build the full Next.js app, it deploys here. Starting with the landing page means your domain, SSL, and deployment pipeline are already wired up.

### Option A: Quick Deploy (No Git — Get Live in 5 Minutes)

1. Install Vercel CLI: `npm i -g vercel`
2. Create a project folder:
   ```
   mkdir steward-landing
   mv steward-landing.html steward-landing/index.html
   ```
3. Deploy:
   ```
   cd steward-landing
   vercel
   ```
4. Follow the prompts (link to your Vercel account, name the project `steward-ai`)
5. Vercel gives you a `.vercel.app` URL immediately

### Option B: Git Deploy (Recommended — 15 Minutes)

This is better because every future change is just a `git push`.

1. Create a GitHub repo: `steward-ai` (private)
2. Create this folder structure:
   ```
   steward-ai/
   ├── public/
   │   └── index.html      ← your landing page
   └── package.json
   ```
3. Minimal package.json:
   ```json
   {
     "name": "steward-ai",
     "version": "0.1.0",
     "scripts": {
       "build": "echo 'Static site, no build needed'"
     }
   }
   ```
4. Push to GitHub
5. Go to `vercel.com` → Import Project → Select your repo
6. Framework Preset: **Other**
7. Output Directory: `public`
8. Deploy

### Connect Your Domain

1. In Vercel dashboard → your project → Settings → Domains
2. Add `stewardai.com` (or your domain)
3. Vercel tells you which DNS records to add:
   - Either an A record pointing to `76.76.21.21`
   - Or a CNAME pointing to `cname.vercel-dns.com`
4. Add the record in your domain registrar's DNS settings
5. SSL is automatic — Vercel provisions a certificate

### Future-Proofing

When you're ready to build the real app, you'll:
1. Run `npx create-next-app@latest` in this same repo
2. Move `index.html` content into a Next.js page (`app/page.tsx`)
3. Push to GitHub — Vercel auto-deploys the Next.js app on the same domain
4. Zero downtime, same URL, same Supabase backend

---

## Step 5: Wire Up Ad Tracking (10 minutes, do before running ads)

### Facebook Pixel

1. Go to Meta Business Suite → Events Manager → Create Pixel
2. Name it `Steward AI`
3. Add the pixel base code to `<head>` in your landing page:
   ```html
   <!-- Meta Pixel -->
   <script>
   !function(f,b,e,v,n,t,s){...}(window,document,'script','https://connect.facebook.net/en_US/fbevents.js');
   fbq('init', 'YOUR_PIXEL_ID');
   fbq('track', 'PageView');
   </script>
   ```
4. Fire a Lead event on successful signup — add this inside the `showSuccess` function:
   ```javascript
   if (typeof fbq !== 'undefined') fbq('track', 'Lead');
   ```

This lets Facebook optimize your $200 ad spend toward people most likely to sign up, and gives you exact cost-per-lead numbers for the validation dashboard.

### Google Analytics (Optional but Free)

1. Create a GA4 property at `analytics.google.com`
2. Add the gtag snippet to `<head>`
3. Useful for: seeing where organic traffic comes from, how far people scroll, which pricing card they click

If you're only running Facebook ads for the $200 test, the Pixel alone is sufficient. GA4 is more useful when you start doing SEO/content marketing later.

---

## Step 6: Test Everything (15 minutes)

Before sharing the URL with anyone:

1. **Desktop**: Load the page, submit a test email, verify it appears in Supabase
2. **iPhone Safari**: Check every section for readability, tap all buttons, submit the form
3. **Android Chrome**: Same checks
4. **UTM tracking**: Visit `yourdomain.com?utm_source=test&utm_medium=test&utm_campaign=test`, submit an email, verify UTM columns populate in Supabase
5. **Email**: Send a test from `hello@stewardai.com` to your personal Gmail. Check it doesn't land in spam. Reply to it and confirm replies route correctly.
6. **Speed**: Run `pagespeed.web.dev` on your URL. Single-page HTML with no heavy assets should score 95+.

### Clean Up Test Data

Delete your test rows from the Supabase `waitlist` table before going live.

---

## What You'll Have When Done

- **stewardai.com** (or your domain) — live, SSL, fast
- **hello@stewardai.com** — professional email that works today and scales with the business
- **Supabase project** — production database that will power the full app, currently tracking waitlist signups with attribution data
- **Vercel deployment** — same platform, same domain, same pipeline you'll use for the Next.js app
- **Facebook Pixel** — ready for the $200 ad test from the validation plan
- **Git repo** — version controlled, ready to evolve into the full codebase

Nothing is throwaway. Every piece you set up today is the foundation for the product.

---

## Total Time: ~1.5 Hours

| Step | Time | Cost |
|------|------|------|
| Buy domain | 10 min | ~$12/year |
| Google Workspace | 15 min | $7.20/month |
| Supabase setup | 20 min | Free (for now) |
| Vercel deploy | 15 min | Free (for now) |
| Ad tracking | 10 min | Free |
| Testing | 15 min | — |
| **Total** | **~1.5 hrs** | **~$8.20/month + domain** |
