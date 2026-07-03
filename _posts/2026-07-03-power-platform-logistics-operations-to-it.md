---
title: Power Platform in logistics — What I learned going from operations to IT
date: 2026-07-03 00:00:00 +0200
categories: [Automatización, Aprendizaje]
tags: [power-bi, power-automate, power-apps, sql, logistics, career]
---

I started in operations — loading docks, schedules, the daily chaos of freight movement. When I moved into IT, I knew the problems intimately. That made the tools easy to choose.

This is what I built, how I built it, and what I'd tell someone starting from the same place.

## Power BI — dashboards that get used

The fleet runs on Excel. The problem isn't data — it's making sense of it across a dozen spreadsheets with different formats, naming conventions, and levels of completeness.

**Shipments by Volume — monthly view:**
Volume trends per client over 3 years, with windowed analysis by year, quarter, month, and week. Filters by origin, destination, and zone. The semantic model runs on clean SQL views — no transformations in Power BI, just well-structured data from the start.

**Performance — on-time vs delayed:**
Weekly breakdown showing on-time delivery rate per client and location. The metric that actually matters to operations. Updated weekly via a Power Automate flow that queries the database, builds an HTML table, and emails the ranking to stakeholders.

**Delay reason analysis:**
A line chart showing delay trends over time with a stacked ranking of root causes. Turned "why are we always late" from a gut feeling into a data question with answers.

I use DBeaver to build the semantic model. Named conventions, documented columns, incremental refresh on large tables. The SQL layer is the most important part — no visualization fixes bad data.

## Power Automate — 9 flows in production

These are the patterns that do the actual work:

**Bookings pipeline (create/update/cancel):**
Three flows triggered by calendar events. A new booking logs to Excel and SharePoint, sends a confirmation email, and creates a calendar event. Updates and cancellations follow the same pattern with appropriate notifications. The triad keeps the dataset in sync automatically.

**HR onboarding (ALTA):**
The biggest flow. Triggered by a new SharePoint item, it runs through approval stages (manager → HR), then auto-generates 16 document types using Word templates, creates a folder structure, converts documents to PDF, sends onboarding emails, and creates calendar events. Multiple approval stages with conditional routing based on employee type.

**Weekly performance pulse:**
SQL query → HTML table with color-coded rankings → branded email with company signature. No dashboards to check — the data arrives in the inbox.

**Monitor flows:**
Running daily — insurance expiry tracking (multiple insurers with different renewal cycles), HR dates (birthdays, passports, medical checks, CAP certificates, licenses, A1 forms, tachograph expirations). Each has empty-result handling so no alert means nothing to report.

**Training authorization:**
Takes a Word template, populates it from SharePoint data, converts to PDF, and emails it to the driver. Replaced a manual document generation process that took 15 minutes per driver.

## Power Apps — RBConnectHub

A container scanning app I built before QRFleet. Drivers scan a QR code on a container, the app shows vehicle info and lets them start the reporting process. It works within the Microsoft ecosystem but has clear limits:

- UI customization is constrained
- Image handling requires workarounds
- Performance degrades as the dataset grows

That experience directly led me to build QRFleet with FastAPI + Next.js for full control.

## What I'd tell someone starting

1. **Know the business problem before choosing the tool.** I built better dashboards because I lived the process. Power BI answered questions I already knew were worth asking.

2. **Power Automate is deceptively simple.** A flow that runs fine at 10 executions a day will break at 200. Think about scale — API limits, concurrency, error handling, logging. Every flow should log its executions somewhere you can debug later.

3. **SQL is the bottleneck.** If your semantic model is weak, no visualization will save it. Invest time in clean views, documented columns, and consistent naming. It pays for itself in the first week.

4. **Use AI as a force multiplier.** I use language models for DAX expressions, HTTP request templates, regex patterns, and debugging flows. Not as a replacement — as a tool that removes friction from the repetitive parts.

5. **Your operations background is an advantage.** You know what questions matter. Most people learn the tools and then try to find problems. You already know the problems.

## What's next

A unified view that combines BI (what happened), real-time ops (what's happening now), and alerts (what needs attention) in one interface. The pieces are all there — Power BI, Power Automate, SQL, and maybe n8n for the integration layer. It's about connecting them cleanly.