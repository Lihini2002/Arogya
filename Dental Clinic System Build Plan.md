# Dental Clinic System: Build Plan

Draft plan, 23 October 2026. Goal: a web system the clinic can rely on daily, with staff login, patient records, appointment booking, and automated SMS reminders. Quality over speed.

## 1. Scope

**In scope (core build)**

- Staff login with two roles (staff, admin)
- Patient records: create, edit, search
- Appointments: book, cancel, complete, mark no-show, calendar view per dentist
- Double-booking prevented at the database level
- SMS reminder 24 hours before each appointment
- Deployed on a live URL, works on phones, tablets, and desktops

**Out of scope for now**

- Billing, treatment history, X-ray and file uploads, multi-branch support, reports

## 2. Tech stack

| Layer | Choice | Why |
| --- | --- | --- |
| App framework | Next.js (App Router) with TypeScript | One codebase for UI and server code, responsive by default |
| UI | Tailwind CSS and shadcn/ui | Accessible components without designing from scratch |
| Calendar | FullCalendar | Day and week views, click-to-book |
| Database | PostgreSQL (Prisma Postgres, already created) | Relational data fits patients and appointments |
| Data access | Prisma ORM | Typed queries, versioned migrations in the repo |
| Validation | zod and react-hook-form | One schema for forms and server checks |
| Phone handling | libphonenumber-js | Normalize every number to E.164 before saving |
| Auth | Auth.js or Better Auth, users stored via Prisma | Keeps data and auth in one place; no hand-rolled password handling |
| SMS | Twilio, or a local gateway if cheaper and more reliable | Decide after checking sender-ID rules |
| Scheduling | Vercel Cron calling a protected route handler | Reminder logic lives in the codebase, where it can be tested |
| Hosting | Vercel for the app | Simple deploys and cron |
| Testing | Vitest (logic), Playwright (booking flow) | Catches booking bugs early |
| Monitoring | Sentry | Know about errors before the clinic does |

**Fallback:** if the Prisma Postgres database rejects `CREATE EXTENSION btree_gist`, host the database on Supabase or Neon instead. Prisma ORM and everything else stay the same; only the connection string changes.

## 3. Key design decisions

- **Double-booking is blocked by an exclusion constraint** in Postgres (needs the `btree_gist` extension), not just by UI checks. Two simultaneous bookings cannot both succeed.
- **All times stored in UTC** (`timestamptz`) and displayed in one clinic timezone constant.
- **Appointments are never hard-deleted.** Status changes instead: scheduled, completed, cancelled, no-show.
- **Authorization is checked on the server** in a single helper (`requireRole`) at the start of every action. The client is never trusted.
- **Two connection strings:** `DATABASE_URL` (pooled, for the running app) and `DIRECT_URL` (direct, for migrations).
- **Patient phone numbers are stored in E.164 format** and an `smsOptOut` flag is respected by the reminder job.

## 4. To-do list

### Before coding

- [ ] Run `CREATE EXTENSION IF NOT EXISTS btree_gist;` on the database and confirm it succeeds
- [ ] Confirm which country and timezone the clinic is in
- [ ] Pick the SMS provider and start sender-ID registration (this can take days)
- [ ] Check local data-protection rules for patient data
- [ ] Create the Git repository and decide on a branch workflow

### Week 1: core build

- [ ] Project scaffold, environment variables, Prisma connected
- [ ] Authentication, route protection, roles
- [ ] Database schema and migration, including the double-booking constraint
- [ ] Patient management (create, edit, search)
- [ ] Appointment booking, cancel, complete, no-show
- [ ] Calendar view with dentist filter
- [ ] Timezone handling, seed data, deployment to a live URL

### Week 2: reminders and reliability

- [ ] SMS provider integration
- [ ] Cron job that selects appointments due for a reminder
- [ ] Retry handling and delivery status tracking
- [ ] Opt-out handling
- [ ] Reschedule flow

### Week 3: security and data protection

- [ ] Finalize role-based permissions
- [ ] Audit log of who viewed or changed records
- [ ] Backups with a tested restore
- [ ] Input validation review and rate limiting
- [ ] Review of what patient data is stored and who can see it

### Week 4: real-world testing

- [ ] Sit with the receptionist and dentist and watch them use it
- [ ] Fix usability issues found
- [ ] Add Sentry monitoring
- [ ] Write a short staff guide
- [ ] Plan migration of any existing patient records

### Buffer

- [ ] About one week reserved for SMS approval delays, device-specific bugs, and change requests

## 5. Week 1: day-by-day tasks

### Day 1: Setup and authentication

- [ ] Create the Next.js app: `npx create-next-app@latest clinic --typescript --tailwind --app`
- [ ] Install dependencies: `@prisma/client`, `zod`, `react-hook-form`, `libphonenumber-js`, plus the chosen auth library; dev dependency `prisma`
- [ ] Initialize shadcn/ui
- [ ] Put `DATABASE_URL` and `DIRECT_URL` in `.env` and add `.env` to `.gitignore`
- [ ] Set up the auth library with email and password, with public sign-up disabled
- [ ] Build the `/login` page
- [ ] Add middleware that redirects unauthenticated users to `/login`
- [ ] Write the `requireRole` server helper
- [ ] Commit

### Day 2: Database schema

- [ ] Write the Prisma models: Profile (or User), Dentist, Patient, Appointment, with enums for role and appointment status
- [ ] Run `npx prisma migrate dev --name init --create-only` so the migration is not yet applied
- [ ] Append to the migration SQL: `CREATE EXTENSION IF NOT EXISTS btree_gist`, a check that `endsAt > startsAt`, and the exclusion constraint on dentist and time range for scheduled appointments
- [ ] Apply the migration with `npx prisma migrate dev`
- [ ] Write a seed script with fake patients and dentists
- [ ] Manually test: insert two overlapping appointments for one dentist and confirm the second is rejected
- [ ] Commit, with the schema and migrations in the repo

### Day 3: Patients

- [ ] `/patients` page with a search box (name or phone) and a table
- [ ] `/patients/new` form with zod validation
- [ ] `/patients/[id]/edit` form
- [ ] Normalize and validate phone numbers on save; reject anything that does not parse
- [ ] Patient detail page with upcoming and past appointments (empty at first)
- [ ] Server actions protected with `requireRole`
- [ ] Write unit tests for phone normalization
- [ ] Commit

### Day 4: Appointments and calendar

- [ ] Install FullCalendar (react, daygrid, timegrid, interaction)
- [ ] Day and week views with a dentist filter
- [ ] Booking dialog: searchable patient picker, dentist, duration, reason
- [ ] Booking server action that catches Postgres error code `23P01` and returns "That time is already booked"
- [ ] Click an appointment to cancel, complete, or mark no-show (status change, never delete)
- [ ] Write tests for the booking action, including the overlap case
- [ ] Commit

### Day 5: Timezones, roles, deployment

- [ ] Define one clinic timezone constant and use it for all display and input
- [ ] Check boundary cases: appointments near midnight and around daylight-saving changes (if the timezone has them)
- [ ] Apply roles: staff manage patients and appointments; admin also manages users and dentists
- [ ] Make layouts usable on a phone-sized screen
- [ ] Deploy to Vercel and set environment variables
- [ ] Test login and booking from a real phone
- [ ] Commit and tag the end of week 1

### Week 1 done when

- [ ] No page is reachable without logging in
- [ ] Patients can be created, edited, and searched
- [ ] Appointments can be booked, cancelled, completed, and marked no-show
- [ ] Overlapping bookings for one dentist fail cleanly with a friendly message
- [ ] The app works on a phone and is live on a URL
- [ ] Schema, migrations, and seed script are all in the repo

## 6. Risks to watch

- **SMS approval delays.** Start sender-ID registration before writing SMS code.
- **Real patient data too early.** Use fake data until security work in week 3 is done.
- **Timezone bugs.** Store UTC, display local, and test around midnight.
- **Scope creep.** New feature requests go on a list for after week 4, not into the current week.
- **Lost database access.** Keep connection strings out of the repository, and test a backup restore before go-live.
