# Valet Platform — Prompt Pack

## Prompt 1

Build the complete database schema and authentication system for a multi-sided valet marketplace platform. This is a greenfield Lovable project connected to Supabase.

**What to build:**

Create the full Supabase database schema with the following tables. Enable Row Level Security on every table.

**Users table** (`users`):
- `id` (uuid, primary key, default gen_random_uuid())
- `role` (text, not null) — one of: `customer`, `driver`, `partner`, `admin`
- `email` (text, unique, not null)
- `phone` (text)
- `full_name` (text)
- `device_token` (text) — for push notifications
- `created_at` (timestamptz, default now())
- `updated_at` (timestamptz, default now())
- `deleted_at` (timestamptz) — soft delete for CCPA
- `pii_nullified_at` (timestamptz) — marks when PII was wiped on deletion request

**Drivers table** (`drivers`):
- `id` (uuid, primary key, default gen_random_uuid())
- `user_id` (uuid, FK to users.id, not null)
- `status` (text, not null, default `pending`) — one of: `pending`, `approved`, `suspended`
- `background_check_status` (text, default `not_submitted`) — one of: `not_submitted`, `submitted`, `passed`, `failed`, `manual_override`
- `background_check_provider_ref` (text)
- `onboarding_completed_at` (timestamptz)
- `city` (text) — one of: `new_york`, `los_angeles`, `chicago`
- `created_at` (timestamptz, default now())

**Partners table** (`partners`):
- `id` (uuid, primary key, default gen_random_uuid())
- `user_id` (uuid, FK to users.id, not null)
- `location_name` (text, not null)
- `address` (text, not null)
- `city` (text, not null) — one of: `new_york`, `los_angeles`, `chicago`
- `capacity` (integer)
- `payout_account_ref` (text)
- `created_at` (timestamptz, default now())

**Jobs table** (`jobs`):
- `id` (uuid, primary key, default gen_random_uuid())
- `customer_id` (uuid, FK to users.id, not null)
- `driver_id` (uuid, FK to users.id, nullable)
- `partner_id` (uuid, FK to partners.id, nullable)
- `city` (text, not null) — one of: `new_york`, `los_angeles`, `chicago`
- `pickup_address` (text, not null)
- `dropoff_address` (text)
- `status` (text, not null, default `scheduled`) — enum: `scheduled`, `dispatched`, `en_route`, `arrived`, `in_progress`, `complete`, `cancelled`
- `scheduled_at` (timestamptz, not null)
- `completed_at` (timestamptz)
- `surge_multiplier` (numeric, default 1.0)
- `base_price` (numeric)
- `final_price` (numeric)
- `cancellation_reason` (text)
- `created_at` (timestamptz, default now())
- `updated_at` (timestamptz, default now())

**Bookings table** (`bookings`):
- `id` (uuid, primary key, default gen_random_uuid())
- `job_id` (uuid, FK to jobs.id, not null)
- `created_at` (timestamptz, default now())
- `modified_at` (timestamptz)
- `cancellation_reason` (text)

**Payments table** (`payments`):
- `id` (uuid, primary key, default gen_random_uuid())
- `job_id` (uuid, FK to jobs.id, not null)
- `stripe_payment_intent_id` (text, unique)
- `amount` (numeric, not null)
- `currency` (text, not null, default `usd`)
- `status` (text, not null, default `pending`) — one of: `pending`, `succeeded`, `refunded`, `disputed`, `failed`
- `receipt_url` (text)
- `receipt_pdf_key` (text) — S3 object key for the PDF
- `created_at` (timestamptz, default now())

**Driver location history table** (`driver_location_history`):
- `id` (uuid, primary key, default gen_random_uuid())
- `driver_id` (uuid, FK to users.id, not null)
- `job_id` (uuid, FK to jobs.id, not null)
- `latitude` (numeric, not null)
- `longitude` (numeric, not null)
- `recorded_at` (timestamptz, default now())

Add a comment to this table: `-- Retention policy: purge records where recorded_at < (job completed_at + 30 days)`

**WebSocket connections table** (`websocket_connections`):
- `id` (uuid, primary key, default gen_random_uuid())
- `connection_id` (text, unique, not null)
- `user_id` (uuid, FK to users.id, not null)
- `role` (text, not null)
- `job_id` (uuid, FK to jobs.id, nullable)
- `connected_at` (timestamptz, default now())

**RLS policies to create:**

- `users`: users can read and update their own row; admins can read all rows
- `drivers`: drivers can read their own row; admins can read and update all rows
- `partners`: partners can read their own row; admins can read all rows
- `jobs`: customers can read jobs where `customer_id = auth.uid()`; drivers can read jobs where `driver_id = auth.uid()`; partners can read jobs where `partner_id` matches their partner record; admins can read all
- `payments`: customers can read payments for their own jobs; admins can read all
- `driver_location_history`: drivers can insert their own location records; admins can read all
- `websocket_connections`: users can manage their own connection records; admins can read all

**Authentication:**

Using Supabase Auth:
- Configure email/password sign-up and login
- After a user signs up, create a corresponding row in the `users` table using a Supabase database trigger (on auth.users insert, insert into public.users with the new user's id and email, defaulting role to `customer`)
- Create a Supabase Edge Function called `set-user-role` that allows an admin to update a user's role in the users table. This function must verify the caller is an admin before making any change.
- Create a Supabase Edge Function called `refresh-session` that returns a fresh session token.

**What NOT to build yet:** No UI screens, no booking flows, no payments, no maps, no notifications. Schema and auth only.

Before moving on, confirm the following are working: all eight tables exist in Supabase with the correct columns and RLS enabled; a test user can sign up via Supabase Auth and a corresponding row appears in the `users` table; the `set-user-role` edge function deploys without errors.

---

## Prompt 2

Run this after Prompt 1 is complete.

Build the booking and dispatch system. This is a Lovable project using Supabase as the backend. The database schema from the previous step is already in place.

**What to build:**

**Booking Edge Functions** — create the following Supabase Edge Functions:

`create-booking`:
- Accepts: `customer_id`, `city`, `pickup_address`, `dropoff_address`, `scheduled_at`, `partner_id` (optional)
- Calls the surge pricing logic (below) to calculate `surge_multiplier`
- Calculates `base_price` (use a flat rate for now: $25.00 base fare)
- Sets `final_price = base_price * surge_multiplier`
- Inserts a row into `jobs` with status `scheduled`
- Inserts a corresponding row into `bookings`
- Returns the created job record
- Requires the caller to be authenticated and have the role `customer`

`modify-booking`:
- Accepts: `job_id`, and any of: `scheduled_at`, `pickup_address`, `dropoff_address`
- Only allowed when job status is `scheduled`
- Updates the job and sets `bookings.modified_at` to now()
- Returns the updated job record
- Requires the caller to own the job (customer_id = auth.uid())

`cancel-booking`:
- Accepts: `job_id`, `cancellation_reason`
- Only allowed when job status is `scheduled` or `dispatched`
- Sets job status to `cancelled`, writes the cancellation reason to both `jobs` and `bookings`
- Returns the updated job record
- Requires the caller to own the job or be an admin

`get-customer-bookings`:
- Returns all jobs for the authenticated customer, ordered by `scheduled_at` descending
- Requires the caller to have the role `customer`

**Surge pricing logic** — implement as a shared utility used inside `create-booking`:
- Count the number of active jobs (status in `scheduled`, `dispatched`, `en_route`, `arrived`, `in_progress`) in the same city
- Apply multiplier tiers:
  - 0–10 active jobs: 1.0x
  - 11–25 active jobs: 1.25x
  - 26–50 active jobs: 1.5x
  - 51+ active jobs: 2.0x
- Store the calculated multiplier as `surge_multiplier` on the job record

**Dispatch Edge Functions:**

`get-available-jobs`:
- Returns all jobs with status `scheduled` in the authenticated driver's city
- Requires the caller to have role `driver` and driver status `approved`

`accept-job`:
- Accepts: `job_id`
- Sets the job's `driver_id` to the authenticated driver
- Transitions job status from `scheduled` to `dispatched`
- Fails if the job is already dispatched or not in `scheduled` state
- Requires the caller to have role `driver` and driver status `approved`

`decline-job`:
- Accepts: `job_id`
- Removes the driver from the job (sets `driver_id` to null) and returns the job to `scheduled` status
- Only works if the job is in `dispatched` state and the driver is the assigned driver

`update-job-status`:
- Accepts: `job_id`, `new_status`
- Enforces valid status transitions only:
  - `dispatched` → `en_route`
  - `en_route` → `arrived`
  - `arrived` → `in_progress`
  - `in_progress` → `complete`
- Sets `completed_at` when transitioning to `complete`
- Requires the caller to be the assigned driver

**What NOT to build yet:** No payment processing, no real-time location, no notifications, no UI screens.

Before moving on, confirm the following are working: a test customer can create a booking and it appears in the `jobs` table with the correct surge multiplier; a test driver can retrieve available jobs in their city, accept a job, and transition it through each status step; an invalid status transition is correctly rejected.

---

## Prompt 3

Run this after Prompt 2 is complete.

Build the real-time driver location system using Supabase Realtime. This is a Lovable project using Supabase. The jobs, driver_location_history, and websocket_connections tables are already in place.

**What to build:**

**Location broadcast Edge Functions:**

`broadcast-driver-location`:
- Accepts: `job_id`, `latitude`, `longitude`
- Requires the caller to have role `driver` and be the assigned driver on the job
- Validates the job is in an active status (`dispatched`, `en_route`, `arrived`, `in_progress`)
- Inserts a record into `driver_location_history` with the driver_id, job_id, coordinates, and current timestamp
- Broadcasts the location update to a Supabase Realtime channel named `job:{job_id}` with payload `{ driver_id, latitude, longitude, recorded_at }`
- Returns 200 OK

`register-connection`:
- Accepts: `job_id`, `role`
- Inserts a row into `websocket_connections` for the authenticated user, linked to the job
- Returns the connection record

`deregister-connection`:
- Accepts: `connection_id`
- Deletes the row from `websocket_connections`
- Requires the caller to own the connection record

**Supabase Realtime configuration:**
- Enable Realtime on the `driver_location_history` table in Supabase settings
- Create a Realtime channel pattern: `job:{job_id}` — customers subscribe to this channel to receive location updates for their active job
- Publish location updates to this channel from within `broadcast-driver-location` using Supabase's broadcast feature

**Customer-facing location subscription — React hook:**

Create a React hook called `useDriverLocation(jobId)` that:
- Subscribes to the Supabase Realtime channel `job:{jobId}` when the hook mounts
- Listens for broadcast events on that channel
- Returns the latest `{ latitude, longitude, recorded_at }` payload, updated in real time
- Unsubscribes and cleans up the channel when the component unmounts
- Handles connection drops by automatically resubscribing with exponential backoff (start at 1 second, max 30 seconds)

**Driver-facing location sender — React hook:**

Create a React hook called `useLocationBroadcast(jobId)` that:
- Calls `broadcast-driver-location` with the current GPS coordinates at a 4-second interval when the hook is active
- Uses the browser Geolocation API (`navigator.geolocation.watchPosition`) to get current coordinates
- Stops broadcasting when the hook unmounts or when called with `stop()`
- Logs an error to the console if the broadcast fails, but does not throw — the driver's UI should not crash on a failed location update

**What NOT to build yet:** No map UI rendering, no payment processing, no notifications, no booking UI.

Before moving on, confirm the following are working: calling `broadcast-driver-location` with a valid active job ID inserts a row into `driver_location_history` and emits a Realtime broadcast on the correct channel; a browser tab subscribing to that channel via `useDriverLocation` receives the update within 5 seconds; disconnecting and reconnecting triggers the resubscription logic.

---

## Prompt 4

Run this after Prompt 3 is complete.

Build the Stripe payment integration. This is a Lovable project using Supabase. The jobs and payments tables are already in place, and `VITE_STRIPE_PUBLISHABLE_KEY` and `STRIPE_SECRET_KEY` are already set in the project environment.

**What to build:**

**Payment Edge Functions:**

`create-payment-intent`:
- Accepts: `job_id`
- Requires the caller to have role `customer` and own the job
- Checks that the job status is `complete`
- Reads `final_price` from the job record (in dollars) and converts to cents for Stripe
- Creates a Stripe Payment Intent via the Stripe API using `STRIPE_SECRET_KEY`
- Sets the Payment Intent metadata to include `job_id`
- Inserts a row into `payments` with `stripe_payment_intent_id`, `amount` (in cents), `currency: usd`, `status: pending`, and the `job_id`
- Returns `{ client_secret, payment_id }` — the client_secret is passed to the Stripe SDK on the frontend

`stripe-webhook`:
- Public endpoint (no auth required) that receives POST requests from Stripe
- Verifies the Stripe webhook signature using `STRIPE_WEBHOOK_SECRET` on every request — reject any request that fails verification
- Handles these events idempotently (check if the payment record is already in the target state before updating):
  - `payment_intent.succeeded`: set payments.status to `succeeded`
  - `payment_intent.payment_failed`: set payments.status to `failed`
  - `charge.refunded`: set payments.status to `refunded`
  - `charge.dispute.created`: set payments.status to `disputed`
- For `payment_intent.succeeded`: also generate a receipt (see below) and update `jobs.status` to `complete` if not already set
- Returns 200 OK to Stripe for all handled events; return 200 for unhandled event types too (do not return errors for unknown events)

`get-payment-status`:
- Accepts: `job_id`
- Returns the payment record for that job
- Requires the caller to own the job or be an admin

**Receipt generation** (called from within `stripe-webhook` on success):
- Generate a simple text-based receipt with: job ID, customer name, pickup address, completed_at date, base price, surge multiplier, final price, and payment status
- Store the receipt as a PDF in the S3 bucket named by `S3_RECEIPTS_BUCKET`, with key `receipts/{job_id}.pdf`
- Update the `payments` row with `receipt_pdf_key` and set `receipt_url` to a pre-signed S3 URL (7-day expiry)

**Frontend — Stripe payment UI component:**

Create a React component called `PaymentForm` that:
- Accepts `jobId` as a prop
- On mount, calls `create-payment-intent` to retrieve the `client_secret`
- Renders a Stripe Elements form using `@stripe/react-stripe-js` and `@stripe/stripe-js` with the publishable key from `VITE_STRIPE_PUBLISHABLE_KEY`
- Uses `CardElement` from Stripe Elements for card capture — card data never leaves the browser before being tokenized by Stripe
- On submit, calls `stripe.confirmCardPayment(clientSecret)` with the card element
- Shows a success state when the payment is confirmed
- Shows a clear error message if payment fails, without crashing the component

**What NOT to build yet:** No notifications, no maps UI, no driver onboarding, no admin or partner portal screens.

Before moving on, confirm the following are working: calling `create-payment-intent` for a complete job creates a Payment Intent in Stripe test mode and inserts a pending row in the payments table; simulating a `payment_intent.succeeded` webhook event from the Stripe Dashboard updates the payment status to `succeeded`; the webhook rejects requests with an invalid signature; the `PaymentForm` component renders without errors in the Lovable preview.

---

## Prompt 5

Run this after Prompt 4 is complete.

Build the SMS and push notification system. This is a Lovable project using Supabase. The jobs and users (with device_token field) tables are already in place. `TWILIO_ACCOUNT_SID`, `TWILIO_AUTH_TOKEN`, `TWILIO_PHONE_NUMBER`, `FCM_SERVER_KEY`, `APNS_KEY_ID`, `APNS_TEAM_ID`, and `APNS_BUNDLE_ID` are already set in the project environment.

**What to build:**

**Notification Edge Function — `send-notification`:**

This is a single internal Edge Function called by other functions (not exposed as a public endpoint). It accepts:
- `user_id` — the recipient
- `event_type` — one of: `booking_confirmed`, `driver_dispatched`, `driver_arrived`, `job_complete`
- `job_id` — for context

Logic:
- Look up the user's `phone` and `device_token` from the `users` table
- Send an SMS via Twilio for every event type using these message templates:
  - `booking_confirmed`: "Your valet is booked for [scheduled_at]. We'll notify you when your driver is on the way."
  - `driver_dispatched`: "Your driver is on the way! Track them live in the app."
  - `driver_arrived`: "Your driver has arrived at [pickup_address]."
  - `job_complete`: "Your valet job is complete. Total charged: $[final_price]. Thank you!"
- Send a push notification via FCM if the user has a `device_token` that starts with the FCM token format (for Android). Use the FCM v1 HTTP API with `FCM_SERVER_KEY`. The push payload should mirror the SMS message.
- Send a push notification via APNs if the user has a `device_token` that starts with an APNs token format (for iOS). Use the APNs HTTP/2 API with the credentials from `APNS_KEY_ID`, `APNS_TEAM_ID`, and `APNS_BUNDLE_ID`.
- If SMS delivery fails, log the Twilio error to the console but do NOT throw — the function should return success regardless of delivery failures, so job status updates are never blocked by notification errors.
- If push delivery fails, log the error but continue.
- Return `{ sms_sent: boolean, push_sent: boolean, errors: [] }`

**Wire notifications into existing job status transitions:**

Update these existing Edge Functions to call `send-notification` after a successful state change:
- `create-booking` → call with `event_type: booking_confirmed`, `user_id: customer_id`
- `accept-job` (driver accepts) → call with `event_type: driver_dispatched`, `user_id: customer_id`
- `update-job-status` when transitioning to `arrived` → call with `event_type: driver_arrived`, `user_id: customer_id`
- `stripe-webhook` on `payment_intent.succeeded` → call with `event_type: job_complete`, `user_id: customer_id`

**Device token registration:**

Update the `users` table update logic so that when a user logs in, they can submit their device token. Create a small Edge Function `register-device-token` that:
- Accepts: `device_token`
- Updates `users.device_token` for the authenticated user
- Returns 200 OK

**What NOT to build yet:** No maps UI, no driver onboarding screens, no admin or partner portal pages.

Before moving on, confirm the following are working: calling `send-notification` directly with a test user ID and `booking_confirmed` event results in an SMS arriving at the test phone number via Twilio; the function returns success even when the device_token is null (graceful fallback); calling `create-booking` triggers a notification call without the booking function failing if notification delivery errors.

---

## Prompt 6

Run this after Prompt 5 is complete.

Build the driver onboarding and background check flow. This is a Lovable project using Supabase. The drivers table, users table, and S3 bucket configuration (`S3_DOCUMENTS_BUCKET`) are already in place. `BGC_API_KEY` and `BGC_WEBHOOK_SECRET` are already set in the project environment.

**What to build:**

**Document upload — Edge Function `get-document-upload-url`:**
- Accepts: `document_type` (one of: `drivers_license`, `vehicle_registration`, `insurance`)
- Requires the caller to have role `driver`
- Generates a pre-signed S3 PUT URL for the bucket `S3_DOCUMENTS_BUCKET` with key `documents/{driver_id}/{document_type}/{timestamp}`
- Returns `{ upload_url, object_key }` — the driver app uploads the file directly to S3 using this URL; the file never passes through your backend

**Background check submission — Edge Function `submit-background-check`:**
- Requires the caller to have role `driver`
- Reads the driver's record from the `drivers` table
- Posts the driver's name, email, and phone to the background check provider API using `BGC_API_KEY`
- Implement the provider call behind an abstraction: create a module called `bgc-provider` that exports a `submitCheck(driverData)` function. The Edge Function calls this module, not the provider API directly. This makes it straightforward to swap providers later.
- Stores the provider's reference ID in `drivers.background_check_provider_ref`
- Updates `drivers.background_check_status` to `submitted`
- Returns the updated driver record

**Background check result — Edge Function `bgc-webhook`:**
- Public endpoint that receives POST results from the background check provider
- Verifies the webhook signature using `BGC_WEBHOOK_SECRET` if the provider supports it; log a warning but continue if signature verification is not available
- Reads the provider reference ID from the payload, looks up the matching driver record
- Maps the provider's result to `passed` or `failed` and updates `drivers.background_check_status`
- If `passed`: sends an admin notification (call `send-notification` with `user_id` of any user with role `admin` and a custom message: "Driver [name] has passed their background check and is awaiting your approval.")
- If `failed`: sets `drivers.status` to `suspended`
- Returns 200 OK

**Admin approval — Edge Function `approve-driver`:**
- Accepts: `driver_id`, `action` (one of: `approve`, `suspend`, `manual_override`)
- Requires the caller to have role `admin`
- `approve`: sets `drivers.status` to `approved` and `drivers.onboarding_completed_at` to now()
- `suspend`: sets `drivers.status` to `suspended`
- `manual_override`: sets `drivers.status` to `approved`, `drivers.background_check_status` to `manual_override`, and `drivers.onboarding_completed_at` to now() — this is the fallback when the background check provider is unavailable
- Returns the updated driver record

**Onboarding status polling — Edge Function `get-onboarding-status`:**
- Requires the caller to have role `driver`
- Returns the authenticated driver's current `status` and `background_check_status`
- Used by the driver app to poll for their approval state

**Driver onboarding UI — React components:**

Build a multi-step onboarding flow with these screens (not a full page layout, just the component screens):

1. `DocumentUploadStep` — shows three upload buttons (Driver's License, Vehicle Registration, Insurance). Each button calls `get-document-upload-url` and then uploads the file to the returned S3 URL. Shows a checkmark when each document is uploaded successfully.

2. `BackgroundCheckStep` — a screen shown after all three documents are uploaded. Has a single "Submit Background Check" button that calls `submit-background-check`. After submission, shows: "Your background check has been submitted. We'll notify you once it's complete — this usually takes 1–2 business days."

3. `OnboardingStatusStep` — shown while the driver is waiting. Polls `get-onboarding-status` every 30 seconds. Shows the current status. If status is `approved`, shows a "You're approved! Start accepting jobs." success state.

**What NOT to build yet:** No customer-facing screens, no admin or partner portal pages, no map views.

Before moving on, confirm the following are working: a driver user can call `get-document-upload-url` and receive a pre-signed URL; calling `submit-background-check` updates the driver record's status to `submitted` and stores a reference ID; calling `approve-driver` with `manual_override` sets the driver status to `approved`; the `DocumentUploadStep` component renders and the upload buttons are functional in the Lovable preview.

---

## Prompt 7

Run this after Prompt 6 is complete.

Build the Partner Portal — the web interface for garage and lot partners to manage their location and view jobs and earnings. This is a Lovable project using Supabase. All database tables and Edge Functions from previous steps are already in place.

**What to build:**

**Partner Edge Functions:**

`get-partner-jobs`:
- Requires the caller to have role `partner`
- Looks up the partner record for the authenticated user
- Returns all jobs where `partner_id` matches the partner's ID
- Accepts optional query params: `status` (filter by job status), `from_date` and `to_date` (filter by `scheduled_at`)
- Returns jobs ordered by `scheduled_at` descending

`get-partner-earnings`:
- Requires the caller to have role `partner`
- Looks up the partner record for the authenticated user
- Joins `jobs` and `payments` for all complete jobs at this partner's location
- Returns: total earnings (sum of final_price for complete jobs), job count, and a list of individual job earnings with date and amount
- Accepts optional `from_date` and `to_date` params for filtering

`update-partner-profile`:
- Accepts: `location_name`, `address`, `capacity`
- Requires the caller to have role `partner`
- Updates the partner's record in the `partners` table
- Returns the updated partner record

**Partner Portal UI — full React SPA:**

Build a complete Partner Portal with these pages/views using React Router for navigation. Use a clean, professional design with a sidebar navigation.

**Login page** (`/login`):
- Email/password login using Supabase Auth
- Redirects to dashboard on success
- Shows an error if credentials are wrong

**Dashboard** (`/dashboard`):
- Summary cards showing: today's job count, today's earnings, total all-time earnings, current active jobs count
- A live jobs table showing all jobs currently in an active status (`dispatched`, `en_route`, `arrived`, `in_progress`) for this partner, with columns: Job ID, Customer name (first name only for privacy), Status, Scheduled time. Auto-refreshes every 30 seconds.

**Jobs page** (`/jobs`):
- A filterable table of all jobs for this partner
- Filter controls: status dropdown, date range picker
- Columns: Job ID, Status, Scheduled At, Driver (first name), Final Price
- Pagination if more than 25 results

**Earnings page** (`/earnings`):
- Total earnings summary at the top (all time, current month)
- A bar chart showing earnings by week for the past 12 weeks (use Recharts)
- A table of individual job earnings with date and amount

**Profile page** (`/profile`):
- Editable form showing the partner's location name, address, and capacity
- Save button that calls `update-partner-profile`
- Shows a success toast on save

**What NOT to build yet:** No admin dashboard, no customer or driver screens.

Before moving on, confirm the following are working: a partner user can log in and see their dashboard with accurate job counts; the jobs page loads and filters work correctly; the earnings page displays the chart and table; the profile form saves correctly and the Supabase record is updated.

---

## Prompt 8

Run this after Prompt 7 is complete.

Build the Admin Dashboard — the internal web interface for managing drivers, jobs, disputes, and users. This is a Lovable project using Supabase. All database tables and Edge Functions from previous steps are in place.

**What to build:**

**Admin Edge Functions:**

`get-all-jobs`:
- Requires the caller to have role `admin`
- Returns all jobs across all cities and partners
- Accepts optional filter params: `status`, `city`, `driver_id`, `customer_id`, `from_date`, `to_date`
- Returns jobs with joined user info (customer name, driver name) and partner name
- Ordered by `created_at` descending, paginated (25 per page)

`get-all-users`:
- Requires the caller to have role `admin`
- Returns all users (excluding PII-nullified records)
- Accepts optional filter: `role`
- Returns user ID, role, email, phone, full name, created_at, deleted_at

`get-pending-drivers`:
- Requires the caller to have role `admin`
- Returns all driver records where `status = pending` and `background_check_status = passed`
- Joins user info (name, email, phone)

`admin-manage-user`:
- Accepts: `user_id`, `action` (one of: `suspend`, `reactivate`, `delete`)
- Requires the caller to have role `admin`
- `suspend`: sets the user's role to a suspended state (add a `suspended` boolean column to users if not already present)
- `reactivate`: reverses the suspension
- `delete`: soft-deletes by setting `users.deleted_at` to now() and nullifying PII fields (set `email`, `phone`, `full_name`, `device_token` to null and set `pii_nullified_at` to now()) — this is the CCPA deletion workflow
- Logs the admin action to CloudWatch (use a Supabase log insert to a new `admin_audit_log` table)

Create the `admin_audit_log` table:
- `id` (uuid, primary key)
- `admin_user_id` (uuid, FK to users.id)
- `action` (text)
- `target_user_id` (uuid, nullable)
- `target_job_id` (uuid, nullable)
- `notes` (text)
- `created_at` (timestamptz, default now())
- RLS: only admins can read this table; no deletes permitted

`get-disputes`:
- Requires role `admin`
- Returns all payments with status `disputed`, joined with job info (customer name, driver name, partner name, final_price, completed_at)

`resolve-dispute`:
- Accepts: `payment_id`, `resolution` (one of: `refund`, `uphold`), `notes`
- Requires role `admin`
- If `refund`: calls the Stripe API to issue a full refund on the `stripe_payment_intent_id` and sets `payments.status` to `refunded`
- If `uphold`: sets `payments.status` to `succeeded` (no Stripe action)
- Logs the action to `admin_audit_log`
- Returns the updated payment record

**Admin Dashboard UI — full React SPA:**

Build a complete Admin Dashboard with sidebar navigation and these pages. Use a clean, data-dense admin design.

**Login page** (`/admin/login`):
- Email/password login using Supabase Auth
- Verifies the user has role `admin` after login — if not, show "Access denied" and sign them out
- Redirects to dashboard on success

**Dashboard** (`/admin/dashboard`):
- Summary cards: total jobs today, active jobs now, total revenue today, pending driver approvals count
- A real-time active jobs table (same columns as partner portal but across all cities) — refreshes every 20 seconds
- A "Pending Approvals" alert banner if there are drivers awaiting approval, with a link to the Drivers page

**Jobs page** (`/admin/jobs`):
- Full filterable jobs table across all cities using `get-all-jobs`
- Filters: status, city, date range
- Columns: Job ID, City, Customer, Driver, Partner, Status, Scheduled At, Final Price
- Click a row to see full job detail in a side panel (all fields)

**Drivers page** (`/admin/drivers`):
- Two tabs: "Pending Approval" and "All Drivers"
- Pending tab uses `get-pending-drivers` and shows each driver with an Approve / Suspend / Manual Override button — each calls `approve-driver` from Prompt 6
- All Drivers tab uses `get-all-users` filtered to role `driver`

**Disputes page** (`/admin/disputes`):
- Table of all disputed payments using `get-disputes`
- Each row has Refund and Uphold buttons that call `resolve-dispute`
- Prompts for a notes field before confirming either action

**Users page** (`/admin/users`):
- Full user table using `get-all-users`
- Filter by role
- Each row has: Suspend, Reactivate, Delete buttons — each calls `admin-manage-user`
- Delete shows a confirmation modal with the text "This will permanently wipe this user's personal data and cannot be undone."

**What NOT to build yet:** No customer or driver mobile screens, no maps integration.

Before moving on, confirm the following are working: an admin user can log in and is blocked if their role is not `admin`; the pending drivers tab shows the correct records and the Approve button updates the driver status; the disputes page loads and the Refund action calls Stripe and updates the payment record; the delete user action nullifies PII fields in the Supabase `users` table and logs to `admin_audit_log`.

---

## Prompt 9

Run this after Prompt 8 is complete.

Build the full customer-facing and driver-facing mobile app screens. This is a Lovable project using Supabase. All Edge Functions, database tables, and real-time infrastructure from previous steps are in place.

Note: Lovable will generate these as React web components. These are designed to be exported and used inside a React Native project — build them as mobile-first, touch-friendly UI components with no desktop layout assumptions. Use `flex` column layouts, large tap targets, and avoid hover states.

**What to build:**

**Customer app screens:**

`CustomerLoginScreen`:
- Email/password login via Supabase Auth
- Link to a sign-up screen
- On success, navigates to the customer home screen

`CustomerSignUpScreen`:
- Email, password, full name, phone number fields
- Creates a Supabase auth user and then calls `register-device-token` to save the device token
- On success, navigates to home

`CustomerHomeScreen`:
- Shows a list of upcoming and past bookings from `get-customer-bookings`
- A prominent "Book a Valet" button that navigates to the booking creation screen
- Each booking shows: status badge, pickup address, scheduled date/time

`BookingCreateScreen`:
- Fields: pickup address (text input), dropoff address (text input), scheduled date/time (datetime picker), city selector (New York / Los Angeles / Chicago), partner selector (optional — show a simple list)
- A "Get Price Estimate" button that calls `create-booking` in preview mode — show the calculated `final_price` and `surge_multiplier` before confirming
- A "Confirm Booking" button that finalizes the booking
- Shows a success screen with the job ID and scheduled time after creation

`BookingDetailScreen`:
- Accepts a `jobId` prop
- Shows all job details: status, pickup/dropoff addresses, scheduled time, driver name (if assigned), partner name, final price
- Shows a "Track Driver" button when job status is `dispatched`, `en_route`, or `arrived` — navigates to `DriverTrackingScreen`
- Shows a "Cancel Booking" button when status is `scheduled` — prompts for a cancellation reason and calls `cancel-booking`
- Shows the `PaymentForm` component (from Prompt 4) when status is `complete` and no payment exists yet

`DriverTrackingScreen`:
- Accepts a `jobId` prop
- Uses the `useDriverLocation(jobId)` hook from Prompt 3 to receive live location updates
- Displays the driver's current latitude and longitude as text (placeholder for the maps SDK, which will be wired in the native app)
- Shows "Last updated: [timestamp]" below the coordinates
- Shows a reconnecting indicator if the Realtime connection drops
- Shows the current job status

**Driver app screens:**

`DriverLoginScreen`:
- Email/password login via Supabase Auth
- Checks driver status after login — if `pending`, redirect to onboarding; if `suspended`, show an error; if `approved`, go to job queue

`DriverOnboardingScreen`:
- Renders the three onboarding step components from Prompt 6 in sequence: `DocumentUploadStep` → `BackgroundCheckStep` → `OnboardingStatusStep`
- Tracks which step is complete and shows a progress indicator

`DriverJobQueueScreen`:
- Calls `get-available-jobs` on mount and polls every 15 seconds
- Shows a list of available jobs with: pickup address, city, estimated fare (final_price), scheduled time
- Each job card has Accept and Decline buttons
- Accept calls `accept-job` and navigates to `DriverActiveJobScreen`

`DriverActiveJobScreen`:
- Accepts a `jobId` prop
- Shows the current job details: pickup address, customer name (first name only), status
- Uses the `useLocationBroadcast(jobId)` hook from Prompt 3 to continuously send GPS location
- Status update buttons that call `update-job-status`:
  - When status is `dispatched`: "I'm On the Way" → transitions to `en_route`
  - When status is `en_route`: "I've Arrived" → transitions to `arrived`
  - When status is `arrived`: "Start Job" → transitions to `in_progress`
  - When status is `in_progress`: "Complete Job" → transitions to `complete`
- Shows earnings amount when the job is complete

`DriverEarningsScreen`:
- Shows the driver's completed jobs from `get-available-jobs` (modify or create a `get-driver-completed-jobs` Edge Function that returns jobs where `driver_id = auth.uid()` and `status = complete`)
- Summary: total earnings this week, total earnings all time
- A list of completed jobs with date, final price, and pickup address

**What NOT to build yet:** No maps SDK integration (placeholder text coordinates only), no background check provider changes.

Before moving on, confirm the following are working: a customer can log in, create a booking, and see it appear in their booking list; a driver can log in, see available jobs, accept a job, and step through all status transitions; the `DriverTrackingScreen` shows location updates when `broadcast-driver-location` is called for that job; the `PaymentForm` appears on a complete job and renders without errors.

---

## Prompt 10

Run this after Prompt 9 is complete.

Build the observability, compliance automation, and launch hardening layer. This is a Lovable project using Supabase. All application features, screens, and Edge Functions from previous steps are in place.

**What to build:**

**Scheduled jobs — Supabase Cron (pg_cron):**

Create these two scheduled database jobs using Supabase's pg_cron extension (enable it in Supabase under Database → Extensions):

`purge-driver-location-history` — runs daily at 2:00 AM UTC:
- Deletes all rows from `driver_location_history` where `recorded_at` is older than 30 days after the associated job's `completed_at`
- SQL: delete rows where `recorded_at < (SELECT completed_at FROM jobs WHERE jobs.id = driver_location_history.job_id) + INTERVAL '30 days'` and the job's `completed_at` is not null
- Log the number of deleted rows to a Supabase log table (create `system_logs` table: `id`, `job_name` text, `rows_affected` integer, `run_at` timestamptz)

`cleanup-stale-websocket-connections` — runs every hour:
- Deletes rows from `websocket_connections` where `connected_at` is older than 2 hours
- This catches connections that didn't cleanly deregister

**CCPA data export — Edge Function `export-user-data`:**
- Requires the caller to be an admin or the user themselves
- Accepts: `user_id`
- Queries all records linked to the user: their `users` row, `jobs` (as customer), `payments`, `driver_location_history` (if driver), `drivers` record (if driver), `bookings`
- Returns a JSON object with all of these records grouped by table name
- This is the CCPA data export response — returns the full structured data as a JSON download

**Monitoring dashboard — Admin UI additions:**

Add a new "System Health" page to the Admin Dashboard (`/admin/system`):

- **Edge Function error log**: Query the `system_logs` table and show recent log entries with job name, rows affected, and run time
- **Purge job status**: Show the last run time and rows deleted for `purge-driver-location-history` — show a red warning badge if the last run was more than 48 hours ago
- **Database stats panel**: Show counts from key tables — total jobs, total users by role, total payments, total location history rows, total websocket_connections rows — query these live from Supabase
- **CCPA tools panel**:
  - A user lookup field (by email) that fetches the user record and shows their ID
  - A "Export User Data" button that calls `export-user-data` and triggers a JSON file download in the browser
  - A "Delete User Data" button (links to the existing delete action in the Users page for that user)

**Input validation and error hardening — review all Edge Functions:**

Go through every Edge Function built in previous prompts and ensure:
- All user-supplied string inputs are trimmed and length-validated (e.g. addresses max 500 chars, reasons max 1000 chars)
- All UUID inputs are validated to be valid UUID format before any database query
- All numeric inputs (latitude, longitude, amounts) are validated to be within reasonable ranges (latitude: -90 to 90, longitude: -180 to 180, amounts: 0 to 100000 cents)
- Every function returns a consistent error response shape: `{ error: true, message: "...", code: "..." }` for all failure cases
- No function returns a raw database error message to the client — map database errors to generic user-facing messages and log the raw error server-side

**Stripe webhook resilience:**

Update the `stripe-webhook` Edge Function from Prompt 4:
- Before processing any event, check if the payment record is already in the target final state (succeeded, refunded, disputed) — if yes, return 200 immediately without re-processing (idempotency guard)
- Log every received webhook event type and the payment_id to the `system_logs` table with `job_name = 'stripe-webhook'`

**Rate limiting — basic protection:**

Add basic rate limiting to the most sensitive public-facing Edge Functions (`create-booking`, `create-payment-intent`, `register-device-token`):
- Track calls per user per minute using a Supabase table `rate_limit_log` (`user_id`, `function_name`, `called_at`)
- If a user calls the same function more than 10 times in 60 seconds, return a 429 response with `{ error: true, message: "Too many requests. Please try again shortly.", code: "RATE_LIMITED" }`
- Clean up `rate_limit_log` entries older than 5 minutes in the same scheduled cron job that cleans up websocket connections

**Staging environment checklist — verify before production:**

Create a simple markdown file called `LAUNCH_CHECKLIST.md` in the project root with the following checklist. The developer must verify each item manually before promoting to production:

```markdown
# Valet Platform Launch Checklist

## Stripe
- [ ] Stripe webhook URL updated to production endpoint
- [ ] Stripe keys switched from test to live in environment variables
- [ ] Webhook signing secret updated for live endpoint

## Twilio
- [ ] Twilio phone number confirmed active in production account
- [ ] SMS sending tested end-to-end with a real phone number

## AWS
- [ ] S3 buckets (documents and receipts) confirmed private with no public access
- [ ] AWS Budget alert configured and email confirmed
- [ ] SNS topic subscribed with founder's email or phone

## Background Check Provider
- [ ] Provider account fully onboarded and compliance agreements signed
- [ ] Webhook URL registered with provider pointing to production bgc-webhook endpoint
- [ ] Test submission completed and result received successfully

## Supabase
- [ ] RLS confirmed enabled on all tables
- [ ] pg_cron extension enabled and both scheduled jobs confirmed running
- [ ] Automated backups enabled with at least 7-day retention
- [ ] Point-in-time recovery enabled

## Security
- [ ] Admin Dashboard access tested — non-admin users are blocked
- [ ] Stripe webhook signature verification confirmed rejecting invalid signatures
- [ ] Rate limiting tested — 11th call within 60 seconds returns 429
- [ ] All PII fields confirmed nullified on user delete test

## Smoke Tests (run against staging before production)
- [ ] Customer can book, driver can accept, status transitions all work
- [ ] Payment flow completes end-to-end in Stripe test mode
- [ ] Notification SMS delivered to test number
- [ ] Driver location broadcast received by customer tracking screen
- [ ] Partner portal shows correct job and earnings data
- [ ] Admin can approve a driver and resolve a dispute
- [ ] CCPA export returns a complete JSON file for a test user
- [ ] CCPA delete nullifies PII and soft-deletes the user record
```

Before moving on, confirm the following are working: the `purge-driver-location-history` cron job is enabled in Supabase and a manual trigger deletes the correct rows; the `export-user-data` function returns a complete JSON structure for a test user; the System Health page in the Admin Dashboard loads and shows accurate table counts; submitting more than 10 booking requests in 60 seconds from the same user returns a 429 response; the `LAUNCH_CHECKLIST.md` file is committed to the repository.