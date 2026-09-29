---
title: "QRFleet's stack: from a QR scan to a completed inspection"
date: 2026-09-29 20:00:00 +0200
categories: [Desarrollo, Automatización]
tags: [qrfleet, next.js, fastapi, postgresql, supabase, docker, python]
image: /assets/img/posts/qrfleet-stack-cover.png
description: "How QRFleet connects its Next.js interface, FastAPI backend, offline queue and managed storage to handle fleet inspections."
---

My [first QRFleet post](/posts/qrfleet-fleet-damage-reporting/) focused on replacing an informal damage-reporting process. The application has grown since then. It now handles inspection checklists, signatures, documents and an offline write queue, alongside the damage reports.

The stack makes more sense when you follow one inspection through it. A QR scan identifies an asset. The person inspecting it records answers, damages and photos, then submits a signature. Closing the inspection is a backend operation: the server checks completeness, builds a record of its content and stores a PDF. The browser cannot simply declare the inspection finished.

## The pieces and their jobs

| Layer | Tools | What they do in QRFleet |
|---|---|---|
| Web application | Next.js App Router, React, TypeScript | Routes, layouts and interactive inspection screens |
| Interface | Tailwind CSS, shadcn/ui, Radix, TanStack Table | Forms, dialogs and office-facing tables |
| Client state and languages | Zustand, next-intl | Shared UI state and five interface languages |
| Scanning and signatures | html5-qrcode, signature_pad | Read a QR with the camera and capture a handwritten signature |
| Offline support | IndexedDB through idb, Serwist | Persist pending writes and support the service-worker/PWA layer |
| API | FastAPI, Uvicorn, Pydantic | HTTP endpoints, request validation and backend business rules |
| Access control | JWT sessions, backend capability checks | Authenticate requests and enforce permissions in the organization context |
| Data model | SQLAlchemy, Alembic, PostgreSQL | Relationships, transactions and versioned schema changes |
| Managed data and files | Supabase Postgres, Supabase Storage | Production database and object storage |
| Generated files | qrcode, Pillow, fpdf2, openpyxl | QR images, image handling, PDF documents and spreadsheet exports |
| Deployment | Docker Compose, Hetzner, Caddy, Cloudflare | Application runtime, reverse proxy, TLS and selective edge caching |
| Email and operations | Resend, Sentry, PostHog, Vector, Better Stack, cron | Transactional email, error tracking, consent-gated analytics, logs and scheduled maintenance |
| Tests | pytest, Vitest, Playwright | Backend checks, client logic and browser workflows |

Not every dependency is visible in an inspection. The office tables and spreadsheet exports solve a different part of the workflow from the camera and mobile wizard.

![Simplified QRFleet architecture showing the browser, edge, application and managed services](/assets/img/posts/qrfleet-stack-architecture.png)
_A simplified view of the main connections, not every network request. The browser can also load image objects directly from storage._

## Scan first, then establish context

The browser uses `html5-qrcode` for camera scanning. On the backend, `qrcode` and Pillow generate the asset labels. These are ordinary black-and-white codes, rather than a logo stamped over the modules. Reliable scanning matters more than decorating the label.

The scan is an entry point, not an authorization mechanism. The API still has to resolve the asset in the user's organization and enforce the appropriate permissions. Hiding an action in a React component does not protect the endpoint behind it.

Next.js handles the application shell and routes. Most of the inspection interaction lives in client components: camera access, damage selection, photo inputs and the signature pad. TypeScript helps keep the client-side contracts explicit, while Pydantic validates incoming data at the API boundary. Those are separate checks, not a substitute for each other.

Tailwind and shadcn/ui let me work on the actual inspection form instead of building every dialog from scratch. TanStack Table is useful on the office side, where filtering and working through lists matter more than a step-by-step mobile screen. `next-intl` provides the language layer; the project has Spanish, English, French, German and Dutch locale files.

## Losing the connection is part of the workflow

A service worker alone does not make an inspection work offline. The application needs the right data available before the connection disappears, and somewhere durable to put new work.

QRFleet stores prefetched inspection bundles, local events and queued mutations in IndexedDB through `idb`. The offline database is partitioned by user and organization. The queue survives a page reload; it is not just an array in a Zustand store.

The sync engine drains each event's writes in creation order. If a mutation fails, it stops that event's queue instead of skipping ahead to the close operation. Retryable failures use backoff. Permanent failures remain visible as failed work, and an expired or revoked session is distinguished from a network problem.

There are limits. The necessary bundle must already be available locally, browser storage can be cleared, and the server still has the final say when the writes arrive. I do not describe this as making the whole application usable offline from a cold start.

## The API owns the completed record

FastAPI is separate from the Next.js application. That adds a deployment component and means I have to maintain the frontend/backend contract, but it keeps the inspection rules in one Python API rather than spreading them across browser code and page handlers.

SQLAlchemy maps the records and relationships to PostgreSQL. Alembic versions the schema. Development uses a PostgreSQL container; the production setup uses managed Postgres in Supabase. The production Compose file does not run its own database container.

Closing an inspection is more involved than setting a status field. Depending on the template, the backend checks required answers, required photo views and identity confirmation. It creates a deterministic summary and computes a SHA-256 content hash, then renders and persists the inspection PDF with `fpdf2`.

The close path also handles repeated submissions. A row lock serializes competing attempts, and a request for an already-closed, hashed inspection returns its existing result. This matters when a person taps twice or retries on a poor connection.

The PDF and hash are application records, not a claim of a qualified electronic signature or a guarantee of legal compliance. Database changes and object-storage writes also do not share a single transaction; the close code has explicit handling for an object left behind by an interrupted attempt.

## Keep files and email behind small interfaces

Photo and document storage go through a backend interface with local and Supabase implementations. Local storage is useful in development. Production uses Supabase Storage, while PostgreSQL keeps the related application records.

Email follows the same pattern: console output in development, Resend in production. Keeping these integrations out of the inspection handlers makes it possible to change an implementation without rewriting each endpoint. It does not eliminate the work of migrating existing objects or validating a new provider.

Pillow checks image content rather than trusting the filename or the browser's declared content type. Caddy also limits request-body size before a large upload reaches the application. They protect different boundaries.

## Running it is another part of the stack

The application runs on a Hetzner VPS with Docker Compose. Caddy routes API requests to FastAPI and application requests to Next.js. Cloudflare sits in front of the origin, with caching reserved for selected static assets rather than authenticated pages or API responses.

The frontend is built into its Docker image. A source change needs a rebuild; restarting an old container does not ship a new UI. Browser-facing `NEXT_PUBLIC_*` settings are also build-time inputs, which is worth remembering when changing an API URL or integration.

For operations, Sentry covers application errors. PostHog is wired behind consent for product analytics. Vector forwards application and Caddy logs to Better Stack, with redaction applied before those logs leave the server. Scheduled Python jobs still run through cron for tasks such as database backups and maintenance. I have not added a separate distributed worker system just to replace those jobs.

This is still a single-VPS application with managed services attached, not a cluster. The current backend includes synchronous database work, and some document operations run in the request path. Adding more services would not automatically remove those limits.

## Test the flow, not only the components

The repository has pytest checks for backend rules, Vitest tests for client logic and Playwright scenarios against the development stack. The browser smoke scenario follows the inspection through to a downloadable PDF. Other scenarios cover losing the connection and syncing again, prefetched bundles and mobile scanning layouts.

Those tests serve different purposes. A mocked client test can check queue ordering, but it cannot prove that a database trigger accepts the final transaction. A successful PDF-generation test does not prove the user can finish the mobile wizard. I keep both levels rather than treating a passing build as proof that an inspection works end to end.

[QRFleet](https://qrfleet.com)
