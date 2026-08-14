# Valet Platform — Before You Build for Lovable

## How to Use This Prompt Pack

This Prompt Pack is a set of sequential prompts you paste into Lovable, one at a time, to build your Valet Platform from the ground up. Each prompt covers one focused area — paste it in, let Lovable finish, confirm everything works, then move to the next.

## How Lovable Works — What to Expect

Lovable is chat-first and conversational. It will sometimes ask clarifying questions before writing code — answer using the context in each prompt. It builds your frontend visually and generates backend logic, but for a project this size you'll be connecting external services (AWS, Stripe, Twilio, etc.) as you go.

A few things to know before you start:

- **Lovable builds React frontends natively.** Your Partner Portal and Admin Dashboard will be built directly inside Lovable. Your mobile apps (React Native) will need to be generated and exported for use in a separate React Native project — Lovable will scaffold the logic and components, but React Native runs outside Lovable's preview environment.
- **Lovable connects to Supabase for your database and auth layer.** Even though the architecture calls for AWS RDS and AWS Lambda, Lovable's native backend integration is Supabase. Use Lovable + Supabase for your database, auth, and edge functions during development. When you're ready to migrate to AWS for production, the schema and logic will be directly portable.
- **Answer clarifying questions with context from the prompt.** If Lovable asks what kind of app you're building or which features to include, use the description in that prompt to guide your answer.
- **Stay scoped.** Each prompt tells you explicitly what NOT to build yet. If Lovable tries to get ahead, steer it back.
- **Confirm before continuing.** At the end of each prompt is a confirmation check. Don't paste the next prompt until that check passes.
- **Connect a git repo before you start.** In Lovable, connect your GitHub account and create a repository for this project before running Prompt 1. This gives you a rollback point at every step — if something breaks, you can revert to the last working commit.

## Platform Configuration — Complete These Before Prompt 1

### Lovable Setup
- Create a new Lovable project.
- Connect your GitHub account in Lovable's settings and link a new repository named `valet-platform`. This enables version control and rollback.
- Enable Supabase in your Lovable project — click "Add Supabase" in the project settings and create a new Supabase project when prompted. Lovable will handle the connection automatically.
- Once Supabase is connected, go to your Supabase project dashboard and confirm the project is live before running Prompt 1.

### Supabase Setup
- In your Supabase dashboard, go to **Authentication → Providers** and enable **Email** as an auth provider.
- Go to **Settings → API** and note your **Project URL** and **anon/public key** — Lovable will use these automatically, but you'll need them when configuring external services.
- Go to **Settings → Database** and note your **connection string** — you'll need this if you migrate to AWS RDS later.
- Enable **Row Level Security (RLS)** on all tables — Lovable will generate RLS policies as tables are created, but confirm this is enabled at the project level.

### Stripe Setup
- Create a Stripe account at [dashboard.stripe.com/register](https://dashboard.stripe.com/register).
- In the Stripe Dashboard, go to **Developers → API Keys** and note your **Publishable key** and **Secret key** (use test keys for now).
- Go to **Developers → Webhooks** and create a new webhook endpoint. You won't have the URL yet — come back and fill this in after Prompt 5 when your Payment function URL is live.
- Note the **Webhook signing secret** from the webhook endpoint page — you'll need it in Prompt 5.

### Twilio Setup
- Create a Twilio account at [twilio.com/try-twilio](https://www.twilio.com/try-twilio).
- Purchase a phone number from the Twilio Console under **Phone Numbers → Manage → Buy a number**.
- Go to **Account Info** on the Twilio Console homepage and note your **Account SID** and **Auth Token**.

### Google Cloud Setup
- Create a project at [console.cloud.google.com](https://console.cloud.google.com).
- Enable the **Maps SDK for Android** and the **Firebase Cloud Messaging (FCM) API** in your project.
- Go to **APIs & Services → Credentials** and create an API key for Maps (restrict it to Android apps).
- Go to **Firebase Console** (linked to your Google Cloud project), open **Project Settings → Cloud Messaging**, and note your **Server key** for FCM.

### Apple Developer Setup
- Enroll in the Apple Developer Program at [developer.apple.com/programs](https://developer.apple.com/programs) if you haven't already.
- In **Certificates, Identifiers & Profiles**, register your app's bundle identifier (e.g. `com.yourcompany.valetplatform`).
- Go to **Keys** and create a new key with **Apple Push Notifications service (APNs)** enabled. Download the `.p8` file — you can only download it once. Note the **Key ID**.
- Go to **Account → Membership** and note your **Team ID**.

### AWS Setup (for production migration — set up now, use in later prompts)
- Create an AWS account at [aws.amazon.com](https://aws.amazon.com) if you haven't already.
- In **IAM**, create a user with programmatic access for deployments. Do not use root credentials.
- In **AWS Secrets Manager**, create placeholder secrets now (you'll populate the values as you collect them):
  - `valet/stripe/secret-key`
  - `valet/stripe/webhook-secret`
  - `valet/twilio/auth-token`
  - `valet/fcm/server-key`
  - `valet/apns/private-key`
  - `valet/bgc/api-key`
  - `valet/jwt/secret`
- Create two private **S3 buckets**: one named `valet-driver-documents` and one named `valet-receipts`. Enable versioning on both. Block all public access.
- Set up an **AWS Budget** in the Billing console with an alert at your monthly comfort threshold, routed to your email.
- Create an **SNS topic** called `valet-ops-alerts` and subscribe your email or phone number to it.

## Environment Variables — Add These in Lovable Before Prompt 1

In Lovable, open your project settings and navigate to the environment variables or secrets panel. Add each of the following before running Prompt 1. Supabase variables are added automatically by Lovable — the ones below are for external services you'll wire up across the prompts.

| Variable | Where to find it |
|---|---|
| `VITE_STRIPE_PUBLISHABLE_KEY` | Stripe Dashboard → Developers → API Keys → Publishable key |
| `STRIPE_SECRET_KEY` | Stripe Dashboard → Developers → API Keys → Secret key (use test key now) |
| `STRIPE_WEBHOOK_SECRET` | Stripe Dashboard → Developers → Webhooks → your endpoint → Signing secret (add after Prompt 5) |
| `TWILIO_ACCOUNT_SID` | Twilio Console → Account Info → Account SID |
| `TWILIO_AUTH_TOKEN` | Twilio Console → Account Info → Auth Token |
| `TWILIO_PHONE_NUMBER` | Twilio Console → Phone Numbers → your purchased number (in E.164 format, e.g. +12125550100) |
| `GOOGLE_MAPS_API_KEY` | Google Cloud Console → APIs & Services → Credentials → your Maps API key |
| `FCM_SERVER_KEY` | Firebase Console → Project Settings → Cloud Messaging → Server key |
| `APNS_KEY_ID` | Apple Developer → Keys → your APNs key → Key ID |
| `APNS_TEAM_ID` | Apple Developer → Account → Membership → Team ID |
| `APNS_BUNDLE_ID` | Your app's bundle identifier (e.g. com.yourcompany.valetplatform) |
| `BGC_API_KEY` | Your background check provider's dashboard → API credentials |
| `BGC_WEBHOOK_SECRET` | Your background check provider's dashboard → webhook signing secret |
| `AWS_REGION` | `us-east-1` |
| `S3_DOCUMENTS_BUCKET` | `valet-driver-documents` (the bucket name you created in AWS) |
| `S3_RECEIPTS_BUCKET` | `valet-receipts` (the bucket name you created in AWS) |

---