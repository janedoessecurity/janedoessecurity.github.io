---
layout: post
title: "Authentication, Enumeration & Predictable Tokens (TryHackMe Lab)"
date: 2026-03-28
tags: [tryhackme, authentication, enumeration, burp-suite, web-security]
---

## Overview

This lab explored common weaknesses in authentication mechanisms, focusing on how small implementation details — particularly error messages and token generation — can introduce real security risks.

Rather than relying on complex exploitation, the exercises showed how attackers can gather useful information simply by observing how an application behaves.

## User Enumeration via Error Messages

One of the most straightforward issues demonstrated was user enumeration through login responses.

When attempting to log in:

- **Invalid email** → "Email does not exist"
- **Valid email, incorrect password** → "Invalid password"

![Login error message revealing valid username](/images/auth-enum-login-error.png)

This difference allows an attacker to confirm whether an email address is registered.

From a security perspective, this is a subtle but impactful flaw:

- It reduces the effort required for brute-force attacks
- It enables targeted attacks against known valid accounts

## Automating Enumeration

To scale this process, enumeration can be automated.

By sending multiple login attempts and analysing the responses, it becomes possible to quickly identify valid users.

![Script output showing enumerated users](/images/auth-enum-script-output.png)

This highlights an important point:

> Small information leaks become far more serious when automated.

Enumeration is rarely the end goal — it is typically used to reduce the search space before attempting further attacks, such as password brute forcing.

## Predictable Tokens and Burp Intruder

The lab then moved on to password reset functionality, specifically focusing on predictable tokens.

A request was captured in Burp Suite and the token parameter was identified as a target for brute forcing:

```
GET /labs/predictable_tokens/reset_password.php?token=§123§ HTTP/1.1
Host: enum.thm
```

![Burp Suite intercept of authentication request](/images/auth-enum-burp-request.png)

A list of potential tokens was generated using crunch:

```bash
crunch 3 3 -o otp.txt -t %%% -s 100 -e 200
```

This created a list of numeric tokens within a defined range, which were then used as payloads in Burp Intruder.

## Identifying the Valid Token

When the attack was executed, most responses appeared identical. However, one request produced a different response, indicating a valid token.

This demonstrates a common weakness in poorly implemented reset mechanisms:

- Tokens are predictable
- There is no rate limiting
- There is no additional validation step

Even a small, constrained token space becomes exploitable under these conditions.

## Password Reset Weaknesses

The lab highlighted several risks associated with password reset functionality:

- **Predictable tokens** → can be guessed or brute forced
- **Token expiration issues** → increase the attack window
- **Weak identity validation** → makes account takeover easier
- **Information disclosure** → supports enumeration
- **Insecure transport (HTTP)** → risks interception of reset links

## Key Takeaways

- Authentication vulnerabilities are often logic-based rather than technical complexity
- Error messages should be consistent and non-descriptive
- Enumeration is typically a precursor to more targeted attacks
- Token generation must be random and unpredictable
- Small weaknesses become significant when combined and automated

## Reflection

What stood out in this lab is how realistic these issues are.

None of the techniques required advanced exploitation — just:

- Observing application behaviour
- Identifying patterns
- Applying simple automation

From a defensive perspective, this reinforces the importance of:

- Reviewing how systems respond to invalid input
- Designing authentication flows carefully
- Balancing usability with security

## Final Thoughts

This lab is a good reminder that:

> Security weaknesses often come from design decisions, not just technical mistakes.

Even small differences in behaviour can provide attackers with enough information to move forward.
