---
layout: post
title: "BC TechDays 2026: The Keynote That Made Me Question My Career (In the Best Possible Way)"
description: "Or: Why a guy who spends every day building AI solutions still walked away thinking, 'Well... this escalated quickly.'"
date: 2026-06-14
category: business-central
tags: [Business Central, AI, Copilot, Agents, MCP, Claude Code, AL, BC TechDays, Microsoft]
---

# BC TechDays 2026: The Keynote That Made Me Question My Career (In the Best Possible Way)

> *Or: Why a guy who spends every day building AI solutions still walked away thinking, "Well... this escalated quickly."*

I've spent the last year building AI integrations for Business Central.

Claude Code.
MCP.
Business Central APIs.
Custom AI agents.
Autonomous workflows.

I'm probably one of those annoying people constantly telling colleagues:

> "No, seriously... you should try AI."

So when BC TechDays 2026 started talking about agents, I expected a few marketing slides, another Copilot demo, and the obligatory *"AI will help you be more productive."*

Instead, Microsoft casually spent an hour explaining why the way we've been developing Business Central for the last twenty years is about to change forever.

And honestly?

That scares me.

Not because AI is bad.

Because this time...

I think they're actually right.

---

# Remember When Copilot Was Cute?

Back in 2024 Copilot felt like Clippy after a protein shake.

It suggested marketing text.

It filled in fields.

Sometimes it even generated something useful.

It was basically autocomplete wearing an expensive Microsoft suit.

We still owned the workflow.

The AI simply handed us a few suggestions and politely stepped aside while we did the real work.

Life was good.

Then 2025 happened.

- Autofill
- Summaries
- Sales Order Agent
- Payables Agent

Still manageable.

The AI was helping.

Now?

Microsoft isn't talking about assistants anymore.

They're talking about coworkers.

Actually...

They're talking about employees that don't complain, don't need holidays, don't ask for salary reviews, and apparently don't spend half the day arguing about whether tabs are better than spaces.

Rude.

---

# The Slide That Changed Everything

The most important slide in the keynote wasn't about Business Central.

It wasn't about Copilot.

It wasn't even about AL.

It was a graph showing something called the **time horizon**.

How much work an AI model can reliably perform before losing context.

| Model | Human-equivalent work |
|--------|-----------------------|
| GPT-3.5 | ~15 seconds |
| GPT-4 | ~53 seconds |
| OpenAI o1 | ~4 minutes |
| OpenAI o4 / Claude Opus 4 | ~20 minutes |
| Gemini 3.1 Pro | ~90 minutes |

That sounds like a random statistic.

It isn't.

It's the reason AI agents suddenly became practical.

For years we've asked:

> "Why can't AI just process the whole workflow?"

Because it literally couldn't.

Now it can.

That's a completely different world.

---

# The Moment I Started Feeling Slightly Uncomfortable

Then Microsoft showed Aptian's Manifest Agent — an autonomous AI agent that processes incoming shipping documents end-to-end without human intervention.

The agent:

- Watches an inbox.
- Reads PDF manifests.
- Extracts structured data.
- Creates purchase orders.
- Logs every action.
- Asks for help only when it genuinely needs it.

Twenty minutes of repetitive office work.

Reduced to two.

No flashy animations.

No exaggerated promises.

Just...

Done.

As a developer, my first reaction wasn't:

> "That's amazing."

It was:

> "...well... there goes another customization project."

---

# Congratulations, You're No Longer a Developer

You're now an AI supervisor.

Microsoft introduced the new Agent Designer inside Business Central.

You don't start by creating tables.

You don't start by writing AL.

You don't even start by opening VS Code.

Instead, you define:

- What the agent can see.
- What it's allowed to do.
- Which tools it can use.
- What instructions it follows.
- When it should ask a human.

Sound familiar?

That's because our job is quietly changing.

We're moving from writing algorithms...

to designing intelligent workers.

Eventually you export everything into AL.

Ironically...

Even the AL project is generated for you.

---

# MCP Is Quietly the Biggest Announcement

Everyone is talking about agents.

I think the real story is MCP.

Business Central is no longer just an ERP.

It's becoming an AI platform.

Expose your APIs.

Expose your queries.

Expose your business logic.

Now every AI agent—whether it's running in Copilot Studio, Claude Code, GitHub Copilot, VS Code, or something that hasn't even been invented yet—can interact with your ERP as if it were another developer.

As someone who's already spent months building Business Central APIs, and AI integrations, and even my own MCP Server for Garage Hive...

This was probably the most exciting part of the keynote.

Because Microsoft isn't fighting the ecosystem anymore.

They're embracing it.

And honestly?

That makes me happy.

---

# The Part That Actually Scared Me

Microsoft kept repeating one sentence:

> "AI won't replace developers."

Technically...

They're right.

But I think that's also a very comfortable sentence to hear.

Here's what I actually believe.

AI probably won't replace developers.

Developers using AI will replace developers who don't.

That's a very different statement.

Because look at what is already becoming a commodity:

- CRUD pages
- Boilerplate AL
- Documentation
- Test generation
- API wrappers
- OCR
- Data extraction
- Repetitive integrations
- Workflow automation

That's a surprisingly large percentage of many Business Central projects.

---

# Why I'm Actually Scared of Losing My Job

Not because an AI agent will wake up tomorrow morning, walk into my office, and ask HR for my laptop.

That's not how this ends.

What scares me is something much more subtle.

For almost twenty years, Business Central developers have been valued because we knew **how** to build things.

How to write AL.

How to expose APIs.

How to debug.

How to integrate systems.

How to automate processes.

Suddenly, **knowing how** is becoming the least interesting part of the job.

AI is getting frighteningly good at implementation.

The value is shifting towards **knowing what to build**, **why**, and **whether it should even exist**.

That means the keyboard is no longer the bottleneck.

Thinking is.

My biggest fear isn't losing my job.

It's becoming obsolete while still having one.

Because the industry won't wait.

The companies embracing AI won't wait.

And neither will the developers who learn how to orchestrate ten agents while I'm still arguing about naming conventions.

---

# So... What Actually Remains Valuable?

If AI keeps taking over implementation, what stays human?

I think it's this:

- Architecture.
- Business understanding.
- System design.
- Security.
- Integration strategy.
- Debugging AI hallucinations.
- Designing agent workflows.
- Knowing when AI is confidently wrong.
- Asking better questions than everyone else.

Ironically, prompt engineering was never the destination.

Good thinking was.

AI simply removed the illusion that typing fast equals being valuable.

---

# Final Thoughts

I don't think we're witnessing another Business Central release.

I think we're watching the beginning of a completely different profession.

Business Central is no longer evolving around pages, reports, and codeunits.

It's evolving around autonomous business processes.

Developers who only write AL will slowly become less valuable.

Developers who understand business, architecture, AI, MCP, integrations, and how to orchestrate intelligent systems...

They're going to build the next generation of Business Central solutions.

That's both terrifying...

and unbelievably exciting.

I'll keep writing AL.

Probably for quite a while.

But I have a feeling that in five years, writing AL will be the smallest part of what makes someone a great Business Central developer.

And yes...

I'll still keep my debugger open.

Just in case one of my agents decides it no longer needs me.
