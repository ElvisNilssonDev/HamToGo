# Christmas Ham Booking System

> Seasonal booking system for a Swedish slaughterhouse's Christmas products (julskinka, revben, kalkon).
> Working title — final brand/domain name still open (e.g. `bokningar.se`, `slakteribokning.se`). Avoid a ham-only name so it survives expansion.

---
| Deploy | Railway (API + PostgreSQL) + Cloudflare Pages (frontend, React + Vite) | free tier |
- **Why not Vercel:** the Hobby plan is non-commercial only. Cloudflare Pages
  is free for commercial use with unlimited static bandwidth (or Vercel Pro,
  $20/mo, if you prefer its DX). The static frontend costs ~0 kr either way —
  this is a licensing fix, not a budget one.

## Overview

Replaces a paper-binder workflow — where finding a single order takes 3–5 minutes — with a searchable digital system. The operation handles **1,800+ orders per season**, active **October 1 through December 31**. Customers book online (or staff enter orders for them); staff stamp a physical number on each finished ham and record it; customers pay on-site at pickup via Swish.

**v1 is built for one customer** — your own slaughterhouse. Single-tenant, with that one company's products seeded directly. Multi-tenancy and scaling to other slaughterhouses come later, once there's a real second customer.

---

## Scope

**v1 (single customer)**
- One slaughterhouse, one product catalog (seeded, not configurable).
- The full booking flow + staff dashboard.
- No multi-tenancy, no per-tenant catalog management, no AI.

**Later (only when a second customer is real)**
- Multi-tenancy (`TenantId` on everything, central query filter, path-based tenant resolution `yoursite.se/almos`).
- A catalog-management UI so each company defines its own products.
- Subdomains / custom domains as premium tiers.

Skipping multi-tenancy in v1 is a deliberate call: adding `TenantId` later is a real but manageable refactor, and it keeps v1 small enough to actually ship. Don't build the scaling machinery before the first customer is live.

---

## Users

- **Guest** — views the homepage and places a booking without an account.
- **Registered customer** — same booking flow, order tied to their account.
- **Staff / admin** — signed-in dashboard to manage all orders.

---

## Features

### Customer-facing

A single clear, modern flow.

**Homepage (guest-accessible)**
- App name at the top.
- Headline: *"Book your Christmas ham."*
- One large **Start here** button.

**Booking flow** — guided, interactive:
1. Add ham (size + preparation type).
2. Add extra options / additional products.
3. **Choose pickup day** — from a configured window (e.g. Dec 18–23), with an
   optional per-day capacity and a booking cutoff (e.g. orders close X days
   before the chosen day).
4. Special requests.
5. Save the booking and carry it forward.

**User information**
- Either enter contact details — **First name, Last name, Email, Phone number** — or log in.
- If the customer logs in **mid-flow, the in-progress booking is preserved** and transfers to the confirm page. (Keep the booking in frontend state, POST only on confirm, and do login in-place so the app doesn't unmount.)

**Confirm**
- Click **Confirm order**.
- Receive a confirmation, including an **email confirmation** (sent async/non-blocking so a failed send never fails the order).
- The confirmation includes an **order reference** so a guest with no account can look the booking up later (reference + email).

### Admin / business-facing

The signed-in front page is a dashboard of blocks.

- **View Orders** — all orders, searchable.
- **Add Order** — staff enter a phone/walk-in order. Reuses the *same* `CreateBooking` logic as the public flow.
- **Search** inside View Orders.
- **Ham numbers** — staff record a `HamNumber` on the order **after creation**, once the physical ham is stamped.
- **Unhandled view** — outstanding orders, driven by **order status** (not by whether `HamNumber` is empty).
- **Edit / cancel** — staff can fix a typo, change a size, or cancel.
- **Stats** *(deferred)* — hams/orders per day/week/month. A `GROUP BY date` query to add later, not a subsystem.
- - **Daily pickup list (v1, not "stats")** — the production view: pick a date,
  see every order due that day **plus an aggregate** ("Dec 22: 14× rimmad 5kg,
  9× kokt 3kg, 3× kalkon"). This is what replaces the binder on the production
  side. Print-friendly via a simple CSS print stylesheet — December staff will
  print it.
- **CSV export** — one button: all orders (or the current filter) as CSV.
  Trivial to build, disproportionately loved.
- **Search keys** — name, phone, email, order reference, ham number.

---

## Domain model

- **Order status lifecycle:** explicit `OrderStatus` — **Booked → Stamped → Collected**, plus **Cancelled**. Status is the workflow; `HamNumber` is data attached at the `Stamped` transition. Expose `order.MarkStamped(hamNumber)` so invariants are enforced (e.g. can't stamp a cancelled order) instead of using anonymous setters.
- **An order is a basket:** `Order` → `OrderItem`s. A customer can order two hams plus ribs. If each physical ham is stamped, `HamNumber` lives on the **item**, not the order header.
- **Guests and users carry the same contact data:** `Order` always has contact fields (or a `CustomerContact` value object) plus an **optional** `UserId`. An account is a convenience, never a requirement. Both entry paths converge on one `CreateBooking` command.
- **Catalog (v1):** products, sizes, and preparation options (*rimmat, kokt, osaltat, normal saltad, rå*) seeded directly for the one company. Make it configurable later, not now.
- - **Pickup day is first-class:** `Order.PickupDate` — store as a *date* in
  Europe/Stockholm, not a UTC timestamp, or you'll ship off-by-one-day bugs.
  If a day has max capacity, enforce it **in the database** (transaction +
  constraint/atomic check), never only in C# — two simultaneous bookings must
  not both squeeze into the last slot. Same lesson as the Inkpact
  race-condition review.
- **Booking window + cutoffs are config, not code** — season open/close and
  per-day cutoff live in appsettings/env so a date change never needs a deploy.

---

## Tech stack — free, MIT/Apache only, no license keys

| Concern | Choice | License |
|---|---|---|
| Architecture | Clean Architecture (Domain / Application / Infrastructure / Api) | pattern — free |
| CQRS / pipeline | `Mediator` (martinothamar, source-generated) — **or** plain handler/service classes | MIT |
| Validation | FluentValidation (as a pipeline behavior) | Apache 2.0 |
| Data | EF Core + Npgsql + PostgreSQL | free |
| Auth | ASP.NET Core Identity + JWT | free |
| Tests | xUnit + NSubstitute + Shouldly | free |
| Email | a transactional provider (Postmark / Brevo / Resend), async & non-blocking | free tier |
| Deploy | Railway (API + PostgreSQL) + Vercel (frontend, React + Vite) | free tier |

**Principle: prefer MIT/Apache dependencies with no license keys.**
- **Avoid MediatR v13+** (free under $5M revenue but requires registering a license key) — the martinothamar `Mediator` library is the no-strings equivalent. MediatR v12 and earlier stay free under MIT/Apache if you ever want them.
- **Do not use Fluent*Assertions*** (different library — went paid at v8, $130/dev for commercial use). Shouldly covers the same need for free.
- For a project this small, plain handler/service classes are perfectly clean too — the mediator mostly earns its keep once you have many cross-cutting pipeline behaviors.

---

## Security & cost safety

Two separate concerns. Almost everything here is free and built into ASP.NET Core or the platform — it's discipline and config, not a budget line.

### Stop the runaway bill
- **Hard spending cap on the host (do this first).** Set a usage ceiling/budget alert on Railway. Worst case becomes "the app pauses until I look" — never a surprise invoice. This is the actual circuit breaker.
- **Rate limiting** — built into ASP.NET Core (`AddRateLimiter`). Tight limits on the public booking endpoint.
- **Request size / body limits.**
- **Email async + rate-limited** so a burst can't blow a paid quota.
- *(Grow into it)* Cloudflare free tier in front (DDoS + bot filtering + hides origin); a CAPTCHA (Cloudflare Turnstile / hCaptcha) on the booking form.

### Fail closed — don't leak data
- **Authorize by default.** `[Authorize]` globally; explicitly opt *out* for the few public endpoints. A forgotten attribute then locks a door, not opens one.
- **Object-level checks (IDOR).** `GET /orders/123` must verify the order belongs to the caller. Never trust an ID just because it's in the URL.
- **Secrets in environment variables**, never in the repo — especially the JWT signing key and DB connection string.
- **HTTPS everywhere** (free on Railway/Vercel).
- **Let ASP.NET Core Identity hash passwords** (PBKDF2) — never roll your own.
- **Don't log request bodies containing PII.**
- **GDPR hygiene:** collect only what the order needs, keep a short privacy notice, allow deletion, and set a retention policy (anonymize/delete old seasons rather than hoarding). The data already exists on paper today and the slaughterhouse is the data controller — confirm formal obligations with them; this list is engineering hygiene, not legal advice.
- *(Grow into it, when multi-tenant)* A global EF Core query filter on `TenantId` so you physically can't forget to scope a query.
- - **Double-submit protection** on Confirm — disable the button on click and
  make `CreateBooking` idempotent (client-generated booking key), so a nervous
  double-tap can't create two orders.
- **Honeypot field** on the public form — a hidden input bots fill and humans
  don't. Free, zero-friction spam filter; add Turnstile only if real abuse
  actually shows up.

---
## Backups & disaster recovery

The data *is* the business. Managed Postgres won't lose it by accident — *you*
might (bad migration, wrong environment), and accounts get paused over billing.
Backups make every failure mode boring.

- **Nightly `pg_dump`** via a scheduled GitHub Action during the season (twice
  daily in the December peak), stored off-platform (private repo / object
  storage). A full season is well under 1 MB — keep every dump forever.
- **Manual dump before every production migration or deploy.** Ten seconds of
  insurance.
- **Test one restore** to a scratch database before the season opens. An
  untested backup is a hope, not a backup.
- **Worst-case budget:** max 24 h of bookings lost (12 h in peak). Put that
  number in the contract — it's a guarantee you can actually keep.
- **After each season:** export the full dataset to the customer (CSV + SQL
  dump). Doubles as the GDPR retention step — hand over, then anonymize/trim
  the live DB.
  
---

## Out of scope for v1

- AI (no real use case here).
- Multi-tenancy / catalog-management UI / custom domains.
- Stats subsystem.
- Payment integration (handled on-site via Swish).

---

## Ops & monitoring

- **`/health` endpoint** + a free uptime monitor (UptimeRobot, 5-min checks)
  during the season. December downtime should ping *you*, not be discovered by
  the customer at the counter.
- **Email deliverability:** set up SPF/DKIM on the sending domain on day one or
  confirmations land in spam. Size the provider's free tier against the *peak
  day* (~150 orders), not the season average — Brevo (~300/day) fits; Resend
  (100/day) may not.
- **Off-season:** scale Railway down or pause Jan–Sep; keep the final dump +
  deploy pipeline so October spin-up is a one-evening job.

---

## Business & pricing

A **seasonal service**, billed as an enskild firma, invoiced **in October** before the season.

- **Year one: ~40,000 kr upfront** — covers building the system, loading in products, training staff, and going live.
- **Each season after: ~15,000 kr** — hosting, support through the Oct–Dec crunch, and keeping it running and improving.

**Pitch framing:** separate the *why* of each number — *"the first year builds your system; after that it's a seasonal fee to host it, support you through the busy period, and keep it running."* The 40k buys something real; the 15k just keeps the lights on.

**Why it's justified:** customer self-service replaces staff taking 1,800 orders by phone/counter and hunting a paper binder — plausibly 40–60+ staff hours saved per season, plus fewer errors. At a loaded staff cost of a few hundred kr/hour, the tool pays for itself in saved labour before counting the better customer experience.

**Scaling note:** per-customer recurring revenue is *meant* to be modest — the value is the upfront build fee, the deployed-product reference for the Hogia LIA, and multiplication across customers later (same codebase, marginal support). Don't judge the venture by one customer's seasonal slice.

*(Tax/pricing figures are general — confirm the firma specifics with Skatteverket.)*

---

## Suggested build order

1. Core domain + EF Core schema (Order, OrderItem, status enum, Product/size/option, contact + optional UserId).
2. `CreateBooking` command + public booking flow (guest path first).
3. Staff dashboard: View Orders + search, Add Order (reuses CreateBooking).
4. Ham-number stamping + status transitions + unhandled view.
5. Login + the mid-flow booking-transfer behaviour.
6. Email confirmation + order-reference lookup.
7. Staff edit / cancel.
8. Hardening pass: spending cap, rate limiting, authorize-by-default audit, secrets check.
9. *(later)* Stats, multi-tenancy, catalog-management UI.

---
## Pricing

Offer three packages and let them choose. Invoiced in October; in options 2–3
the ~1,000 kr/season of hosting + domain is yours to carry (it's noise).

| | Year 1 | 3-yr total | Code owner | Your recurring |
|---|---|---|---|---|
| **1. Buy outright** | 60,000 | 60,000 | Them | None — support at 800 kr/h |
| **2. Rent (product + service)** | 20,000/season | 60,000 | **You** | 20,000/season, ongoing |
| **3. Buy + 3-yr service** | 40,000 + 12,000/season | 76,000 | Them | 12,000 × 3 seasons |

- **1** is mostly an anchor — and the worst fit for a customer with no
  in-house tech. Handover day + docs included; everything after is hourly.
- **2** is the SaaS path: you keep the code and sell the same system to the
  next slaughterhouse. Require a 2-season minimum the first time, plus an
  index clause (+3–5 %/yr).
- **3** maximises this deal: paid build, three predictable seasons, they own
  the system. The 12k (vs 15k) is the commitment discount. Renegotiate or do
  an option-1-style handover after year 3.

**Contract, regardless of option:** define support (response time; what's
included vs billed at 800 kr/h), the seasonal data export to the customer, the
backup guarantee (max 24 h data loss), and who owns the domain name.
