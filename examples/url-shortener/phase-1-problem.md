# Phase 1 -- Problem Statement

## The Problem

Users share long, ugly URLs that break in emails, messages, and social media. A URL shortener converts long URLs into short redirects. Without this, users resort to third-party services with no control over link reliability, analytics, or data privacy.

## Gate Answers

**What breaks if this isn't built?**
Teams keep using third-party shorteners. Marketing can't track campaign clicks reliably. Long URLs break in plain-text emails and get truncated by SMS gateways. IT has no way to kill a malicious shortened link shared internally.

**Why is code the right solution?**
A process change doesn't fix URL truncation -- that's a technical problem. Existing tools either lack self-hosting (so data leaves your control) or charge per feature you actually need. A simple API covers the core need in a few hundred lines.

## Alternatives Considered

| Alternative | Why rejected |
|---|---|
| Google's URL shortener (discontinued, was go.gl) | Shut down 2019. Even when active: no self-hosting, no data ownership, no custom domains |
| Bitly | Works fine until you want custom domains or API access -- then it's $348/yr minimum. Analytics data lives on their servers |
| Manual nginx redirect rules | Works for 10 links. Doesn't scale. No analytics. Every new link requires a deploy |
| Just paste the long URL | Breaks in email clients, SMS, Slack previews. Looks unprofessional in printed materials |
