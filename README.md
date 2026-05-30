# Christmas Ham Booking System

> Seasonal booking system for a Swedish slaughterhouse's Christmas products (julskinka, revben, kalkon).
> Working title — final brand/domain name still open (e.g. `bokningar.se`, `slakteribokning.se`). Avoid a ham-only name so it survives expansion.

---

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
3. Special requests.
4. Save the booking and carry it forward.

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

---

## Domain model

- **Order status lifecycle:** explicit `OrderStatus` — **Booked → Stamped → Collected**, plus **Cancelled**. Status is the workflow; `HamNumber` is data attached at the `Stamped` transition. Expose `order.MarkStamped(hamNumber)` so invariants are enforced (e.g. can't stamp a cancelled order) instead of using anonymous setters.
- **An order is a basket:** `Order` → `OrderItem`s. A customer can order two hams plus ribs. If each physical ham is stamped, `HamNumber` lives on the **item**, not the order header.
- **Guests and users carry the same contact data:** `Order` always has contact fields (or a `CustomerContact` value object) plus an **optional** `UserId`. An account is a convenience, never a requirement. Both entry paths converge on one `CreateBooking` command.
- **Catalog (v1):** products, sizes, and preparation options (*rimmat, kokt, osaltat, normal saltad, rå*) seeded directly for the one company. Make it configurable later, not now.

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

---

## Out of scope for v1

- AI (no real use case here).
- Multi-tenancy / catalog-management UI / custom domains.
- Stats subsystem.
- Payment integration (handled on-site via Swish).

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
