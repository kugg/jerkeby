---
title: "Insecure design and failed business logic"
date: "2023-03-14"
draft: false
authors:
  - Christoffer Jerkeby
images: 
  - /images/business_logic.gif
summary: Spot the miss-alignment between technology and business flow. Learn how to handle failures in business logic.
description: Spot the miss-alignment between technology and business flow. Learn how to handle failures in business logic.
---
![Insecure design](/images/business_logic.gif)

> "When things were supposed to work one way, but they also worked in other ways." - Christoffer Jerkeby

[OWASP 2021 A04 Insecure Design](https://owasp.org/Top10/A04_2021-Insecure_Design/) is a miss-alignment between technology and business flow. We cannot predict all problems, but we can handle failure gracefully.

Insecure design happens when we develop components in accordance with the specification. Insecure design happens when we use secure frameworks. Insecure design happens when we use state-of-the-art testing tools. We come prepared, and yet the system fails us.

# Bummer!
Why is it that we are still getting these vulnerabilities despite [shifting left](https://learn.microsoft.com/en-us/previous-versions/visualstudio/visual-studio-2012/ee330950(v=vs.110)?redirectedfrom=MSDN)?

We must:
1. Accept that we cannot predict all faults
2. Learn from real world examples
3. Anticipate failure in business logic

In threat modelling, we use the acronym ["STRIDE"](https://www.youtube.com/watch?v=iGkX06sVFFM) (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service and Elevation of privilege) to remind us what we are trying to achieve and what can go wrong. Even with STRIDE, anticipating logical flaws can sometimes be challenging. [All models are incorrect, but some are useful](https://jamesclear.com/all-models-are-wrong#:~:text=In%201976%2C%20a%20British%20statistician,is%20correct%20in%20all%20cases.). Insecure design of software stems from incorrect assumptions and less accurate models. The misalignment can start when we draw our `Data Flow Diagram` (DFD) for the threat model. I call this anti-pattern the `inaccurate model`. Misalignment can also occur when [circumstances change](https://how.complexsystems.fail/#14) after the software is released to production. When outside conditions outside of the model change, I call this `changed circumstances`.

!["Do not assume anything, clear your mind must be" - Master Yoda](/images/Business_logic_yoda.jpeg)

Just as in [Domain Driven Design](https://martinfowler.com/bliki/BoundedContext.html), we want to ensure developers and testers understand the domain the application serves. Most importantly, we want to avoid implicit assumptions about user behaviour or the behaviour of other parts of the application. The question is, how do we achieve this?

Let's take a look at our real world examples!

## Business logic errors [(CWE-840)](https://cwe.mitre.org/data/definitions/840.html)
### Trust Boundary Violation [(CWE-501)](https://cwe.mitre.org/data/definitions/501.html)
Trust boundary violation is when an untrusted application handles privileged information. Typically violations occur when handling user-supplied parameters before authentication. In my experience these violations commonly occur in low-risk software that adds security-dependent features late in development. To a product owner or lead developer, it's not apparent that the legacy code requirements have changed to include security. I call this behavioural pattern `security afterthought`. The `security afterthought` pattern is often a consequence of insecure design.
 
In this example, the user-supplied parameter `username` value becomes a session attribute without any authentication check ( or input validation).

```
username = request.getParameter("username");
if (session.getAttribute(ATTR_USR) == null) {
   session.setAttribute(ATTR_USR, username);
}
```

Preferably a more structured method for handling user input securely should have been provided from the design phase. Such as the entity modelling pattern from [Secure by design by Dan Bergh Johnsson, Daniel Deogun and Daniel Sawano](https://www.manning.com/books/secure-by-design).

### Authentication Bypass Using an Alternate Path or Channel [(CWE-288)](https://cwe.mitre.org/data/definitions/288.html)
In my experience authentication bypasses have traditionally occurred when inconsistent authentication checks are applied, commonly in frameworks where the developer must explicitly declare that authentication is required. The bug-class has since grown to include applications with more states than "logged in" and "not logged in".

When a user receives an authentication token for performing one single action, the rest of the application is unaware of any "new" token state limitations.

Here are a few examples:

#### Registration flow token
At the end of a user registration flow, it's common to give the user a logged-in session as their new user without validating the email address or requiring the user to log in. A consequence of providing a "post-registration login token" may be that from the point of receiving the token, they can request any API endpoint inside the application without having a fully verified account.

#### Password reset token
Users who have requested a password request token and clicked the link receive a session token dedicated to changing the user password. In this case, the user does not change the password. They use that session to perform other API actions. This authentication bypass method skips a mandated MFA authentication.

### Race conditions [(CWE-362)](https://cwe.mitre.org/data/definitions/362.html)
A race condition is when events occur in an unexpected order. A consequence is that contextual information is missing or overwritten, resulting in faults that an attacker can exploit. A popular example is a bank transfer conducted twice to overdraft the account before checking the balance.

An unexpected version is when we realise that one client can simultaneously perform thousands of concurrent requests. A feature that attempts to count requests or rate limit connections can be overwhelmed if the attacker aligns their threads sufficiently.

These three scenarios are part of a pattern that I call `missing perspective`.

Once again, the assumptions made during design phase are broken.

# Anticipating potential failure
These vulnerability classes break assumptions in three dimensions: "time", "space-boundary" and "legacy logic".
As "shift-left" aim to prevent fault, we must accept that misconceptions like these will happen as long as we develop code incrementally. If we had a detailed understanding of the software we are designing before starting, we could design robust software with the three dimensions in mind. However, even if we desire clairvoyance capabilities, it's not who we are. *Yet!*

Johan Bergström writes in [Becomming Resilient](https://www.routledge.com/Resilience-Engineering-in-Practice-Volume-2-Becoming-Resilient/Nemeth-Hollnagel/p/book/9781472425157) Chapter 9 that we must:

1. Respond to the actual
2. Monitor the critical
3. Anticipate the potential
4. Learn from the factual

![All this is true, because it rhymes](/images/Business_logic_lego_truth.png)

How should we respond to a bug of this kind?
* We will `raise an incident`.
* We will `relay observations from systems`, `analyse the fault` and `form a hypothesis`. 
* We will prepare a `fix of the vulnerability`.

See: [Allspaw, John. (2015). TRADE-OFFS UNDER PRESSURE, Lund University](https://lup.lub.lu.se/luur/download?func=downloadFile&recordOId=8084520&fileOId=8084521).

The cost of these remediation efforts depends on our preparation and collective [experience dealing with failure](https://how.complexsystems.fail/#18).

One way to prepare for the unexpected is to `monitor the customer journey`. To us, the customer journey is a critical business logic function. Monitoring it would only be natural to business stakeholders. Cataloguing exceptions raised by our application, creating actionable error descriptions, and a coherent log output gives us insight into our customer journey's `negative experiences`.

Once identified, we can anticipate exceptions raised by software executing unanticipated flows. In the banking case, the race condition could result in multiple `InsufficientFundsException`'s. In the case of out-of-order requests, we could monitor for `reference before assignment` or `NullPointerException` vulnerabilities as they would indicate that the request came out of order. Neat, now we have a method to evaluate our business logic!

Lastly, we will learn from past experiences. These events are factual shared stories. Let us cherish them with [blameless retrospects](https://www.youtube.com/watch?v=4nRahQddtJ0). In this process, we identify risks and return to them in our next threat model workshop and decide on controls for our design. Great!

# Actively challenging assumptions
Using penetration testing, bug bounty and [security chaos engineering experiments](https://www.oreilly.com/library/view/security-chaos-engineering/9781492080350/), we can trigger failure states in our business logic that will present an alternative view to the development organisation. The alternative view challenges the `missing perspective`, `security afterthought`, `changed circumstances` and `inacurate model`. To improve value of the alternative view it's important to share the assumptions with the tester prior to the test. Contextual knowledge combined with the experience of failure improves our capacity to deal with the unexpected and help us align with business logic.

# Summary
Reading this, you have learned that:
1. We will attempt to design our software with security in mind based on assumptions.
2. Logical vulnerabilities can occur because we always carry (false) assumptions when we develop code, this is called insecure design.
3. Software resilience is about accepting the premise that we will make false assumptions and that software needs maintenance.

# Learn More
If you speak Swedish, you might be interested in this video where I talk about how to establish an AppSec program and make alternative views more valuable.

{{< youtube QTDn-KVDAcg >}}
