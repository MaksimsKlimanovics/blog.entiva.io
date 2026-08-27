---
layout: post
title: "ECHO: I Spent My Nights Building a Music App That Refuses to Make You a Playlist"
description: "Or: what happens when you hand an opinionated product idea to Claude Code, tell it never to build a Spotify clone, and let a Codex review loop keep it honest."
date: 2026-08-26
category: ai-copilot
tags: [AI, Claude Code, Codex, Next.js, Prisma, Product, Music, Agents, Side Project]
---

I build ERP systems and AI agents for a living. Business Central during the day, Fiona and Anatoly and Anna in whatever's left of the evening.

Echo is none of those.

Echo is a side project about music. Except it turned into one of the more interesting engineering stories I've had this year, so here we are.

> We don't make playlists. We build paths through music.

That line is in the README. I didn't soften it for the blog. It's the whole thesis.

![Echo's entrance screen: "Music for where you are — and where you want to go," over a dusk lake photo]({{ "/assets/images/echo/shot-entrance.webp" | relative_url }})
*Echo's actual entrance screen. "Every journey begins somewhere."*

---

# The Itch

Every music app I've used eventually asks the same question: *what genre are you in the mood for?*

Which is a strange question, because I am never "in the mood for indie folk." I'm in the mood for the version of me that exists at 11pm on a Tuesday, slightly tired, thinking about a conversation from four years ago, wanting to feel something specific and not entirely sure what to call it.

Genre doesn't cover that. A shuffled "Chill Vibes" playlist doesn't cover that either.

So Echo asks three different questions instead:

1. **Where have you been?** — musical history, formative songs, the tracks that are basically load-bearing walls in your identity.
2. **Where are you now?** — the actual emotional weather, not a mood emoji.
3. **Where do you want to go?** — stay here, go deeper, release it, escape, rise, find peace, or just don't tell me and let the system carry you.

The output isn't a playlist. It's a **Journey** — an ordered, six-act emotional arc, built from those three coordinates, with the order itself being part of the recommendation.

There are three doors in: **Journey** ("tell me where you are"), **Dream** ("tell me what you see"), **Memory** ("tell me where you want to return"). No genre picker anywhere near the entrance.

![Echo's three-doors screen: "Where do we begin?" with Journey, Dream, and Memory as the options]({{ "/assets/images/echo/shot-doors.webp" | relative_url }})
*"Where do we begin?" No fourth door, no genre picker hiding behind it.*

---

# The Architecture Nobody Asked Me to Build This Carefully

A Journey has six acts, and they're not decorative:

```text
ARRIVAL → DESCENT → MOTION → TENSION → RELEASE → AFTERGLOW
```

Sixteen tracks, deterministically composed. Each direction (`deeper`, `release`, `escape`, `rise`, `peace`, `surprise`, `stay`) maps to its own intensity curve across those six acts. "Deeper" climbs to `.88` at RELEASE. "Peace" tops out at `.45` and never sounds like it's trying to prove anything.

The part I actually care about, architecturally, is this rule from the project's own `AGENTS.md`:

> Keep recommendation *sequencing and scoring* separate from provider integrations and LLM interpretation. An LLM may propose which real songs are even candidates — but it never chooses the final sequence, act placement, or which of its own suggestions actually get used. The deterministic engine alone owns that.

In practice: an LLM is allowed to suggest "here are 32 real songs that might fit this person's dream description." It is not allowed to decide what order they play in, when the emotional peak lands, or how much repetition an artist gets. That's a plain deterministic scoring function — familiarity, DNA compatibility, transition continuity between tracks, an artist-repetition penalty — doing arithmetic, not vibes.

There's also a rule I'm oddly proud of, straight out of the product bible:

- 60–70% safe picks
- 20–30% exploration
- 5–10% wildcard

The wildcard slot exists for exactly one reason: the moment someone thinks *"how did it know I needed this?"* That's not a metaphor, it's a literal tier in the scoring code (`TIER_SLOT_OVERRIDES`), landing at 68.75/25/6.25 in the actual implementation — inside spec, deterministic, no `Math.random` anywhere. Reproducible weirdness. My favorite kind.

![Echo's Musical DNA screen: "Give me five songs you would hate to lose," with anchor tracks listed]({{ "/assets/images/echo/shot-dna.webp" | relative_url }})
*Musical DNA onboarding. Anchor tracks, not a genre checklist — mine currently reads Korn, Disturbed, TesseracT, Lorna Shore, Whitechapel.*

---

# Building This With an AI Agent Watching Its Own Homework

Here's the part that actually made me want to write this post.

I didn't just use Claude Code to write Echo. I used it in phases, with a running project status document, and — this is the interesting bit — I had **Codex review its own pull requests**, and I made it fix what Codex found.

Some of what got caught was not cosmetic:

- **`POST /api/journeys` was blocking the HTTP response on a database write.** A slow Postgres meant a slow "creating your journey" spinner, even though the journey itself was already built in memory. Fixed by pushing the save into Next's `after()` — the response goes out immediately, persistence happens once the client already has its answer.
- **Returning visitors got permanently stuck replaying their last Journey.** There was no way back to the entrance screen. Nobody noticed until an automated review pointed out the obvious: if the only exit is closing the tab, that's not a feature.
- **A purchase could be marked "completed" before fulfillment actually ran.** If the composer threw partway through building someone's paid "Odyssey" extension, the payment record said *done* while nothing was delivered — and worse, nothing could retry it. That one got an actual state machine: `pending → fulfilling → completed`, with a forced revert to `pending` on any failure, and the Stripe webhook now deliberately returns `500` on that revert so Stripe's own retry logic does the reconciliation for me.
- **Someone could buy the same gift Journey twice.** Nobody checked for an existing `Gift` row before charging a second time. Now they do.

That last category is my favorite kind of bug, because it's not a typo — it's a business rule nobody said out loud until a payment flow made it real.

And then there's the finding I'm keeping for the honesty of it: I asked the sandbox to verify the real Stripe integration end-to-end, and it correctly discovered that the sandbox's own network egress doesn't reach `api.stripe.com` at all. Not a code bug. A wall. The commit history has a very calm line about it: *"even after real credentials are added, a live checkout still can't be created from inside this session."* I appreciated the honesty more than I would have appreciated a fake green checkmark.

---

# The Catalog Ran Out of Songs, Which Is a Very Real Problem

At one point Echo had two paid tiers — "Deeper" (48 tracks) and "Odyssey" (96 tracks) — sitting on top of a curated catalog of exactly 35 songs.

You can see where this goes. Someone could pay for a 96-track Odyssey and receive 35 tracks and then... silence. Gracefully degraded silence, because the composer is built to never crash and never duplicate a track, but still: silence someone paid for.

The fix wasn't clever. It was 70 more hand-picked, correctly-attributed real songs (no invented tracks — that's a hard rule), and a `catalog_too_small` check that now blocks the purchase *before* anyone gets charged for a Journey the catalog can't actually deliver.

Then the product direction changed again, because it should: tracks aren't hardcoded anymore at all. Every Journey now asks an LLM to propose ~32 real candidate songs live, per request, based on that specific person's answers — and the deterministic engine still does 100% of the sequencing on top of whatever comes back. The AI got a bigger job. It still doesn't get to pick the ending.

---

# Where the Money Doesn't Come From

Echo's monetization page is one paragraph I'd frame:

> The person must be able to experience a complete useful Journey without creating an account or starting a subscription. Payment buys depth or a special experience, not relief from artificial frustration.

Free anonymous Journey, no signup wall, no ads anywhere near the experience, no deliberately nerfed free tier to manufacture upgrade pressure. If you want to pay, you're paying for **Deeper**, **Odyssey**, gifting a Journey to someone else, or just "Keep ECHO alive" — three fixed amounts and a custom field, dismissible with "Not now," never shown before someone's actually finished a Journey.

It's not a growth-hacking monetization model. It's closer to a tip jar with better production values. I'm fully aware that's a harder business than a subscription paywall. I built it this way on purpose anyway.

---

# What's Still Genuinely Unfinished

I'm not going to pretend this is done, because the project's own status doc doesn't pretend either — which is exactly why I keep it:

- Real OpenAI, Spotify, and Stripe *success* paths are only verified against mocks in this environment. The fallback paths are proven live. The happy paths need real credentials outside a sandboxed container.
- Google OAuth login is wired and unverified for the same reason.
- Component-level UI tests don't exist yet — this codebase tests routes live against a real Postgres instance instead, which is a defensible choice for a solo project and a slightly terrifying one for anything bigger.

None of that is spin. It's the same discipline I'd want from anyone reviewing my Business Central integrations: tell me what you actually ran, not what you assume works.

---

# Why I'm Telling You This

I spent an entire post a couple of months back being nervous about what AI agents mean for developers like me. Echo is the other half of that story.

An agent didn't design Echo's emotional model, its monetization philosophy, or the rule that a wildcard track should feel like it knows you. That part was mine, stubbornly, the whole way through.

What the agent — and a second agent reviewing the first agent's work — actually did was turn "sequencing should never trust the LLM's suggestions blindly" from a sentence in a design doc into an enforced architectural boundary, catch a payment bug before it shipped, and be honest when a network sandbox simply couldn't reach Stripe.

That's not "AI replaced the developer." That's a very good pair of hands that also happens to leave a paper trail.

I'll still keep my debugger open. Echo just gave it slightly better company.

You can try it yourself at [echoes-app.xyz](https://echoes-app.xyz). It builds paths through music. It does not, on principle, ask what genre you're in the mood for.

---

A quick credits line, because it feels dishonest not to include one on a post about AI-built things: Echo's visual design was made with Claude Design, and its background imagery was generated with GPT-5, styled around my Finnish Lapphund, Lorna.

![Lorna, a four-month-old Finnish Lapphund puppy, sitting on a sandy lakeshore]({{ "/assets/images/echo/lorna-photo.jpg" | relative_url }})
*The real Lorna. Four months old, named after Lorna Shore — my favorite band, and, purely by coincidence of good taste, one that made it into Echo's own catalog three separate times (`lorna-pain-1`, `lorna-pain-3`, `lorna-hellfire`, tagged `grief`, `climax`, and `cathartic` respectively). She did not review those pull requests. I checked.*
