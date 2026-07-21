# Celestial — Deployment Guide

Celestial is an Expo (React Native) app with a Supabase backend.

- **Web** → built with `expo export` and served on GitHub Pages
- **Mobile** → built with EAS for the App Store / Play Store
- **Backend** → Supabase (auth + Postgres + edge functions)
- **AI** → Anthropic Claude (`claude-haiku-4-5-20251001`)
- **Payments** → Stripe (subscriptions)

This repo is standalone: the app lives at the repo **root** (no `Celestial/` subfolder).

---

## How deploys happen (CI/CD)

Two GitHub Actions workflows run on push to `main`:

| Workflow | File | Trigger | What it does |
|---|---|---|---|
| Deploy Celestial to Pages | `.github/workflows/static.yml` | every push to `main` (+ manual) | `npm ci` → `expo export -p web` → publish `dist/` to GitHub Pages |
| Deploy Supabase Functions | `.github/workflows/deploy-functions.yml` | push to `main` touching `supabase/**` (+ manual) | `supabase functions deploy` (all functions) |
| EAS Build & Submit | `.github/workflows/eas-build.yml` | manual / push to `main` | builds native binaries via EAS |

Both deploy workflows can also be run manually from the **Actions** tab ("Run workflow").

### Live URLs
- **Web app:** https://nomadwill78.github.io/celestial
- **Supabase project ref:** `rjucvqthsseegxlwryru`
- **Edge functions base:** `https://rjucvqthsseegxlwryru.functions.supabase.co/<function-name>`

---

## Required secrets

### A. GitHub repository secrets
`Settings → Secrets and variables → Actions`

| Secret | Used by | Where to get it |
|---|---|---|
| `SUPABASE_ACCESS_TOKEN` | `deploy-functions.yml` | supabase.com/dashboard/account/tokens |
| `EXPO_TOKEN` | `eas-build.yml` | expo.dev → Account → Access Tokens |

### B. Supabase Edge Function secrets
`Supabase Dashboard → Project Settings → Edge Functions → Secrets`
(or `supabase secrets set NAME=value --project-ref rjucvqthsseegxlwryru`)

| Secret | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | AI readings: advisor, horoscope, birth-chart, compatibility |
| `STRIPE_SECRET_KEY` | Stripe checkout + billing portal |
| `STRIPE_STARSEED_MONTHLY_PRICE_ID` | Starseed plan price |
| `STRIPE_COSMIC_YEARLY_PRICE_ID` | Cosmic plan price |
| `STRIPE_WEBHOOK_SECRET` | Verifies incoming Stripe webhooks |
| `APP_URL` | Redirect base, e.g. `https://nomadwill78.github.io/celestial/` |

> **Auto-injected — do NOT set these:** `SUPABASE_URL`, `SUPABASE_ANON_KEY`,
> `SUPABASE_SERVICE_ROLE_KEY`, `SUPABASE_DB_URL`. Supabase provides them to every
> edge function automatically.

### C. Web build public keys
These are **public** client keys, committed in `static.yml` (safe by design):
`EXPO_PUBLIC_SUPABASE_URL`, `EXPO_PUBLIC_SUPABASE_ANON_KEY`,
`EXPO_PUBLIC_STRIPE_PUBLISHABLE_KEY`, `EXPO_PUBLIC_APP_URL`. Update them there if
the project changes.

---

## 1. Web / GitHub Pages

One-time setup:
1. `Settings → Pages → Build and deployment → Source = **GitHub Actions**`.
2. The app serves from the `/celestial` subpath, so `app.json` sets
   `expo.experiments.baseUrl = "/celestial"`. If you rename the repo or use a
   custom domain, update `baseUrl` and `EXPO_PUBLIC_APP_URL` in `static.yml`.
3. Push to `main` (or run the workflow manually) → site publishes to
   https://nomadwill78.github.io/celestial

## 2. Supabase

### Migrations
Apply in order (`001` → `002` → `003`) via `supabase db push`, or paste each file
from `supabase/migrations/` into the SQL editor:
- `001_initial.sql` — profiles, subscriptions, saved_readings, notification_log, RLS, signup trigger
- `002_fix_signup.sql` — hardens the new-user trigger (`SECURITY DEFINER SET search_path`)
- `003_advisor_usage.sql` — `advisor_usage` table (per-account free "Ask Celeste" counter; read-only to clients, written only by the service role)

### Edge functions (7)
Deployed automatically by `deploy-functions.yml`, or manually:
```bash
supabase functions deploy --project-ref rjucvqthsseegxlwryru
```
| Function | Auth | Notes |
|---|---|---|
| `advisor` | user JWT required | 3 free questions for non-Cosmic; unlimited for Cosmic |
| `horoscope` | user JWT required | "general" free; love/career/wellness require premium |
| `birth-chart` | user JWT + premium | full natal interpretation |
| `compatibility` | user JWT + premium | returns validated JSON |
| `stripe-checkout` | user JWT | creates a Checkout session |
| `stripe-portal` | user JWT | opens the billing portal |
| `stripe-webhook` | Stripe signature (`verify_jwt = false`) | syncs subscription state |

`supabase/functions/_shared/entitlement.ts` is a shared helper (JWT → plan lookup),
not a deployable function — the `_`-prefix keeps the CLI from deploying it.

## 3. Stripe

1. Create two recurring products and copy their **Price IDs**:
   - Starseed — $9.99/month → `STRIPE_STARSEED_MONTHLY_PRICE_ID`
   - Cosmic — $79.99/year → `STRIPE_COSMIC_YEARLY_PRICE_ID`
2. Add a webhook endpoint:
   - **URL:** `https://rjucvqthsseegxlwryru.functions.supabase.co/stripe-webhook`
   - **Events:** `checkout.session.completed`, `customer.subscription.deleted`, `invoice.payment_failed`
     (recommended to also add: `customer.subscription.updated`, `invoice.payment_succeeded`)
   - Copy the signing secret → `STRIPE_WEBHOOK_SECRET`.

## 4. Anthropic

Create a key at console.anthropic.com → set `ANTHROPIC_API_KEY` in Supabase secrets.
Without it, the AI screens fall back to canned copy (they never hard-fail).

---

## Local development

```bash
cp .env.example .env.local   # fill EXPO_PUBLIC_* values
npm install
npx expo start               # press "w" for web
```
Fonts (Playfair Display + Inter) load from `@expo-google-fonts/*` at runtime —
no manual font files needed.

---

## Mobile (EAS)

```bash
npm install -g eas-cli && eas login
eas build:configure                 # sets projectId in app.json / eas.json
eas build --platform android --profile preview     # test build
eas build --platform all --profile production      # store builds
eas submit --platform android        # or: ios
```
Store assets: 512×512 (Play) / 1024×1024 (App Store) icons, plus screenshots.

---

## Go-live checklist

- [ ] Pages source set to "GitHub Actions"; web deploy green
- [ ] Migrations `001`–`003` applied
- [ ] All 7 edge functions deployed
- [ ] GitHub secrets set: `SUPABASE_ACCESS_TOKEN` (+ `EXPO_TOKEN` for mobile)
- [ ] Supabase secrets set: `ANTHROPIC_API_KEY`, all `STRIPE_*`, `APP_URL`
- [ ] Stripe products created; webhook endpoint + `STRIPE_WEBHOOK_SECRET` configured
- [ ] Sign up → onboarding → home flow works end to end
- [ ] Ask Celeste free-question limit + upgrade path verified
- [ ] Checkout and "Manage Billing" verified against Stripe
