# Implementation Plan: Hitify Fitness & Coaching Platform

## Goal Description
Build a modern, full-stack personal training and coaching marketplace (**Hitify**) where clients can discover certified trainers, purchase video-based workout programs, track their exercise progress, engage in real-time chat with coaches, and book 1-on-1 live video coaching sessions. 

The platform features strict role segregation between Clients and Trainers, automated split payouts via Stripe Connect Express, direct-to-ImageKit video uploads to bypass serverless payload limits, an in-app calendar slot engine with timezone conversion, real-time messaging and video calls powered by Stream, and an athletic dark luxury design system.

---

## User Review Required

> [!IMPORTANT]
> **API Keys & External Accounts Setup Required**
> Before production deployment, the following 3rd-party services must be provisioned and keys added to `.env.local`:
> 1. **Neon PostgreSQL:** Connection string with pooled connection for serverless runtime.
> 2. **Stripe:** Account with Stripe Connect enabled (Standard/Express), API secret key, publishable key, and webhook secret.
> 3. **Stream (GetStream.io):** App ID, API Key, and Secret supporting both Chat and Video products.
> 4. **ImageKit:** Public Key, Private Key, and URL Endpoint for video uploads and streaming.
> 5. **Sentry:** DSN for `@sentry/nextjs` error tracking and performance profiling.

> [!WARNING]
> **Stripe Connect Webhook Local Testing**
> Local development requires the Stripe CLI (`stripe listen --forward-to localhost:3000/api/webhooks/stripe`) to process `checkout.session.completed` and `account.updated` events.

---

## Architecture Overview

```mermaid
graph TD
    subgraph Client App [Next.js 14 App Router]
        UI[Luxury Dark UI / Tailwind + shadcn]
        BA[Better-Auth Client]
        SC[Stream Chat & Video SDK]
        IK[ImageKit Video Player & Upload]
    end

    subgraph Backend APIs [Next.js Server Actions & Route Handlers]
        AuthAPI["/api/auth/* (Better-Auth)"]
        StreamAPI["/api/stream/token"]
        ImageKitAPI["/api/imagekit/auth"]
        StripeAPI["/api/stripe/* & /api/webhooks/stripe"]
        BookingEngine[Slot Calculation & Booking Engine]
    end

    subgraph External Infrastructure
        Neon[(Neon Serverless Postgres + Drizzle ORM)]
        StripeSvc[Stripe Connect Express]
        StreamSvc[Stream Chat & Video Servers]
        ImageKitSvc[ImageKit CDN & Media Storage]
        SentrySvc[Sentry Observability]
    end

    UI --> AuthAPI
    UI --> StreamAPI
    UI --> ImageKitAPI
    UI --> StripeAPI
    AuthAPI --> Neon
    BookingEngine --> Neon
    StreamAPI --> StreamSvc
    ImageKitAPI --> ImageKitSvc
    StripeAPI --> StripeSvc
    SC --> StreamSvc
    IK --> ImageKitSvc
    Client App --> SentrySvc
```

---

## Proposed Changes

### 1. Project Initialization & Foundation
Bootstrap the Next.js 14 App Router application with TypeScript, Tailwind CSS, shadcn/ui components, and baseline configuration.

#### [NEW] `package.json`
Dependencies:
* `next`, `react`, `react-dom`
* `drizzle-orm`, `@neondatabase/serverless`, `drizzle-kit`
* `better-auth`
* `stripe`, `@stripe/stripe-js`
* `stream-chat`, `stream-chat-react`, `@stream-io/video-react-sdk`
* `imagekit`, `imagekit-javascript`
* `@sentry/nextjs`
* `lucide-react`, `tailwind-merge`, `clsx`, `class-variance-authority`, `framer-motion`, `next-themes`, `date-fns`

#### [NEW] `tailwind.config.ts` & `src/app/globals.css`
* Obsidian dark base (`#09090b`), slate borders (`#27272a`), energetic athletic accents (electric lime `#D4FF00` and cyan `#00F0FF`).
* Custom scrollbars, glassmorphism card utilities, and smooth transitions.

---

### 2. Database & Data Model Layer (Drizzle + Neon)

#### [NEW] `src/db/schema.ts`
Define complete relational tables:
1. **Auth & Identity:**
   * `users`: `id`, `name`, `email`, `emailVerified`, `image`, `role` (`'client' | 'trainer'`), `createdAt`, `updatedAt`.
   * `sessions`, `accounts`, `verifications` (Better-Auth standard tables).
2. **Trainer Profiles & Availability:**
   * `trainer_profiles`: `id`, `userId` (fk), `headline`, `bio`, `specialties` (jsonb/text array), `hourlyRate`, `stripeAccountId`, `stripeOnboardingComplete`, `timezone`, `rating`, `reviewCount`.
   * `availability_slots`: `id`, `trainerId` (fk), `dayOfWeek` (0-6), `startTime` (e.g. "09:00"), `endTime` (e.g. "17:00"), `slotDurationMinutes` (default 45), `bufferMinutes` (default 15), `isActive`.
   * `availability_overrides`: `id`, `trainerId` (fk), `date` (YYYY-MM-DD), `isBlocked` (boolean), `customStartTime`, `customEndTime`.
3. **Programs & Media:**
   * `programs`: `id`, `trainerId` (fk), `title`, `slug`, `description`, `price` (cents), `category` (HIIT, Strength, Yoga, etc.), `thumbnailUrl`, `isPublished`, `createdAt`.
   * `chapters`: `id`, `programId` (fk), `title`, `orderIndex`.
   * `lessons`: `id`, `chapterId` (fk), `title`, `description`, `videoUrl` (ImageKit), `videoDuration` (seconds), `orderIndex`, `isFreePreview`.
4. **Commerce & Access:**
   * `purchases`: `id`, `clientId` (fk), `programId` (fk), `stripePaymentIntentId`, `amount`, `platformFee`, `status`, `createdAt`.
   * `lesson_progress`: `id`, `clientId` (fk), `lessonId` (fk), `isCompleted`, `lastPositionSeconds`, `completedAt`.
   * `bookings`: `id`, `clientId` (fk), `trainerId` (fk), `startTime` (timestamp with tz), `endTime` (timestamp with tz), `status` (`'pending' | 'confirmed' | 'canceled' | 'completed'`), `stripeTransferId`, `streamCallId`, `notes`.

#### [NEW] `src/db/index.ts`
* Neon serverless client configuration with Drizzle ORM instance.

---

### 3. Authentication & Authorization (Better-Auth)

#### [NEW] `src/lib/auth.ts` & `src/lib/auth-client.ts`
* Configure Better-Auth with Neon Drizzle adapter.
* Multi-role support (`client` vs `trainer`).
* Email/password credentials + OAuth (Google/GitHub optional).

#### [NEW] `src/middleware.ts`
* Route protection guards:
  * `/trainer/*`: Enforce authenticated user with `role === 'trainer'`.
  * `/client/*`: Enforce authenticated user with `role === 'client'`.
  * Redirect unauthenticated users to `/sign-in`.

---

### 4. Stripe Connect & Payments Engine

#### [NEW] `src/lib/stripe.ts`
* Stripe server SDK initialization.
* Helper functions:
  * `createTrainerConnectAccount(trainerId)`: Generates Express onboarding link with return URLs.
  * `createProgramCheckoutSession(programId, clientId)`: Direct charge with application fee (15% platform cut) transferred to trainer's connected account.
  * `createBookingCheckoutSession(bookingData, clientId)`: Escrows/transfers payment for 1-on-1 session.

#### [NEW] `src/app/api/webhooks/stripe/route.ts`
* Handles:
  * `account.updated`: Updates `trainer_profiles.stripeOnboardingComplete` when verification passes.
  * `checkout.session.completed`: Creates `purchases` record or marks `bookings` as confirmed, creates Stream channel/call, and triggers notifications.

---

### 5. Media Pipeline (ImageKit Direct Upload & Player)

#### [NEW] `src/app/api/imagekit/auth/route.ts`
* Issues short-lived cryptographic upload tokens for client-side uploads directly to ImageKit.

#### [NEW] `src/components/media/VideoUploader.tsx`
* Direct client-to-ImageKit chunked file uploader with upload percentage indicator, drag-and-drop, and cancel handling.

#### [NEW] `src/components/media/VideoPlayer.tsx`
* Premium video player with custom controls, playback speed (0.75x–2x), theater mode, and automatic progress bookmarking to `lesson_progress`.

---

### 6. Real-time Communication (Stream Chat & Video SDKs)

#### [NEW] `src/app/api/stream/token/route.ts`
* Authenticates session user.
* Generates Stream user token valid for both Stream Chat and Stream Video.
* Ensures user is synced to Stream with avatar and display name.

#### [NEW] `src/components/chat/StreamChatWrapper.tsx`
* Integrated chat window supporting channels, message reactions, threads, file attachments, and active trainer presence.
* Channel auto-created upon program purchase or confirmed 1-on-1 booking.

#### [NEW] `src/app/(dashboard)/sessions/[id]/room/page.tsx`
* Full-screen 1-on-1 video call room using Stream Video React SDK:
  * Pre-call lobby (mic/camera toggle and speaker test).
  * In-call HUD with remaining session countdown timer.
  * Screen sharing, camera flip, and end-call confirmation.
  * Auto-navigation to session summary upon completion.

---

### 7. Availability & Slot Engine

#### [NEW] `src/lib/scheduling.ts`
* `getAvailableSlots(trainerId, targetDate, userTimezone)`:
  * Fetches trainer's weekly template + overrides.
  * Subtracts existing confirmed bookings.
  * Generates 45-min slots with 15-min buffers.
  * Converts UTC slots to client's local timezone.

---

### 8. Core UI Pages & Dashboards

#### Public Pages:
* `src/app/page.tsx`: Landing page with hero banner, high-impact typography, featured trainers carousel, and category pills.
* `src/app/explore/page.tsx`: Browse all programs and trainers with instant search, category filters, and price sorting.
* `src/app/trainers/[id]/page.tsx`: Trainer profile showcasing bio, specialties, programs catalog, and 1-on-1 booking calendar.
* `src/app/programs/[slug]/page.tsx`: Program sales page with curriculum preview, trailer video, and Stripe purchase button.

#### Client Experience:
* `src/app/(dashboard)/client/programs/page.tsx`: Enrolled programs library with completion progress bars.
* `src/app/(dashboard)/client/programs/[slug]/watch/page.tsx`: Course player with chapter/lesson sidebar and "Mark as Complete" toggle.
* `src/app/(dashboard)/client/messages/page.tsx`: Stream chat inbox with coaches.
* `src/app/(dashboard)/client/bookings/page.tsx`: Upcoming & past 1-on-1 video sessions with "Join Call" button.

#### Trainer Dashboard:
* `src/app/(dashboard)/trainer/page.tsx`: Analytics dashboard with gross earnings, platform cut, active clients, and upcoming calls.
* `src/app/(dashboard)/trainer/programs/new/page.tsx`: Curriculum builder with drag-and-drop chapters, lesson sorting, and direct ImageKit video upload.
* `src/app/(dashboard)/trainer/availability/page.tsx`: Weekly availability schedule editor with timezone picker and buffer configuration.
* `src/app/(dashboard)/trainer/payouts/page.tsx`: Stripe Connect onboarding banner, balance view, and Stripe dashboard portal link.

---

### 9. Observability & Deployment

#### [NEW] `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`
* Sentry instrumentation for error tracking, unhandled exceptions, and API route tracing.

#### [NEW] `.env.example`
* Full template documenting all required environment variables.

---

## Verification Plan

### Automated Tests & Quality Checks
* `npm run build`: Verify zero TypeScript compilation errors and successful Next.js static/dynamic route generation.
* `npx drizzle-kit check`: Verify schema migrations are synchronized with Neon.
* `npm run lint`: ESLint check with zero warnings or errors.

### Manual Verification Workflows

#### 1. Trainer Onboarding & Content Publishing
1. Register new user as `Trainer`.
2. Navigate to `/trainer/payouts` and click "Connect with Stripe" -> complete simulated Express onboarding.
3. Set availability in `/trainer/availability` (e.g. Mon–Fri 9am–5pm).
4. Create new program in `/trainer/programs/new`:
   * Upload video lesson via ImageKit uploader -> confirm progress bar updates and preview displays.
   * Publish program for $49.00.

#### 2. Client Purchase & Content Consumption
1. Register new user as `Client`.
2. Browse `/explore`, find published program, click "Buy Now".
3. Complete Stripe Checkout in test mode (`4242...`).
4. Verify immediate redirect to program player:
   * Play video via ImageKit player.
   * Click "Mark as Complete" -> verify progress percentage increments in DB and UI.

#### 3. 1-on-1 Booking & Live Video Call
1. From trainer profile, select tomorrow at 10:00 AM -> complete Stripe checkout for 1-on-1 session.
2. Verify booking appears in both Client and Trainer calendars.
3. Open `/sessions/[id]/room` as both Client and Trainer in separate browser profiles:
   * Verify pre-call device selection works.
   * Join room -> confirm bidirectional audio, video, and active countdown timer.
   * End call -> verify clean exit.

#### 4. Stream Real-time Chat
1. Open `/client/messages` -> verify 1-on-1 channel with trainer is open.
2. Send message, reaction, and thread reply.
3. Open `/trainer/messages` -> verify instant real-time message delivery without page reload.

