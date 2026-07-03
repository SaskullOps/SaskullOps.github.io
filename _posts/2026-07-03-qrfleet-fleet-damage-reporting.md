---
title: QRFleet — Building a production fleet damage reporting app
date: 2026-07-03 00:00:00 +0200
categories: [Desarrollo, Automatización]
tags: [fastapi, next.js, docker, postgresql, qrfleet, supabase]
image: /assets/img/posts/qrfleet-cover.png
---

## The problem

Drivers reporting damage via WhatsApp. Photos lost in the chat, descriptions misunderstood, no traceability. Management needing a dashboard that didn't exist.

The requirement was simple: scan a QR code on a vehicle, take a photo, describe the damage, and have it land in a dashboard instantly. No paper, no WhatsApp, no back-and-forth.

I'd already built RBConnectHub in Power Apps (a container/QR scanning app) and hit its limits. Power Apps is great for rapid prototyping inside Microsoft ecosystems, but this needed to feel like a real application — custom UI, proper image storage, version control, full control over the stack.

I made a conscious call: build it from scratch.

## The stack

```
Frontend:  Next.js (SSR, dynamic routes, i18n ES/EN)
Backend:   FastAPI (JWT auth, REST endpoints, photo pipeline)
Database:  PostgreSQL (relational model: fleet, drivers, reports, incidents)
Storage:   Supabase bucket (photos)
Deployment: Docker Compose on Hetzner VPS — behind Nginx Proxy Manager
Domain:    qrfleet.com
```

## Architecture

The app has four core models:

- **Company** — multi-tenant from day one
- **Fleet** — vehicles with unique QR codes
- **Drivers** — linked to company, assigned to vehicles
- **Reports** — damage reports with photos, status tracking, history

Reports flow through states: `draft → submitted → reviewed → resolved`. Each transition triggers a notification to the relevant party.

### The photo pipeline

Photos are the core feature. The pipeline:

1. Frontend compresses the image client-side before upload
2. Backend validates file type, size, and dimensions
3. EXIF metadata stripped before storage (privacy)
4. Two versions stored: full-res (for review) and thumbnail (for lists)
5. Signed URLs for access — no public bucket

This was harder than it sounds. JPEG encoding mismatches between the browser's compression and Python's Pillow library caused silent failures for weeks. The fix was explicit format enforcement on both sides.

## QR integration

Each vehicle gets a unique QR code generated server-side. Scanning opens `qrfleet.com/report?vehicle={id}` — the app loads with the vehicle context pre-filled. No login screen for drivers, just camera → scan → report.

The QR generation uses a lightweight Python library, no external service. Every time the fleet changes, the QR codes regenerate automatically.

## Auth

Simple JWT with refresh tokens. No Google/GitHub OAuth, no third-party provider. Two roles:

- **Driver** — can create reports for their assigned vehicle
- **Admin** — full access, dashboard, user management

Token rotation every 15 minutes, refresh tokens with 7-day expiry. Stored in httpOnly cookies.

## What broke in production

**CORS.** The first deployment worked locally. On the server, the frontend couldn't reach the API because CORS headers weren't configured for the production domain. Classic.

**Upload timeouts.** Large photos taken with modern phones (12MP+) would timeout the default 30-second upload window. Solution: client-side compression + backend timeout bump to 120s for the upload endpoint.

**The rebrand.** RB Hub → QRFleet meant renaming across the entire stack: database schema, API routes, frontend routes, email templates, Docker service names, Supabase bucket, GitHub repo. Took a full afternoon. Name things well the first time.

## What I'd do differently

- **Version the API from day one.** `/api/v1/` costs nothing and saves a major refactor later.
- **Add request logging earlier.** I added structured logging (request ID, duration, endpoint, user) after the first bug hunt. Should have been there from the start.
- **Better error messages on the frontend.** "Something went wrong" is useless. Now every API error returns a user-facing message in the correct language.

## The repo

The codebase is open on GitHub. It's not perfect — there are things I'd refactor — but it works, it's in production, and drivers use it daily.

[GitHub — SaskullOps/rb-hub](https://github.com/SaskullOps/rb-hub)
[Live — qrfleet.com](https://qrfleet.com)