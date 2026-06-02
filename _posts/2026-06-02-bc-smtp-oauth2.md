---
layout: post
title: "Business Central SMTP OAuth2 with Exchange Online"
date: 2026-06-02
category: business-central
tags: [OAuth, EntraID, PowerShell, M365, SMTP, Exchange]
description: "Complete setup guide for SMTP OAuth2 authentication in Business Central using Exchange Online and a dedicated Entra App Registration — including the PowerShell commands that actually work."
---

*Or: How I Learned to Stop Trusting Error Messages and Love Exchange*

This guide documents the complete setup required to configure SMTP OAuth2 authentication for Business Central using Exchange Online and a dedicated Entra App Registration.

## Architecture

```text
Business Central
       ↓
Entra App Registration
       ↓
Exchange Service Principal
       ↓
Mailbox Permissions
       ↓
Exchange Online SMTP
```

## Install Exchange Online PowerShell

```powershell
Install-Module -Name ExchangeOnlineManagement -Force
```

Required before you can do anything else here.

## Connect to Exchange Online

```powershell
Connect-ExchangeOnline
```

## Create Exchange Service Principal

This is the step most guides skip entirely.

```powershell
New-ServicePrincipal `
    -AppId "<CLIENT_ID>" `
    -ObjectId "<ENTERPRISE_APPLICATION_OBJECT_ID>"
```

Creates Exchange's own representation of the Entra application. Without this, Exchange has no idea your app exists — regardless of what Graph says.

The `ObjectId` here is the **Enterprise Application** Object ID, not the App Registration Object ID. They are different. This is where most people lose an afternoon.

## Grant SendAs Permission

```powershell
Add-RecipientPermission `
    -Identity "<MAILBOX_EMAIL>" `
    -Trustee "<ENTERPRISE_APPLICATION_OBJECT_ID>" `
    -AccessRights SendAs
```

Without this, authentication succeeds but sending fails. Exchange will let you in and then refuse to cooperate.

## Verify SendAs Permission

```powershell
Get-RecipientPermission "<MAILBOX_EMAIL>" |
    Where-Object { $_.Trustee -like "*<PARTIAL_OBJECT_ID>*" }
```

## Verify SMTP AUTH is enabled on the mailbox

```powershell
Get-CASMailbox "<MAILBOX_EMAIL>" |
    Format-List SmtpClientAuthenticationDisabled
```

Expected:

```text
SmtpClientAuthenticationDisabled : False
```

If it says `True`, SMTP AUTH is disabled at the mailbox level. Enable it in the Exchange Admin Center or via `Set-CASMailbox`.

## Required API Permissions

In the App Registration, grant the following and hit **Grant Admin Consent**:

**Microsoft Graph**
- `User.Read` (Delegated)

**Office 365 Exchange Online**
- `Mail.Read` (Application)
- `Mail.ReadWrite` (Application)
- `Mail.Send` (Application)
- `SMTP.SendAsApp` (Application)
- `User.Read.All` (Application)

## Redirect URI

Add this to the App Registration under **Authentication**:

```text
https://businesscentral.dynamics.com/OAuthLanding.htm
```

Without it, Entra returns:

```text
AADSTS500113: No reply address is registered for the application.
```

That error message is technically accurate and completely unhelpful if you don't know what to look for.

## Troubleshooting

| Error | What it actually means |
|---|---|
| `AADSTS500113` | Missing Redirect URI in App Registration |
| `AADServicePrincipalNotFound` | Wrong Object ID — you used App Registration ID instead of Enterprise App ID |
| `Cannot open mailbox` | Missing mailbox permissions |
| `AuthenticationContext has no rights on this session` | Missing SendAs permission |
| `535 Authentication unsuccessful` | OAuth or Exchange configuration issue — check all of the above |

## Final Thoughts

Once everything is configured correctly, OAuth authentication succeeds, Exchange recognizes the Service Principal, mailbox permissions are assigned, and Business Central sends email without complaint.

At that point you may confidently tell your colleagues the setup was "straightforward" — while quietly forgetting how many browser tabs and error messages it actually took.
