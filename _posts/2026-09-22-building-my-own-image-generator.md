---
title: "I replaced a template API with a small HTML-to-PNG image generator"
date: 2026-09-22 12:00:00 +0200
categories: [Desarrollo, Automatización]
tags: [python, chromium, automation, html, css]
image: /assets/img/posts/brand-engine-cover.png
---

The Projects page on this site used to describe an image pipeline built around Bannerbear. That description was out of date. I wanted to change a layout without going back to a template editor or depending on an image API, so I made a small generator that renders HTML with Chromium instead.

It produces newsletter headers, LinkedIn images and carousel slides. The cover of this post came out of the same script.

## The part that does the work

There are three HTML templates and a Python CLI. The script fills placeholders in a template, writes a temporary HTML file and asks headless Chromium to take a screenshot. Newsletter headers and carousel slides are 1080 × 1350 pixels; the LinkedIn image is 1200 × 630.

A single image can be generated with:

```bash
python3 generate.py --mode linkedin \
  --edition "SaskullOps" \
  --title "Building my own image generator" \
  --date "September 2026" \
  --output brand-engine-cover.png
```

For a carousel, the input is a JSON array of slides. The script currently understands title, bullet, code and call-to-action slides, then renders each one as a separate PNG. The typography and colours live in CSS, where I can edit them directly.

I also added a small HTTP endpoint so an n8n workflow can request images. That integration is an entry point, not a public service: the server uses Python's basic `http.server`, has no authentication and listens on all interfaces. I would not expose it to the internet as it stands. It also needs proper supervision to survive a reboot.

## Why keep it this small?

I wanted control over the output and a way to generate assets locally. This does both. It also means I own the awkward bits: font installation, text that overflows a slide, and checking the result before using it. HTML-to-PNG does not decide whether a composition looks good.

The current layouts are a starting point, not a promise that every post will look different automatically. I still need to adjust the content and composition rather than feeding a new headline into the same template every time.

At least the workflow now matches how I like to work: edit the HTML/CSS, run the command, inspect the PNG, change what needs changing.
