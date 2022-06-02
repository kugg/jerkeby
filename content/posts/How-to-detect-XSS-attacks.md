---
title: "How to detect XSS attacks"
date: "2022-06-02"
draft: true
---
Key indicators for successful XSS attacks are Cookie use from multiple origins, CSP policy violations, .
This article explains the typical success and attempt indicators for Cross Site Scripting attacks.

Indicators of violation make out a risk score.
Once a risk score threshold has been breached multiple indicators are presented to the investigator in an alert.

As a defender of large web services finding live attacks can be difficult. Knowing what to look for means that its possible to build detection for typical Cross Site Scripting attacks.

## Success indicators:
A successindicator is an indicator that a security violation has already occured.

### CSP policy violation
#### Descrption
#### Indication
#### Detection point
#### Confidence level

### Cookie use from multiple IP addresses and/or agents
#### Descrption
#### Indication
#### Detection point
#### Confidence level

### Unexpected `Content-Type`
#### Descrption
#### Indication
#### Detection point
#### Confidence level

### Unexpected `Accept` headers
#### Descrption
The accept header indicate to the server what type of response format the client expects. Following links leads to an open ended Accept header that is use dto request any file format.
Loading Images, Videos, CSS, JavaScript or XML using specified tags generate a specific Acceppt header for expected format.
#### Indication
An application component receives an accept header that indicate that the client expects an image when it attempts to send a Search query.
This is an indication that the client was tricked to load the resource as part of an `<IMG>`.
This method is typically used to automate an XSS attack to avoid user interraction with the victim.
#### Detection point
The detection point is an application component that is filetype aware. It must know what the resource format it provides and compare it to the Accept header.
#### Confidence level
The confidence level for this indicator is Moderate to High.
There may be false possitives in ambigous endpoints such as `/download?filename=`.

### Unexpected DOM changes
#### Descrption
#### Indication
#### Detection point
Sentry.io shows a DOM for the given user that contains methods that was not originally in the server loaded DOM
#### Confidence level

### The user journey is erratic
#### Descrption
The user journey is the order or requests ro reach a specific state. Frameworks like castle.io provide monitoring of user journeys.
An interrupted or erratic journey is one where the client unexpectedly make a request to an endpoint that hsouldnt be available to user from its current view.
#### Indication
The erratic user behaviour may be an indicator that the victim browser is now acting on behalf of the attacker.
#### Detection point
Catle.io and Sentry.io both provide information about the client context and the user journey.
#### Confidence level
Low

## Attempt indicators
An attempt indicator has a lower risk score than a success indicator but are instrumental in finding the offending XSS request.

### Request contains JavaScript
#### Descrption
An input validation detects that the input does not follow expected syntactic format.
#### Indication
#### Detection point
#### Confidence level


### Request is triggered from external origin
#### Descrption
The Orgin header is an HTTP header that cannot be altered by the client. The header indicates from which protocol, domain and port the request originated.
An application may keep track of its own origin and verify the origin header of a request to ensure that it originated from a controlled location.
#### Indication
An unexpected `origin` header indicate that the request was sent from another domain. The external domain may be allowed but not reasonable for the destination.
#### Detection point
If a cross origin policy is not in place, the attempt is detected by a check of the `Origin` header on a POST request.
#### Confidence level
A HTTP POST request was made from an unexpected external origin indicate an attempted XSS with moderate confidence.

### The request contains encoded input
#### Description
The user controlled input in the request contains a characterset that is typically used by one specific encoding format.
#### Indication
The attacker use encoding or doubleencoding to mask the XSS attack intent from its victim.
#### Detection point
Detected by a syntactic input validator or exception thrown by application while parsing a format.
#### Confidence level
User input may contain URL and Base64 encoded input for multiple reasons.
If the encoding of the request violate the expected input data type or syntax there is a Low confidence level of attempted XSS.

