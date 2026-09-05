# Automated Lifecycle Email System for a Cross-Border E-Commerce Logistics Operation
**`n8n` · `PostgreSQL` · `LLM API` · `email gateway`**
 
> A self-running marketing pipeline that segments customers by behaviour and lifecycle stage, then generates and sends personalised, LLM-written emails per segment, fully automated, owned by one person.
 
---
 
## The problem
Customers moved through a multi-stage shipment lifecycle (order placed → warehouse → shipped → at port → delivered) across several markets and account types (repeat buyers, dormant users, agents vs. direct, companies vs. individuals). Marketing was ad-hoc, and the pain was twofold: **targeting**, nobody could answer "who's inactive right now?" or "who just had an order delivered?" without a manual query, and **content**, writing tailored copy per segment per week doesn't scale for one marketer. The system makes lifecycle marketing a standing, hands-off capability instead of an occasional manual effort.
 
## What I built
Two things at once: a scheduled classifier that keeps an always-current segment table, and an LLM that generates the right message for each segment automatically. One person can own a marketing system that previously would have needed constant manual querying and copywriting.
 
## Architecture
**Two decoupled layers.** Segmentation decides *who*; execution decides *what to send* and sends it. They communicate only through one table (`user_segments`), so either can change without touching the other.
 
```
SEGMENTATION LAYER (nightly, batch)
 Schedule → parallel SELECT branches (one per sub-segment)
 → Merge → UPSERT into user_segments
 (INSERT ... ON CONFLICT DO UPDATE, preserves cooldown clock)
 → targeted DELETE (drop users who left the segment)
 │ writes
 ▼
 PostgreSQL: source tables + user_segments (single source of truth)
 │ reads
 ▼
EXECUTION LAYER (daily, fixed UTC)
 Schedule → config (segment, cooldown, test_mode)
 → fetch eligible recipients (cooldown + per-user throttle filters)
 → IF recipients? ──no──► clean exit
 → build prompt (segment intent + shared brand context)
 → ONE LLM call → structured JSON {subject, body, cta_token}
 → build per-recipient items (dedupe, map cta_token → trusted URL)
 → IF live? → send one-at-a-time via gateway → mark last_sent_at
 └ shadow mode → log generated content, no send
```
 
~14 named segments are produced by 6 segmentation workflows; every segment query emits the same shape (`user_id, email, segment_name`) and applies a standard customer filter. The LLM is called **per campaign, not per recipient**, via a structured-output endpoint grounded in a shared brand-context block of real facts.
 
## Engineering decisions worth discussing
- **Two-layer split with a segment table as single source of truth.** Segmentation writes `user_segments`; execution reads it. "Who's in this segment right now?" becomes one SELECT, and either layer evolves independently. The tempting shortcut, running the segmentation SQL inside the campaign, was deliberately avoided because it destroys debuggability and reuse.
- **Caught a "priority filter" that was silently suppressing audiences.** The first design used a CTE to pick each user's single "winning" segment, so a campaign for segment X only reached users whose *top* segment happened to be X, silently dropping most intended recipients. The system "worked" but emailed a fraction of who it should have; I caught it by reconciling expected vs. actual recipient counts, then removed priority-as-filter entirely via a weekday-spread model.
- **Cooldown strictly less than cadence.** A cooldown window equal to the send interval skipped users every other cycle, a send "exactly N days ago" still counted as inside the cooldown at the boundary. Fixed with a 5-day cooldown against a ~7-day cadence, plus an 18-hour daily throttle so a late-finishing job doesn't bleed into the next window. A time-boundary-conditions lesson in scheduled systems.
- **Made an unreviewed LLM safe enough to email customers.** Three sub-decisions: guardrails live in the *system* prompt (negative instructions in the user prompt got echoed verbatim into email bodies); the model returns a CTA *token* from a fixed enum and the workflow maps it to a hardcoded trusted URL, so the model can never emit a fabricated link; and a shared brand-context grounds copy in real facts. This is the core "AI automation" competency.
- **Idempotent segmentation via upsert, not wipe-and-reload.** The obvious refresh (delete all rows, re-insert) wiped the cooldown timestamp and caused silent double-sends. `INSERT … ON CONFLICT … DO UPDATE` refreshes membership without touching the cooldown clock, then a targeted delete removes only users who left.
## Impact
No production engagement numbers were captured. Open/click/conversion tracking was scoped as a later phase and never built, so the send log records sends, not outcomes. I won't present staging counts as impact. The honest, defensible framing today is **effort removed**: multiple segments × multiple sends/week that previously required hand-writing and hand-sending are now zero-touch. Metrics I'd pull to make this section concrete: per-campaign deliverability and engagement (already sitting in the email provider's dashboard), production recipient counts per segment per week, and (only if attributable) recovery-email conversion on abandoned orders.
 
## Tech stack
`n8n (self-hosted, Kubernetes)` · `PostgreSQL` · `LLM structured-output service` · `email/messaging gateway` · `SQL` · `JavaScript (n8n Code nodes)`
 
## Retrospective, what I'd do differently
- **Replace the send loop.** The split-batch → HTTP → wait pattern ties up the workflow for tens of minutes on a large campaign and is fragile to restarts mid-loop. I'd use the provider's batch API or a real queue + worker with bounded concurrency, and make each run resumable with a run ID.
- **Measurement from day one**, the biggest gap. Provider webhooks → an events table, wired before launch, so every campaign is measurable. You can't improve open rates you never recorded.
- **Consent handled through the shared notification service.** Emails went out through a central notification service that exposed an `includeUnsubscribe` option, so unsubscribe links and footers stayed consistent and compliant across every email type rather than being reimplemented per workflow. My pipeline set that option, keeping consent handling in the one service designed to own it. The natural next step I'd add is defence-in-depth on my own side, an explicit suppression check (`WHERE NOT EXISTS`) against opted-out users on every segment query, so consent is enforced both at send and at selection. Worth calling out explicitly for the German market, where DSGVO expects that full loop.
- **Config over cloning.** ~9–10 near-identical campaign workflows were cloned per segment/country; one parameterised workflow driven by a config table would cut the maintenance tax sharply.
- **Prompt versioning + evals.** Prompts were edited live; a versioned prompt store with a small eval set would make content changes testable.

**What I learned:** how to make an LLM safe enough to run unreviewed in production (system-prompt guardrails, structured output, token→URL substitution, factual grounding); how to design idempotent scheduled pipelines where re-runs don't double-act; the subtle time-window bugs that live in cooldown/throttle logic; and to trust the live schema over the docs every time.
