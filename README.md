# Kippi — Marketing Website

Static marketing and waitlist site for [kippi.help](https://kippi.help).

Built with Astro. Deploys to Vercel or Netlify. Stores waitlist emails in Supabase.

## Current state
- **Live on kippi.help**, auto-deployed by Vercel on push to `main` (Vercel bot comments preview URLs on PRs; preview URLs are behind SSO/deployment-protection — verify prod after merge, not the preview).
- **SEO/AIO live (2026-07-03, PR #4):** homepage-only JSON-LD `@graph` (SoftwareApplication + FAQPage + WebSite) injected in `Base.astro` via `set:html` (gated on the `isHome` prop / root path — do NOT let it render on `/privacy` or `/terms`); keyword title + long meta description in `Base.astro`; a visible static FAQ section in `index.astro` (`§E2`, between §E and §F). All FAQ/metadata copy is **safety-locked** (Sage-approved) — never reword it without a fresh Sage pass. Source spec lives in the business repo: `marketing/content/drafts/2026-07-03-seo-structured-data.md`.

---

## Live infrastructure (current — went live 2026-06-14)

- **Host:** Vercel — project `kippi-website`, team `alon-gefens-projects` (account `alongefen`). **Auto-deploys from GitHub `main`** (push → production).
- **Domain/DNS:** registrar **Porkbun**. Records: apex `kippi.help A → 76.76.21.21`, `www CNAME → cname.vercel-dns.com` (plus a leftover `*.kippi.help → pixie.porkbun.com` wildcard). `try.kippi.help` points at the separate `kippi-validation` Vercel project. SSL = Let's Encrypt via Vercel (auto-renews).
- **Porkbun API:** DNS has a working API (`api.porkbun.com/api/json/v3`, apikey+secretapikey in the POST body; the per-domain "API ACCESS" toggle must be ON). **No email-forwarding API** (that endpoint 404s — forwarding is dashboard-only); `hello@kippi.help` forwarding is still a manual toggle.
- **Legal pages:** operator **Alon Gefen Technology** (Kadima, Israel), Israeli governing law; waitlist emails stored in Supabase **Singapore** region.
- **Known open items:** `www`→apex redirect unset (canonical tags cover SEO).

---

## Local development

**Prerequisites:** Node.js 22+, npm 10+.

```bash
# 1. Install dependencies
npm install

# 2. Set up environment variables
cp .env.example .env
# Edit .env and fill in your Supabase URL and anon key (see below)

# 3. Start the dev server
npm run dev
# Opens at http://localhost:4321
```

---

## Environment variables

Copy `.env.example` to `.env` for local dev. Add the same variables in your host dashboard for production.

| Variable | Where to find it |
|---|---|
| `PUBLIC_SUPABASE_URL` | Supabase dashboard > Project Settings > API > Project URL |
| `PUBLIC_SUPABASE_ANON_KEY` | Supabase dashboard > Project Settings > API > anon/public key |

The anon key is public-safe. The `service_role` key must NEVER appear in this project.

---

## Supabase setup (waitlist table)

Run this SQL in the Supabase SQL editor to create the waitlist table and apply the correct RLS policy.

**NOTE: Vault (security reviewer) must sign off on this RLS before the domain goes live.**

```sql
-- Create the waitlist table
CREATE TABLE public.waitlist (
  id          uuid DEFAULT gen_random_uuid() PRIMARY KEY,
  email       text NOT NULL,
  source      text CHECK (char_length(source) <= 500),
  created_at  timestamptz DEFAULT now() NOT NULL,

  CONSTRAINT waitlist_email_unique UNIQUE (email),
  CONSTRAINT waitlist_email_format CHECK (email ~* '^[^\\s@]+@[^\\s@]+\\.[^\\s@]+$')
);

-- Enable Row Level Security
ALTER TABLE public.waitlist ENABLE ROW LEVEL SECURITY;

-- Allow anonymous INSERT only (no SELECT, UPDATE, DELETE for anon role)
CREATE POLICY "anon_insert_only"
  ON public.waitlist
  FOR INSERT
  TO anon
  WITH CHECK (true);

-- Authenticated users (admin) can read everything
-- Uncomment and adjust if you need admin read access:
-- CREATE POLICY "authenticated_read"
--   ON public.waitlist
--   FOR SELECT
--   TO authenticated
--   USING (true);
```

---

## Build

```bash
npm run build
# Output goes to ./dist/ — this is what the host deploys
```

---

## Deploy to Vercel

1. Push this repo to GitHub (see below if not done yet).
2. Go to [vercel.com](https://vercel.com) and click "Add New Project."
3. Import the `kippi-website` GitHub repository.
4. Framework preset: Astro (Vercel auto-detects it).
5. Add environment variables:
   - `PUBLIC_SUPABASE_URL`
   - `PUBLIC_SUPABASE_ANON_KEY`
6. Click Deploy.
7. In Vercel project settings > Domains, add `kippi.help`.
8. Update your domain DNS:
   - Add an `A` record: `@` pointing to Vercel's IP (shown in the dashboard), OR
   - Add a `CNAME` record: `www` pointing to `cname.vercel-dns.com`
   - Vercel will issue an SSL certificate automatically.

---

## Deploy to Netlify

1. Push this repo to GitHub.
2. Go to [netlify.com](https://netlify.com) and click "Add new site" > "Import an existing project."
3. Connect GitHub and select `kippi-website`.
4. Build command: `npm run build`
5. Publish directory: `dist`
6. Add environment variables (same as Vercel above) under Site settings > Environment variables.
7. Deploy.
8. In Domain management, add `kippi.help` and follow DNS instructions.

---

## Pointing kippi.help at the host

Regardless of host, the DNS changes are:

- Vercel: follow [Vercel's custom domain docs](https://vercel.com/docs/concepts/projects/domains)
- Netlify: follow [Netlify's custom domain docs](https://docs.netlify.com/domains-https/custom-domains/)

SSL is handled automatically by both hosts.

---

## Creating the GitHub repo (if not yet done)

If the GitHub repo does not exist yet, run these commands from inside `kippi-website/`:

```bash
git init
git add .
git commit -m "Initial commit: Kippi marketing website"
gh repo create kippi-website --public --source=. --remote=origin --push
```

If `gh` is not installed: create the repo manually at github.com, then:

```bash
git init
git add .
git commit -m "Initial commit: Kippi marketing website"
git remote add origin https://github.com/alongefen/kippi-website.git
git push -u origin main
```

---

## Before go-live checklist

- [ ] Vault security review of Supabase RLS policy (insert-only, no anon SELECT)
- [ ] Sage safety review of all landing page copy and crisis line wording
- [ ] Founder review of Privacy Policy and Terms of Use (both marked DRAFT with [PLACEHOLDER] fields)
- [ ] Replace all [PLACEHOLDER] values in `/privacy` and `/terms`
- [ ] OG image: request Frame (designer) to produce a clean 1200x630 composite (Kippi + wordmark + tagline on warm-dark). Current `/public/og-image.png` is a placeholder -- do not ship without it.
- [ ] Favicon: current SVG is a hand-drawn SVG approximation. Frame can produce a higher-quality version from the expression PNGs if desired.
- [ ] Test waitlist form with real Supabase keys (dev and production)
- [ ] Run Lighthouse audit (target score 95+)

---

## Project structure

```
kippi-website/
  src/
    components/
      WaitlistForm.astro     -- email form with Supabase insert + honeypot
    layouts/
      Base.astro             -- shared head, fonts, global CSS, scroll-reveal
    pages/
      index.astro            -- landing page (sections A-H)
      privacy.astro          -- /privacy (template copy, needs founder/legal review)
      terms.astro            -- /terms (template copy, needs founder/legal review)
  public/
    mascot/                  -- Kippi PNG assets (from brand/mascot/)
    favicon.svg
    robots.txt
    og-image.png             -- PLACEHOLDER: needs Frame to produce
  .env.example               -- documents required env vars
  astro.config.mjs           -- Astro + sitemap config
```
