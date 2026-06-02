---
layout: post
title: "The Curious Case of Wanting to Be SUPER and Read-Only at the Same Time"
date: 2026-06-02
category: business-central
tags: [OAuth, EntraID, BusinessCentral, AL, Security, ClaudeCode]
description: "When you want SUPER permissions in the Business Central UI but read-only access for Claude Code — and why Business Central has strong feelings about this."
---

## Problem Statement

I am integrating Claude Code with Microsoft Dynamics 365 Business Central SaaS using OAuth Device Authorization Flow.

The idea seemed simple:

> "I want to keep my SUPER permissions in Business Central, but I only want Claude Code to have read-only access."

As it turns out, Business Central security has a rather different opinion on the matter.

---

## The Dream

The desired architecture looks something like this:

```text
Maksims in BC UI
    ↓
SUPER permissions

Maksims in Claude Code
    ↓
Read-only permissions
```

A beautiful vision.

Unfortunately, Business Central doesn't support security models based on hopes, dreams, or personal preferences.

---

## What Actually Happens

When using OAuth Device Flow (Delegated Authentication), Business Central sees:

```text
User = Maksims
Permissions = SUPER
```

It does not see:

```text
User = Maksims
Application = Claude
Mood = Responsible
Permissions = Read Only
```

Business Central evaluates permissions based on the authenticated user.

And if that user is SUPER...

```text
SUPER = SUPER
```

Everywhere.

Every time.

No exceptions.

---

## The Harsh Reality

Delegated authentication is essentially:

```text
"This application is acting as the signed-in user."
```

Therefore:

| Access Method | Identity Seen by BC | Effective Permissions |
|---|---|---|
| BC Web Client | Maksims | SUPER |
| Postman | Maksims | SUPER |
| PowerShell | Maksims | SUPER |
| Claude Code | Maksims | SUPER |
| Random Script Written at 2AM | Maksims | SUPER |

Business Central does not care whether the request originates from:

- Claude Code
- Postman
- VS Code
- PowerShell
- A script created after three coffees and no sleep

If the token belongs to a SUPER user:

```text
The request is SUPER.
```

---

## Things That Will Not Work

Several tempting ideas often appear during this discussion.

### ❌ Restrict Through API Pages Only

```al
InsertAllowed = false;
ModifyAllowed = false;
DeleteAllowed = false;
```

Useful?

Yes.

Security boundary?

No.

You are simply making one doorway read-only while leaving the master key in your pocket.

---

### ❌ Restrict Through Entra Scopes

Business Central is not Microsoft Graph.

Fine-grained delegated scopes do not magically convert a SUPER user into a read-only user.

---

### ❌ Detect Claude Code Inside AL

Business Central permissions are evaluated before your AL code gets a chance to be clever.

No amount of creative coding can convince BC that:

> "This SUPER user is only SUPER on alternate Wednesdays."

---

### ❌ Trust the AI

A surprisingly popular architecture:

```text
Give AI full administrator access

↓

Hope for the best
```

This is generally considered a suboptimal security strategy.

---

## What Actually Works

### Option 1 — Separate User

Create a dedicated integration account:

```text
claude@company.com
```

Assign only the required read permissions.

Authenticate Claude Code using that account.

Done.

Simple.

Boring.

Effective.

---

### Option 2 — Service-to-Service Authentication

The grown-up solution.

Instead of impersonating a human:

```text
Claude → Entra App → Business Central
```

Assign permissions directly to the application.

Benefits:

- Proper security boundary
- Separate identity
- Principle of least privilege
- No accidental ERP apocalypse

---

## Additional Hardening

Even with correct authentication, API pages should still be locked down:

```al
page 50100 MyApi
{
    PageType = API;

    InsertAllowed = false;
    ModifyAllowed = false;
    DeleteAllowed = false;
}
```

This is defense-in-depth.

Not the primary security model.

Think seatbelt, not brakes.

---

## Final Verdict

The request can be summarized as:

> "I would like to be God in the Business Central UI, but merely a humble observer when using Claude Code."

Business Central politely declines.

Delegated authentication does not support split personalities.

You are either:

```text
SUPER everywhere
```

or

```text
Not SUPER everywhere
```

There is no:

```text
SUPER in UI
READ ONLY in Claude
MODERATELY DANGEROUS ON WEEKENDS
```

security model.

---

## Recommended Architecture

```text
┌─────────────────────────┐
│ Maksims                 │
│ Business Central UI     │
│ SUPER                   │
└───────────┬─────────────┘
            │
            │
            ▼

┌─────────────────────────┐
│ Claude Integration User │
│ Read Only Permissions   │
└───────────┬─────────────┘
            │
            ▼

┌─────────────────────────┐
│ Read-Only API Pages     │
│ Insert = false          │
│ Modify = false          │
│ Delete = false          │
└─────────────────────────┘
```

Or even better:

```text
┌─────────────────────────┐
│ Claude Code             │
└───────────┬─────────────┘
            │
            ▼

┌─────────────────────────┐
│ Entra Application       │
│ Service-to-Service Auth │
└───────────┬─────────────┘
            │
            ▼

┌─────────────────────────┐
│ Business Central        │
│ Read-Only Permissions   │
└─────────────────────────┘
```

---

## Executive Summary

> You cannot be simultaneously reckless and safe using the same identity.
>
> Split the identity.
>
> Or accept the chaos.
>
> Business Central certainly will.
