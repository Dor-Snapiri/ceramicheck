# Ceramicheck

Ceramicheck is a web app for a small pottery studio: students sign up to studio sessions, and the admin (studio owner) tracks bookings, payments, income, expenses and receipts. Low traffic: one admin and a few dozen students.

## Working with me

- I know JS, React and web basics, with a little backend experience. This project is also how I'm learning to build a full web app.
- **Explain as you go:** say briefly why you chose an approach, and name new concepts (server actions, RLS, migrations, etc.) the first time they appear.
- Work in small steps I can review. Don't build features beyond the current stage without asking.
- I'm on Windows and use **Command Prompt (cmd)** in the VS Code terminal, not PowerShell. Give commands that work in cmd.
- Don't run `npm audit fix --force`, and don't upgrade npm or major package versions unless I ask.

## Stack

- Next.js (App Router) + TypeScript, with code in `src/` and imports via `@/*`
- Tailwind CSS
- Supabase (planned): Postgres, Auth, Storage for receipt files
- Hosting on Vercel (planned); code on GitHub

## Language and locale

- The UI is **Hebrew only, right-to-left**: `<html lang="he" dir="rtl">` in `src/app/layout.tsx`, with the Heebo font.
- Use Tailwind **logical** classes (`ms-*`, `me-*`, `ps-*`, `pe-*`, `start-*`, `end-*`, `text-start`), never `ml/mr/pl/pr/left/right`.
- Code, file names, database tables and columns are in **English**; user-facing text is in Hebrew.
- Times use the Asia/Jerusalem time zone. Store timestamps as `timestamptz`, format dates with `Intl` and `he-IL`, and weeks start on Sunday.

## Architecture

The app is organized by feature so new features plug in without touching existing ones:

```
src/
  app/                  routes only (thin): pages, layouts
    admin/              admin pages (protected)
  features/<feature>/   components, server actions, queries, types per feature
                        e.g. sessions/, bookings/, slots/, payments/, finances/
  lib/supabase/         Supabase client helpers (browser + server)
  components/ui/        shared generic UI components
supabase/migrations/    SQL migrations (every schema change goes here)
```

- All schema changes are SQL migrations. Never edit the database by hand without a migration.
- Business rules (capacity, uniqueness) are enforced **in the database** (constraints, RLS, transactions), not only in the UI.

## Users and auth

- **Students** register freely with email and password, plus email verification. Their profile holds full name and phone.
- **Admin:** a single account created manually in Supabase; there is no admin sign-up. Two-factor login is planned for later.
- Roles live in `profiles.role` (`student` | `admin`).
- Admin pages are protected on the server (role check) **and** by row-level security. Students can only read and write their own data.

## Domain model

| Table | Purpose |
|---|---|
| `profiles` | id (= auth user), full_name, phone, role |
| `weekly_slots` | recurring opening times: day_of_week, start_time, end_time, capacity (default 4), active. **Admin-editable, never hard-coded.** Initially Monday and Thursday 17:00–19:30 |
| `slot_memberships` | student ↔ weekly slot, with start_date and end_date (null = ongoing) |
| `sessions` | a concrete opening: date, start/end time, capacity, slot_id (null = one-time session), status (`open` \| `cancelled`) |
| `bookings` | student ↔ session: status (`booked` \| `cancelled` \| `cancelled_by_studio`), source (`default` \| `moved` \| `manual`) |
| `settings` | app settings, e.g. `cancel_cutoff_hours` (currently 0) |
| `payments` *(stage 3)* | separate table, linked to bookings with an indexed `booking_id`. Students pay per session |
| `transactions`, `receipts` *(stage 4)* | income and expenses, with uploaded receipt files in Supabase Storage |

### Rules

- Sessions and default bookings are **created ahead of time** (a few weeks) by a scheduled job, from weekly slots and memberships.
- A weekly slot holds at most `capacity` (4) active members; a session holds at most `capacity` (4) booked students.
- A student can have any number of active memberships, but only one per slot, and only one booking per session.
- Students can join or leave slots, cancel single sessions, and book any session with a free spot, without limits. Cancellation is allowed any time (`cancel_cutoff_hours = 0`).
- Joining a slot creates bookings for its upcoming sessions, skipping weeks that are already full. Leaving a slot cancels only *future* default bookings.
- A member who cancelled a week can rebook it only if the spot is still free.
- The admin can open one-time sessions, cancel a single session (its bookings become `cancelled_by_studio`), and add, edit or deactivate weekly slots. Slots are never deleted. Editing a slot's time applies to its sessions from a chosen date; past sessions keep their original time.
- Holidays are handled by the admin cancelling sessions by hand.

## Roadmap

1. **Stage 1 – Foundation (in progress):**
   - [x] Create Next.js project (create-next-app defaults)
   - [x] Hebrew/RTL layout and Heebo font
   - [x] Push to GitHub
   - [ ] Supabase project and client helpers
   - [ ] Auth with student/admin roles
   - [ ] Schema migrations
   - [ ] Deploy to Vercel
2. **Stage 2 – Sessions and bookings:** admin manages slots and sessions; students join slots and book or cancel.
3. **Stage 3 – Payments:** record payments per booking, a view of who owes what, and a student balance.
4. **Stage 4 – Finances:** income and expenses, receipt uploads, monthly summaries, CSV/Excel export for the accountant.
5. **Stage 5 – Extras:** reminders, waitlists, punch cards, admin two-factor login, backups.

## Commands

```
npm run dev     # dev server at http://localhost:3000
npm run build   # production build
npm run lint    # ESLint
```