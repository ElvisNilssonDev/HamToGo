# Christmas Ham Booking System

> Working title — final brand/domain name still undecided (e.g. `bokningar.se`, `slakteribokning.se`). Avoid locking into a ham-only name since the catalog will expand.

A seasonal booking system for a Swedish slaughterhouse's Christmas products (julskinka, revben, kalkon). Replaces a paper-binder workflow — where finding a single order takes 3–5 minutes — with a searchable digital system handling 1,800+ orders per season. Active October 1 through December 24, and designed from the start to scale to other slaughterhouses.

---

## Users

- **Guest** — can view the homepage and place a booking without an account.
- **Registered customer** — same booking flow, but the order is tied to their account.
- **Staff / admin** — signed-in dashboard for managing all orders.

---

## Customer-facing

The public side is built around one clear, modern flow.

**Homepage (guest-accessible)**
- App name at the top.
- Headline: *"Book your Christmas ham."*
- A single large **Start here** button.

**Booking flow** — a guided, interactive UI:
1. Add ham (size + preparation type).
2. Add extra options / additional products.
3. Special requests window.
4. Save the booking and carry it forward.

**User information**
- Either fill in contact details — **First name, Last name, Email, Phone number** — or log in.
- If the customer clicks **log in mid-flow, the in-progress booking is NOT lost.** It transfers to the confirm page once they're authenticated.

**Confirm & confirmation**
- Click **Confirm order**.
- Receive a confirmation, including an **email confirmation**.
- *(added)* The confirmation includes an **order reference** so a guest with no account can look their booking up later (reference + email).

---

## Admin / business-facing

The signed-in front page is a dashboard made of blocks.

**Primary blocks**
- **View Orders** — all handled orders, searchable.
- **Add Order** — staff can enter an order on behalf of a phone or walk-in customer. *(This reuses the same booking logic as the public flow — built once, used by both.)*

**Search** inside View Orders.

**Ham numbers**
- The slaughterhouse stamps a physical number on each finished ham.
- A `HamNumber` is recorded on the order **after creation** — the customer books first, staff assign the number once the ham is done.

**Unhandled vs handled**
- An **Unhandled orders** view shows what's still outstanding (newly booked, not yet stamped).
- *(added / refined)* This is driven by an explicit **order status**, not by whether `HamNumber` is empty — see Domain Model below.

*(added)* **Staff edit / cancel** — staff can correct a typo'd phone number, change a size, or cancel an order, not just view and stamp.

**Stats** *(deferred — nice to have, not v1)*
- Count of hams / orders per day, week, and month.

---

## Domain model

These refinements keep the data clean as the system grows. *(Mostly additions — the original ideas were sound, this is how they wire together.)*

**Order status lifecycle** *(added)*
- Model an explicit `OrderStatus`: **Booked → Stamped → Collected**, plus **Cancelled**.
- Status is the *workflow*; `HamNumber` is *data* attached at the `Stamped` transition.
- Don't infer "handled" from `HamNumber != null` — that breaks on cancellations, no-shows, and picked-up orders.
- Expose a domain method like `order.MarkStamped(hamNumber)` that enforces invariants (e.g. you can't stamp a cancelled order) instead of an anonymous setter.

**An order is a basket, not a single ham** *(added)*
- `Order` → `OrderItem`s. A customer can order two hams plus ribs.
- If each physical ham is stamped, `HamNumber` lives on the **item**, not the order header.

**Guests and users carry the same contact data** *(added)*
- A guest still provides Name / Email / Phone — identical to a logged-in user.
- `Order` always has contact fields (or a small `CustomerContact` value object) plus an **optional** `UserId` foreign key.
- An account is a convenience, never a requirement. Both entry paths converge on one `CreateBooking` command.

**Configurable, tenant-scoped catalog**
- Each company defines its own products — different ham sizes, ribs, turkey, and extra options.
- Model: `Product` → variants (size) + options (preparation: *rimmat, kokt, osaltat, normal saltad, rå*).
- Everything is scoped to a `Tenant`.

**Operational fields** *(added)*
- **Pickup date / slots** — Christmas hams are collected on specific days; the company needs to know who's coming when.
- **Season cutoff / bookings-open toggle** — close orders at a point in the season and (optionally) cap volume, so a stray January order doesn't land in an empty kitchen.

---

## Multi-tenancy & scaling

- Design the schema **tenant-scoped and data-driven from day one** — `TenantId` on products, orders, and queries.
- **Do not build the catalog-management admin UI in v1.** Seed the first company's catalog directly and refactor management screens in once a customer is live.
- For v1, **path-based tenant resolution** on a single domain (`yoursite.se/almos`, `yoursite.se/liljevikens`). Subdomains are a year-2 option; per-tenant custom domains are a year-3 premium feature.

---

## Explicitly out of scope for v1

- **AI** — no real AI-shaped problem here; booking a ham doesn't need it. (The AI showcase lives in a separate project, ReceiptScout.)
- **Stats subsystem** — it's a `GROUP BY date` query to write later, not infrastructure to design up front.
- **Catalog-management UI** — seed first, build later.
- **Custom domains per tenant** — year 3.
- **Payment integration** — payment is handled on-site at pickup (Swish).

---

## Proposed tech stack

*Based on the usual stack — adjust as needed.*

- **Backend:** C# / .NET 10, ASP.NET Core, Clean Architecture, CQRS with MediatR, EF Core.
- **Auth:** ASP.NET Core Identity + JWT.
- **Database:** PostgreSQL.
- **Frontend:** React + Vite.
- **Email:** a transactional provider (e.g. Postmark / Brevo / Resend), with sends made **async and non-blocking** so a failed email never fails the order.
- **Deployment:** Railway (API + PostgreSQL) + Vercel (frontend).

---

## Suggested build order

1. Core domain + EF Core schema (Order, OrderItem, status enum, Product/variant/option, Tenant, contact + optional UserId).
2. `CreateBooking` command + the public booking flow (guest path first).
3. Staff dashboard: View Orders + search, Add Order (reuses CreateBooking).
4. Ham-number stamping + status transitions + unhandled view.
5. Login + the mid-flow booking-transfer behaviour.
6. Email confirmation + order-reference lookup.
7. Staff edit / cancel.
8. *(later)* Stats, catalog-management UI, multi-tenant polish.
