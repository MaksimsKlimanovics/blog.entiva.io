---
layout: post
title: "Business Central SMTP OAuth2 with Exchange Online"
date: 2026-06-02
category: business-central
tags: [OAuth, EntraID, PowerShell, M365, SMTP, Exchange]
description: "Complete setup guide for SMTP OAuth2 authentication in Business Central using Exchange Online — including the parts Microsoft documentation conveniently omits."
---

Complete Setup Guide (Battle-Tested Edition)

After spending several hours fighting Entra ID, Exchange Online, Business Central and Microsoft's favourite game *"Let's return a completely unrelated error message"*, here's the complete setup guide that actually works.

This guide is intended for:

- Business Central SaaS
- SMTP Connector
- OAuth2 Authentication
- Exchange Online (Microsoft 365)
- Dedicated App Registration

---

## Architecture Overview

The final solution looks like this:

```text
Business Central
       |
       ↓
Entra App Registration
       |
       ↓
Exchange Service Principal
       |
       ↓
Mailbox Permissions
       |
       ↓
Exchange Online SMTP
       |
       ↓
Recipient
```

In plain English:

Business Central authenticates using an Entra Application. Exchange does not trust applications directly, therefore an Exchange Service Principal must be created. The Service Principal then receives permissions to act on behalf of a mailbox. Only after all three pieces are connected will email actually send.

---

## Step 1 — Create App Registration

Navigate to:

```text
Microsoft Entra Admin Center
→ App registrations
→ New registration
```

Settings:

```text
Name:                        Business Central SMTP
Supported account types:     Accounts in this organizational directory only
```

Register the application and record:

```text
Application (Client) ID:    <CLIENT_ID>
Directory (Tenant) ID:      <TENANT_ID>
```

---

## Step 2 — Create Client Secret

Navigate to:

```text
Certificates & Secrets
→ New Client Secret
```

Create a secret and immediately copy its value. Because Microsoft only shows it once. Because apparently showing secrets twice would destroy the internet.

Store it securely:

```text
Client Secret: <CLIENT_SECRET>
```

---

## Step 3 — Configure Redirect URI

Navigate to:

```text
Authentication
→ Add Platform
→ Web
```

Add:

```text
https://businesscentral.dynamics.com/OAuthLanding.htm
```

Save.

**Why?** Without it Business Central returns:

```text
AADSTS500113: No reply address is registered for the application
```

Even if everything else is configured correctly.

---

## Step 4 — Configure API Permissions

Navigate to **API Permissions** and add the following. Permissions must match exactly.

**Microsoft Graph**

| Permission  | Type      |
|-------------|-----------|
| `User.Read` | Delegated |

**Office 365 Exchange Online**

| Permission        | Type        |
|-------------------|-------------|
| `Mail.Read`       | Application |
| `Mail.ReadWrite`  | Application |
| `Mail.Send`       | Application |
| `SMTP.SendAsApp`  | Application |
| `User.Read.All`   | Application |

Grant **Admin Consent**. You should see green ticks for all permissions.

---

## Step 5 — Install Exchange Online PowerShell

```powershell
Install-Module -Name ExchangeOnlineManagement -Force
```

**Why?** This installs the Exchange Online PowerShell cmdlets. Without it, `Connect-ExchangeOnline` doesn't exist. Which is usually Microsoft's subtle way of saying:

```text
Command not found. Good luck.
```

---

## Step 6 — Connect to Exchange Online

```powershell
Connect-ExchangeOnline
```

Sign in using an Exchange Administrator account.

**Why?** Everything that follows is Exchange configuration. Entra alone is not enough. This is one of those wonderful Microsoft integrations where both teams apparently agreed:

> Let's each store half the configuration.

---

## Step 7 — Create Exchange Service Principal

First, locate the correct IDs from your App Registration:

```text
Application (Client) ID:              <CLIENT_ID>
Enterprise Application Object ID:     <ENTERPRISE_APPLICATION_OBJECT_ID>
```

**Important:** do NOT use the App Registration Object ID. Use the **Enterprise Application Object ID**. These are different. Ask me how I know.

```powershell
New-ServicePrincipal `
    -AppId "<CLIENT_ID>" `
    -ObjectId "<ENTERPRISE_APPLICATION_OBJECT_ID>"
```

**Why?** Exchange cannot use Entra applications directly. This creates an Exchange-side representation of the application. Without this step: authentication succeeds, email sending fails.

---

## Step 8 — Grant SendAs Permission

```powershell
Add-RecipientPermission `
    -Identity "<MAILBOX_EMAIL>" `
    -Trustee "<ENTERPRISE_APPLICATION_OBJECT_ID>" `
    -AccessRights SendAs
```

**Why?** This allows the application to send email as the mailbox configured in Business Central. Without it, Exchange throws:

```text
Cannot open mailbox
AuthenticationContext has no rights on this session
SendAsDenied
```

---

## Step 9 — Verify SendAs Permission

```powershell
Get-RecipientPermission "<MAILBOX_EMAIL>" |
    Where-Object { $_.Trustee -like "*<PARTIAL_OBJECT_ID>*" }
```

Expected:

```text
AccessRights : {SendAs}
```

**Why?** Trust. But verify. Especially when dealing with Exchange.

---

## Step 10 — Verify SMTP AUTH

```powershell
Get-CASMailbox "<MAILBOX_EMAIL>" |
    Format-List SmtpClientAuthenticationDisabled
```

Expected:

```text
SmtpClientAuthenticationDisabled : False
```

**Why?** If it says `True`, SMTP authentication is disabled for the mailbox. Enable it:

```powershell
Set-CASMailbox `
    -Identity "<MAILBOX_EMAIL>" `
    -SmtpClientAuthenticationDisabled $false
```

---

## Step 11 — Optional: FullAccess Permission

```powershell
Add-MailboxPermission `
    -Identity "<MAILBOX_EMAIL>" `
    -User "<ENTERPRISE_APPLICATION_OBJECT_ID>" `
    -AccessRights FullAccess
```

**Why?** In many environments email sending works with SendAs alone. However Microsoft documentation and several examples grant FullAccess as well. Useful when Exchange needs mailbox access beyond simple submission.

---

## Step 12 — Configure Business Central

Navigate to:

```text
Email Accounts → New → SMTP Connector
```

Settings:

```text
SMTP Server:          smtp.office365.com
Port:                 587
Secure Connection:    STARTTLS
Authentication:       OAuth2
Tenant ID:            <TENANT_ID>
Client ID:            <CLIENT_ID>
Client Secret:        <CLIENT_SECRET>
User Name:            <MAILBOX_EMAIL>
```

---

## Troubleshooting

**`AADSTS500113: No reply address is registered for the application`**

Add Redirect URI: `https://businesscentral.dynamics.com/OAuthLanding.htm`

---

**`AADServicePrincipalNotFound`**

You used the App Registration Object ID instead of the Enterprise Application Object ID.

---

**`Cannot open mailbox` / `AuthenticationContext has no rights on this session`**

Run `Add-RecipientPermission` from Step 8.

---

**`535 Authentication unsuccessful`**

Check all of the following:

- Client Secret is valid and not expired
- Admin Consent was granted
- `SMTP.SendAsApp` permission is present
- Exchange Service Principal was created (Step 7)
- SMTP AUTH is enabled on the mailbox (Step 10)

---

## Final Result

When everything is configured correctly:

- Business Central authenticates successfully
- OAuth login succeeds
- Exchange recognizes the Service Principal
- Mailbox grants SendAs permissions
- Email is sent through Microsoft 365 SMTP
- No more mysterious errors containing 14 pages of hexadecimal diagnostics from 17 different Microsoft teams

At which point you may finally close the browser tabs and pretend this was a straightforward configuration task.
