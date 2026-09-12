---
layout: post
title: "OAuth Consent Phishing: When 'Allow' Beats a Password"
date: 2026-09-12
tags: [oauth, saas-security, phishing, detection-engineering]
---

Most security awareness training still teaches people to spot a fake login page: wrong domain, missing padlock, a form asking for a password somewhere it shouldn't. OAuth consent phishing — also called an illicit consent grant attack — doesn't need any of that. The victim logs into the real identity provider, on the real domain, with their real password. Then they click "Allow" on a permissions screen, and an attacker-controlled application walks away with a token to their mailbox, files, or calendar. No credentials stolen, no MFA to bypass, and nothing that looks like phishing to someone trained to check the address bar.

This post covers how the attack works, why it slips past both users and a lot of standard controls, and what a detection engineer should actually be looking for in Microsoft 365 or Google Workspace logs.

## How the attack works

OAuth's authorization code flow is designed to let a third-party app request access to a user's account without ever seeing their password. That's the whole point of it, and it's a good design for legitimate integrations. The abuse case just repurposes the same mechanism:

1. The attacker registers an OAuth application with the identity provider (Azure AD / Entra ID, Google Cloud, Okta, etc.). It can have an innocuous name — "Document Viewer," "Meeting Scheduler" — and doesn't need to pass any real vetting in most tenants.
2. The attacker requests scopes that are useful to them: `Mail.Read`, `Mail.Send`, `Files.ReadWrite.All`, `offline_access`. That last one matters — it issues a refresh token, so the attacker keeps access even after the session ends or the password is changed.
3. The attacker sends a phishing email with a link to the identity provider's real consent URL, often disguised behind a link shortener or an open redirect. There's no credential-harvesting page involved at all.
4. The victim clicks through, authenticates normally against the real IdP, and lands on a legitimate-looking consent screen listing the requested permissions.
5. The victim clicks **Accept**. The IdP issues an access token (and refresh token, if requested) directly to the attacker's application. The attacker now has standing access that survives a password reset.

## Why security awareness training misses it

Traditional phishing training optimizes for one decision point: "is this login page real?" Consent phishing sidesteps that decision point entirely — the login page *is* real. The only judgment call the user has to make is whether an app's requested permissions are reasonable, which is a much harder question for a non-technical user than "does this URL look right."

A few things make the consent screen itself a weak control:

- **Scope descriptions are vague.** "Read your mail" doesn't convey "forward all your mail to an external address indefinitely."
- **Publisher verification is often missing or ignored.** Unverified publisher warnings exist in most platforms but are easy to click past, especially under time pressure — which is exactly the state a good phishing lure creates.
- **Users are trained to trust the identity provider's UI**, and correctly so. The problem isn't the IdP; it's that consent, once granted, isn't revisited.

This is worth internalizing for anyone building awareness content: "check the URL" is necessary but not sufficient guidance anymore. Training needs a line item for "think before granting an app permissions," which is a much less intuitive habit to build.

## What to actually detect

Consent phishing produces a specific, logged event that credential phishing doesn't: an application being granted delegated permissions. That's the detection anchor.

**In Microsoft 365 / Entra ID:**

- The Unified Audit Log records `Consent to application` events (`Add app role assignment to service principal` / `Add delegated permission grant` in Entra ID sign-in and audit logs). Every OAuth grant is logged — the gap is usually that nobody's alerting on it.
- Priority signals to alert on:
  - Consent granted to an app with **no verified publisher**.
  - Consent including **high-privilege or sensitive scopes** — `Mail.Read`, `Mail.ReadWrite`, `Mail.Send`, `Files.ReadWrite.All`, `Directory.ReadWrite.All`, and especially `offline_access` combined with any mail or file scope.
  - **User consent** (not admin consent) granted to an app that **didn't exist in the tenant before** — first-time app registrations acting on a single user are far more suspicious than an app already in use org-wide.
  - A burst of consent grants to the **same application across multiple users** in a short window — indicates a phishing campaign rather than one user's misjudgment.
- A starting KQL query against `AuditLogs` in Sentinel / Log Analytics:

```kql
AuditLogs
| where OperationName == "Consent to application"
| extend Scopes = tostring(TargetResources[0].modifiedProperties[?(@.displayName=='ConsentAction.Permissions')].newValue)
| where Scopes has_any ("Mail.Read", "Mail.Send", "Files.ReadWrite.All", "offline_access")
| project TimeGenerated, InitiatedBy = tostring(InitiatedBy.user.userPrincipalName), Scopes, TargetResources
```

**In Google Workspace:**

- The Admin Console's **OAuth Token Audit** activity log (and the `token` events in the Admin SDK Reports API) records third-party app authorizations.
- Alert on:
  - Apps requesting **sensitive or restricted scopes** (`gmail.readonly`, `gmail.send`, `drive`) that aren't on an org-approved list.
  - **New client IDs** authorizing for the first time in the domain.
  - Multiple users authorizing the **same unfamiliar app** within a short time window.

**Cross-platform response controls, not just detection:**

- Restrict user consent so only admin-approved publishers or pre-vetted apps can be granted access (Entra ID: "user consent for apps" set to admin-only or verified-publisher-only; Workspace: app access control allowlisting).
- Treat a confirmed malicious grant like a credential compromise, not a lesser event: revoke the app's access, invalidate refresh tokens and sessions, and check what the token was actually used for during its window of access — mailbox rules, forwarding rules, and file access are the usual follow-on abuse.

## The takeaway

OAuth consent phishing is a good example of an attack that doesn't fail because a control is missing — the audit logs exist, the consent screen exists, publisher verification exists. It succeeds because none of those controls are tuned or alerted on by default, and because awareness training hasn't caught up to a phishing technique that never asks for a password. Closing that gap is less about a new tool and more about writing the handful of detection rules above and pointing them at logs you probably already collect.
