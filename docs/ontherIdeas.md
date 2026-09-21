# Dawr (دور) — Self-Service Booking Platform

> **Status:** Planning / early development. This README describes the design and goals of the project. "Dawr" is a working title; it means "turn" in Arabic.

Dawr is a booking platform where any small appointment-based business (barbershops, salons, clinics, tutors, repair shops) can sign up on its own, set up its services and staff, and get a public booking page in minutes. Customers find the business, choose a service and a time, and book. No developer has to build anything per business.

## The problem

Small businesses in many places still take appointments through phone calls and chat messages. That causes double bookings, forgotten appointments, no-shows, and time lost answering the same questions. Existing booking tools are often built for other markets and don't support Arabic and right-to-left layout well.

## How it works

The platform is **one website and one backend shared by all businesses**. A business page is not a separate file that gets deployed. It is data in a database, displayed by one page template.

1. **An owner signs up** and goes through a first-time setup wizard: business name, page address, working hours, services (name, duration, price), and staff.
2. **The owner publishes.** The business page goes live immediately at an address like `/b/ahmad-barbershop`.
3. **Customers open the page**, choose a service, a staff member (or "anyone"), and a free time slot, and confirm the booking.
4. **The owner and staff see the booking** in their calendars and manage it from there.
5. **Reminders** are sent before the appointment.

## Roles

| Role | How they join | What they can do |
|---|---|---|
| **Owner** | Signs up on their own | Manages their own business: services, staff, working hours, page details. Sees all calendars and a dashboard. |
| **Staff** | Invited by the owner by email | Sees their own calendar, marks appointments as done or no-show, requests time off. |
| **Customer** | Signs up when booking | Books, cancels, and reschedules. Sees only their own bookings. |

## Features (v1)

- Setup wizard for new businesses, with draft and published states
- Public business page generated from the owner's data
- Availability engine: free slots are calculated from working hours, breaks, time off, and existing bookings
- Protection against double booking, even when two customers pick the same slot at the same moment
- Booking payment: a 15% deposit is taken when a customer books (**test/sandbox mode only**, no real money)
- Cancel and reschedule rules (for example, no changes within 2 hours of the appointment)
- Staff invitations with expiring, single-use links
- Reminders before appointments, never sent twice
- Owner dashboard: bookings per week, busiest hours, no-show rate, top services
- Arabic and English, with correct right-to-left layout
- Business directory with search and filters by name, category, and city

### Planned AI features (small and optional)

- **Setup helper:** the owner describes the business in one sentence and the AI drafts the page description in Arabic and English. The owner edits it before publishing.
- **Weekly summary:** the AI writes a short summary from numbers the system has already calculated. If the AI is unavailable, the dashboard still works.

The AI only writes wording. All scheduling, availability, and payment logic is deterministic code.

## Booking and payment flow

A booking moves through these states:

```
pending_payment ──(payment confirmed)──> confirmed ──> completed
       │                                     │
       │ (not paid in ~10 minutes)           ├──> cancelled
       ▼                                     └──> no_show
    expired
```

1. The customer picks a slot and the booking is created as `pending_payment`.
2. The slot is **held for about 10 minutes** so nobody else can take it.
3. The customer pays the deposit at the payment provider.
4. The provider sends a **webhook** to the API. The signature is verified, and duplicate webhooks are handled only once.
5. The booking becomes `confirmed` and appears in the staff calendar.
6. If payment doesn't arrive in time, a scheduled job releases the slot.

## Architecture

```
Browser
   │
   ▼
Static hosting ── React app (one template for every business)
   │  API calls
   ▼
Laravel API ── MySQL (data), Redis (queues, cache)
   │              ▲
   │              │ queue worker + scheduler (reminders, slot expiry)
   ▲
   │  webhooks
Payment provider (test mode)
```

- **Frontend:** React, deployed as static files. The page template reads the business address from the URL, asks the API for that business's data, and renders it.
- **Backend:** Laravel REST API with token-based authentication and role-based authorization.
- **Database:** MySQL. Every business-owned table has a `business_id` column, and every query is filtered by it.
- **Async work:** Laravel queues and the scheduler handle reminders, slot expiry, and AI requests.

## Tech stack

| Layer | Technology |
|---|---|
| Frontend | React, a server-state library (TanStack Query), i18n with RTL support |
| Backend | PHP, Laravel (Sanctum for tokens, Policies for authorization) |
| Database | MySQL |
| Queue and cache | Redis |
| AI | LLM API for text generation only |
| Payments | Payment provider in sandbox mode |
| DevOps | Docker Compose, GitHub Actions (tests on every push) |

## Key engineering challenges

- **Double-booking prevention:** appointments have different lengths and can overlap, so a simple unique index is not enough. The design uses database transactions and locking, verified by a concurrency test.
- **Availability calculation:** working hours, minus breaks, time off, and existing bookings, including edge cases such as appointments that would end after closing time.
- **Multi-business isolation:** one business must never see another's data, even by changing an ID in a request. This is covered by dedicated tests.
- **Idempotent background jobs:** reminders and payment webhooks must not run their effect twice, even if a job or request is repeated.
- **Slot holds and expiry:** unpaid bookings release their slots automatically.
- **Invitation security:** tokens expire and can be used only once.

## Data model (overview)

`businesses`, `users` (role, `business_id`), `services`, `staff_services`, `working_hours`, `time_off`, `appointments`, `invitations`, `payments`, `notification_log`.

## Testing

- Feature tests for the booking API, including a concurrent booking test
- Permission tests for every role
- Business isolation tests
- Unit tests for the availability engine and its edge cases
- Continuous integration with GitHub Actions

## Out of scope for v1

- Real payments and payouts, and platform commissions
- Custom domains per business
- Multiple branches per business
- Native mobile app
- Recommendation feeds

## Roadmap

1. Schema, authentication, roles, Docker, CI
2. Owner signup, setup wizard, services, staff, working hours
3. Availability engine and booking API
4. React booking flow, business pages, calendars, Arabic and English
5. Invitations, reminders, payment flow, dashboard, AI features
6. Deployment, demo data, documentation

## Demo

The demo uses seeded fake businesses (a barbershop, a clinic, and a tutor) to show that one system serves different businesses and that their data stays separate. A live link and screenshots will be added here.

## Author

Salah Qaraeen, backend and full-stack developer.
GitHub: [github.com/salahqr](https://github.com/salahqr)

## License

To be decided.