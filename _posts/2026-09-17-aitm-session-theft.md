---
layout: post
title: "Adversary-in-the-Middle Phishing: When the Session Cookie Beats MFA"
date: 2026-09-17
tags: [okta, aitm, session-hijacking, google-workspace, saas-security, phishing, detection-engineering]
---

For years the standard advice for phishing-resistant... well, phishing, was simple: turn on MFA.

Adversary-in-the-middle phishing, AiTM for short, is the reason that advice stopped being sufficient on its own.

An AiTM attack doesn't try to guess a password or bypass a second factor. It sits between the victim and the real identity provider, relays the entire login in real time, including the MFA challenge, and then steals the session cookie that gets issued at the end of it.

The victim authenticates correctly. MFA is satisfied correctly. And the attacker still ends up with a working, authenticated session.

That's a different problem to the one most security awareness training was built to solve.

![Diagram of the AiTM phishing attack flow: an attacker-controlled proxy relays credentials and MFA between the victim and Okta, captures the resulting session cookie, then replays it later to pivot into Google Workspace via SSO without triggering another MFA prompt](/images/aitm-session-theft-flow.svg)

## How does the attack work

AiTM phishing kits work as a reverse proxy. The victim's browser talks to the attacker's infrastructure, which in turn talks to the real identity provider, in this case Okta, and passes traffic back and forth in both directions.

### 1. The victim reaches the proxy

The phishing lure leads to infrastructure controlled by the attacker rather than to Okta directly. This is usually a lookalike domain, sometimes registered specifically for the campaign, sometimes a compromised or newly created site.

This is one meaningful difference from [OAuth consent phishing](/oauth-consent-phishing/). There, the victim genuinely reaches Google's real domain. Here, checking the URL still matters, because the victim is not actually on `okta.com` or your org's Okta subdomain, they're on the attacker's proxy.

### 2. Credentials are relayed live

The victim enters their username and password on the proxy page. The proxy forwards those credentials to the real Okta sign-in flow immediately and returns whatever Okta sends back.

Nothing is stored and replayed later. The attack has to happen in real time, with the victim unknowingly acting as the middleman for their own compromise.

### 3. The MFA challenge is relayed too

If Okta prompts for a second factor, the proxy passes that prompt straight through to the victim, and passes the victim's response straight back to Okta.

Push notification, OTP, whatever the policy requires, the victim completes it against the real Okta service. This is the step that defeats the "just enable MFA" mental model. MFA isn't bypassed or brute forced. It's satisfied honestly, just via a relay the victim doesn't know is there.

### 4. Okta issues a session

Once authentication succeeds, Okta issues a session, backed by a session cookie in the victim's browser.

Because the proxy sat in the middle of the entire exchange, it also sees this cookie as it's set.

### 5. The attacker captures and replays the cookie

The attacker takes the captured session cookie and loads it into their own browser.

As far as Okta is concerned, this is just a valid, already-authenticated session. No password is needed. No MFA prompt fires. The authentication event already happened, successfully, and the cookie is the proof of it.

From here the attacker can access anything the session is scoped for, including pivoting through SSO into downstream apps like Google Workspace, without ever needing the victim's credentials again.

## Why this is a harder problem than credential phishing

Traditional phishing training focuses on two decisions: is this login page real, and should I enter my password here.

AiTM technically leaves both of those checks intact and still succeeds.

**The login experience feels completely normal.** The user sees Okta's real prompts, relayed live. There's no obviously broken flow to notice, unless someone checks the URL carefully or the org has controls that only a genuine origin can satisfy.

**MFA success creates false confidence.** Users, and often the organisations training them, treat "I completed MFA" as the finish line. AiTM is precisely the class of attack where that assumption fails.

**The compromise happens after the part users are trained to scrutinise.** By the time the session cookie is issued, the victim has already done everything "right" from a traditional phishing-awareness standpoint.

This is worth being explicit about in awareness content: MFA, on its own, tells you the user proved who they are at one point in time. It doesn't tell you whether the resulting session is still in the hands of that user.

## What to actually detect

The good news is that session theft still leaves a trail, because the token has to be used from somewhere the original authentication didn't happen.

**Session start and subsequent activity from mismatched context**

The most direct signal is a gap between the device, IP address, or user agent seen at authentication time and the device, IP, or user agent using the session afterwards.

A session authenticated from a managed laptop on a known corporate network, then immediately active from an unfamiliar ASN or a different device fingerprint, is a strong candidate for a stolen token rather than a legitimate user simply changing location.

**Okta's risk and behaviour signals**

Okta's System Log surfaces risk scoring and behaviour detection, including new device, new IP, new geolocation, and impossible travel evaluations tied to authentication and session events.

None of these individually proves session theft. A legitimate user does occasionally sign in from a new device or a new location. But:

> authentication succeeded cleanly + session immediately used from an anomalous location or device

is a much more interesting pattern than either signal alone, especially when the anomaly appears within minutes of the original sign-in rather than hours or days later.

**Downstream SSO activity that doesn't match the story**

Because Okta sessions are frequently used to pivot into other SaaS platforms via SSO, it's worth correlating suspicious Okta session activity with what happens next in Google Workspace.

An anomalous Okta session followed by unusual Gmail or Drive activity, unexpected app access, or admin console changes is a strong indicator the session itself, not just the login attempt, has been compromised.

**Multiple users hitting the same phishing infrastructure**

As with OAuth consent phishing, one anomalous session might be noise. Several users authenticating through a similar pattern, or session anomalies clustering around the same time window, points toward a campaign rather than an isolated incident.

The point of collecting this telemetry isn't to admire it. It's to turn "a user authenticated successfully" from the end of the investigation into the start of one.

## Prevention matters as much as detection

Detection catches AiTM after the fact. The more durable fix is authentication that AiTM simply can't relay.

Phishing-resistant authenticators, FIDO2 and WebAuthn based methods like Okta FastPass or hardware security keys, are cryptographically bound to the real origin they were registered against. A proxy sitting on a lookalike domain can't relay that challenge successfully, because the authenticator itself checks the origin as part of the protocol, not the user.

That's a meaningfully different guarantee to OTP or push-based MFA, both of which can be relayed by a proxy without the user noticing anything wrong.

Beyond authenticator choice, session lifetime and binding controls matter too. Shorter session lifetimes reduce the window a stolen cookie is useful for. Network zone restrictions and device assurance policies can require that a session continue to be used from a context consistent with where it was issued, making a replayed cookie from different infrastructure harder to use even after capture.

The goal is the same shift as before: move from "hopefully the user notices the lookalike domain" to "the authentication method itself doesn't work against a relay."

## Responding to suspected session theft

Once session hijacking is suspected, treat it as an active account compromise, not just a phishing report.

At minimum, investigate:

- which session or sessions are associated with the anomalous activity;
- what device, IP, and user agent authenticated the session versus what used it afterwards;
- what was accessed during the session, including any downstream SSO activity into Google Workspace or other connected apps;
- whether MFA was genuinely satisfied by the user, or relayed without their awareness; and
- whether other users show the same pattern, indicating a wider campaign.

Revoking the specific session isn't enough on its own if the underlying credentials were also captured by the same infrastructure. Both the session and the credentials should be treated as compromised, and the user should re-authenticate with a phishing-resistant method where possible.

## The takeaway

AiTM phishing is a reminder that authentication and session integrity are two different security properties.

A user can authenticate correctly, satisfy MFA correctly, and still hand an attacker a fully working session, because the thing that actually grants access afterwards is the cookie, not the memory of having logged in.

For security awareness teams, that means the message needs to expand beyond "check the URL and use MFA" to include the idea that a session, once issued, is itself a credential worth protecting.

For detection engineers, it means watching the gap between where a session was authenticated and where it's subsequently used, not just whether authentication happened at all.

And for identity teams, it means recognising that not all MFA is equal against this technique. Phishing-resistant authenticators close the gap that OTP and push notifications leave open.

The session log already tells this story. The question is whether anyone's watching the handoff, not just the login.

## Further reading

- [Okta — Phishing-resistant authentication](https://help.okta.com/en-us/content/topics/security/phishing-resistant-auth.htm)
- [Okta — ThreatInsight](https://help.okta.com/en-us/content/topics/security/threat-insight/about-threat-insight.htm)
- [Okta System Log — Event types reference](https://developer.okta.com/docs/reference/api/system-log/)
- [CISA — Phishing-Resistant MFA](https://www.cisa.gov/sites/default/files/publications/fact-sheet-implementing-phishing-resistant-mfa-508c.pdf)
