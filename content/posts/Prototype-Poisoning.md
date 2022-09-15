---
title: "What is prototype poisoning? Prototype bugs explained!"
date: "2022-09-14"
draft: false
authors:
  - Christoffer Jerkeby
  - Anton Linné
images: 
  - /images/Prototype-poisoning-Anton-Linne-Christoffer-Jerkeby.jpg
description: What is prototype poisoning? Prototype bugs explained! Prototype poisoning is when an object inherits a prototype from user input. It leads to input filter bypass, parameter injection and denial of service.
---
![Prototype Poisoning on stage](/images/Prototype-poisoning-Anton-Linne-Christoffer-Jerkeby.jpg)

Prototype poisoning is when an object inherits a prototype from user input. It leads to input filter bypass, parameter injection and denial of service.

Prototype mutation is a JavaScript feature that can be exploited by an attacker using a "`__proto__`" key in structured input. The value of the "`__proto__`" key overwrites the prototype of the destination object and its members. Poisoning can be found in many formats and protocols, but this article will focus on JSON.

![Prototype Poisoning](/images/Prototype-Poisoning.png)

## Example
Server code:
```
let input = Object.assign({}, JSON.parse(req.body));
```

Request:
```
POST / HTTP/2
Host: example.com
Content-Type: application/JSON
Content-Length: 42

{
    "value": 1,
    "__proto__": {
        "toString": null
    }
}
```

The attacker overwrites the prototype members of the `input` object using the `"__proto__"` value from req.body. The pollution occurs when the request body is parsed and assigned to an object without sanitizing the "`__proto__`" key.

The `toString` function is no longer a function. Instead, `toString` is a variable with the value `null`.

## Impact
To interact with an object, the consumer use member functions and variables. The object prototype is a list of member functions and variables added to the object if the same member name isn't already present.

Running `req.body.toString()` results in a `TypeError`. The server will respond to the request with a `500 Internal Error message`.

Using the prototype poisoning technique, the attacker has replaced that function with `null`, causing a `TypeError`.
The application never logs the value of the `req.body`.

The typical attack avenues for this bug class are:

* Adding implicitly called functions with executable code. Function overwriting may be possible if the input format supports it. `JSON` does not support function bodies as a data type.
* Remote Code Execution or Command Injection, through code evaluation, when any prototype value is consumed by an `eval()` or `system()` statement.
* Property injection by appending member variables such as "`debug: true`" can cause drastic alterations in application flow.
* Appending member values used as arguments to third-party libraries causes changes in application behaviour (Argument injection). 
* Overwrite prototype members to bypass input validation schemas.
* Causing Denial of Service by inferring infinite loops (Loop Manipulation or recursive calls).
* Manipulating global values to cause program flow alteration (Global variable tampering) or Denial of Service.

## Video
See the full talk from [SEC-T community night](https://www.sec-t.org/talks/#Christoffer-Jerkeby-Anton-Linne) here:
{{< youtube K3ocqLFHkpw 10959 >}}

## Identification
This section covers guidance on how to get started with building prototype poisoning detections.
The static code analysis ([SAST](https://en.wikipedia.org/wiki/Static_application_security_testing)) approach uses a code scanning tool such as SemGrep or CodeQL.
A more dynamic approach ([DAST](https://en.wikipedia.org/wiki/Dynamic_application_security_testing)), by tests the target application inputs with a `Prototype Poisoning Polyglot`.

### DAST Testing (Prototype Poisoning Polyglot)
Appending a custom prototype to all JSON objects can be a useful method to detect prototype poisoning. A "`__proto__`" string value must be appended to a structurally valid JSON input containing the expected fields of the target application. The input will get more coverage in the target application with a valid base request.

Assuming the default input to an application is "`username`", "`password`" the base request would look like this.

```
{
   'username': 'a',
   'password': 'b'
}
```

The modified discovery structure would be:

```
{
   'username': 'a',
   'password': 'b',
   '__proto__': null
}
```

The modified discovery string replaces any reference to any protoype attribute with null. Access to source code can contribute to a more comprehensive list of possible implicit prototype members. The expected outcome of a successful poisoning attack is `500 Internal Server Error` or a connection Timeout. The application log should have `TypeError` message.

For input filtering bypass inspiration see read the [bourne testcases](https://github.com/hapijs/bourne/blob/master/test/index.js).
For query string format dynamic testing see [qs testcases](https://github.com/ljharb/qs/pull/428/commits/8b4cc14cda94a5c89341b77e5fe435ec6c41be2d#diff-b266c4cbb2cefa3caf90e138f935e3cf489c6655e84c1ca9d0425bb0d9af9cc6).

### SAST (CodeQL and LYNX)
A CodeQL query allows researchers to search for a defined behaviour through large codebases. The query to find prototype poisoning vulnerabilities must define a relevant source and sink. The source is a user-controlled input with hierarchical structures, such as an HTTP field containing named arrays, dictionaries or `JSON`. The sink is a reference to an inherited prototype member.
The implicit prototype functions inherited from `Object` are:

* hasOwnProperty()
* isPrototypeOf()
* propertyIsEnumerable()
* toLocaleString()
* toSource()
* toString()
* valueOf()

[Jorges](https://twitter.com/jorge_ctf) writeup on [Finding Prototype Pollution with CodeQL](https://jorgectf.github.io/blog/post/finding-prototype-pollution-gadgets-with-codeql/)
 provides inspiration on how to query JavaScript for prototype mutation.

There may be other implicit members in the target environment. If none of the standard properties are used by the target application a app-specific attribute manipulation approach may be more applicable.

The Hidden Property Abuse scanning tool [LYNX](https://github.com/xiaofen9/Lynx) is capable of identifying potentially user controlled property candidates in the target application.

# Prior art in prototype Mutation, Poisoning and Pollution
As researchers, we acknowledge that it's difficult to distinguish one security flaw from another. This chapter aims to bring clarity to the related concepts.

## Prototype Mutation
A prototype mutation is an intended effect of attempting to alter the object's prototype. Performing prototype poisoning and pollution is a form of [prototype mutation](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf).

## Prototype pollution and poisoning
The term `Prototype Poisoning` has been used to discuss two types of prototype mutations. In the early days (2018), the two bug classes were reported as prototype pollution and poisoning interchangeably.

### Poisoning
Prototype poisoning is when an attacker passes a structure containing a "`__proto__`" field to cause prototype assignment on `Object` initialization.

Prototype poisoning is distinguished from pollution by the limitation that the parent object prototypes are immutable. The attacker can only affect the input object and children that inherit its prototype. Therefore the availability, attack methodology and
impact are different. Prototype inheritance is an unavoidable JavaScript functionality.

### Pollution
The prototype pollution vulnerability is similar in input but different in impact and effect. A prototype pollution vulnerability
can affect the parent Object prototype by chaining multiple "`__proto__`" to affect the `parent object prototype`.

Prototype pollution typically occurs when the attacker can control the name of the array field and its value.

```
variable[NAME] = VALUE
```

In this example the attacker can control both `NAME` and `VALUE` the attacker can choose to set `NAME` to "`__proto__`" and `VALUE` to "`{'toString': null}`". Thus overwriting the prototype with any structure or array of members.

Prototype pollution occurs in deep merge functions where a local Object merges with a user input Object. Read more [here](https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf)

### The early days (2018)
[Bryan English](https://github.com/bengl) writes in 2018 about the [prototype poisoning labeled reports](https://medium.com/intrinsic-blog/javascript-prototype-poisoning-vulnerabilities-in-the-wild-7bc15347c96) recevied at [HackerOne](https://hackerone.com/nodejs-ecosystem) to the [Node.JS Securety Working Group](https://github.com/nodejs/security-wg).
Bryan describes an attack scenario where the attacker uses a prototype pollution method to append a prototype member called `admin:true` to all objects during a deep copy. The appended prototype is added to all new Objects created after the pollution, thus altering implicit prototype values in foreign objects.

### Issues in Joi, Hapi and Bourn (2019)
Joi is a schema-based input validation framework closely associated with the web framework Hapi.
[Eran Hammer](https://github.com/hueniverse) writes a [lengthy article](https://www.fastify.io/docs/latest/Guides/Prototype-Poisoning/) about the devastating effect of prototype poisoning (at assignment initialization) in Joi and Hapi. Eran is, at the time, working with the Hapi community to research and act on a report from the engineering team at [Lob](https://www.lob.com/category/engineering). In the report, Lob engineering showed that they could bypass the schema validation in Joi using a client-supplied "`__proto__`" to sneak in values.

This vulnerability is still at large in Hapi and Joi. The recommended solution is to wrap all incoming input in a "safer" JSON parser called [Bourne](https://www.npmjs.com/package/@hapi/bourne).
The [test cases provided with bourne](https://github.com/hapijs/bourne/blob/master/test/index.js) demonstrates various methods on how to inject prototypes in JSON, bypassing rudimentary content filtering.

In a [hapi GitHub issue](https://github.com/hapijs/hapi/issues/3916) Eran explains that:
> If you use `onCredentials` or `onPostAuth` in your code, or if you use the `base64json` cookie encoding format, review your handling of `request.payload` and `request.state` objects to ensure your current (pre-patched) code is not at risk.

[Bourne](https://github.com/hapijs/bourne), is a wrapper to filter out "`__proto__`" strings from user input to Joi. [Hapi documentation](https://hapi.dev/#security) doesn't mention that Bourne filtering is mandatory on all input. This is likely a reason why Bourne adaptation is not the norm.

### The body-parser (2019)
[Body-parser](https://github.com/ljharb/qs/pull/428) is a popular library used for HTTP field parsing in Express.
The `body-parser` library use `qs` (query string) that [was patched against prototype poisoning](https://github.com/ljharb/qs/pull/428) in February 2022.
The [testcase in the `qs` patch](https://github.com/ljharb/qs/pull/428/commits/8b4cc14cda94a5c89341b77e5fe435ec6c41be2d) demonstrates how to do prototype poisoning in a query string or body.

> `categories[__proto__]=cats&categories[__proto__]=dogs&categories[some][json]=toInject`

The Express [body-parser](https://github.com/expressjs/body-parser/releases/tag/1.19.2) library has an Open issue from 2019 discussing the issue of [input containing "`__proto__`"](https://github.com/expressjs/body-parser/issues/347).

### Hidden Property Abuse (2019-2021)
Last but not least, a paper on [hidden propery abuse by Feng Xiao et. al USENIX21](https://www.usenix.org/system/files/sec21fall-xiao.pdf) presented at usenix describes how to manipulate add non prototype properties on Objects to alter program flow. The authors created a scanning tool called [LYNX](https://github.com/xiaofen9/Lynx) for detecting Hidden Property Abuse (HPA) in JavaScript code. The bug class attack paths are identical to prototype poisoning but the attack vector use various "prototype carrying objects" instead of prototype pollution.
The researchers shows the power of symbolic execution tools by reporting [15 different CVE's in the period 2019-2020](https://www.usenix.org/system/files/sec21_slides_xiao.pdf). For a full [CVE reference see page 4 here](https://i.blackhat.com/USA-20/Wednesday/us-20-Xiao-Discovering-Hidden-Properties-To-Attack-Nodejs-Ecosystem.pdf).
Fen Xiao, devices two attack paths for HPA, the prototype inheritance hijacking and app-specific attribute manipilation. Prototype inheritance hijacking means to define attributes in user input that mimic prototype names. The attributes are referenced instead of the (unmodified) prototype. The exploit method have lead to SQL injection in [mongodb](https://jira.mongodb.org/browse/NODE-2514) and [taffy](https://security.snyk.io/vuln/SNYK-JS-TAFFY-546521).
The app-specific attribute manipulation technique is a technique where an additional attribute is added that are going to be refered to by the target application directly.
In this case the [addDocument](https://github.com/mongo-express/mongo-express/blob/master/lib/routes/document.js#L49) of [mongo-express](https://github.com/mongo-express/mongo-express/) make use of the bson parser to convert `req.document` in to a bson structure.
The `mongodb-query-parser` provides a bson format parser that executes `toBSON()` if it exists. If the mongo document contains an attribute named `toBSON` the values can be passed to an [MQL query called `safer-eval`](https://github.com/mongodb-js/query-parser/compare/v1.6.0-rc.0...master) which can result in an infinite loop and [block the event handler](https://github.com/mongo-express/mongo-express/commit/58f4cc50b5a93104505dd3bea6b6324e9d56729c). This issue is tracked as [CVE-2020-6639](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2020-6639).

While the LYNX symbolic execution tool currently support query string and JSON input, note that there are many other implicit input formats such as the HTTP header format itself, XML etc. that may be input channels for prototype pollution.

# Contact

For media, consulting and guidance contact:

JavaScript security consulting: Anton Linné, anton at intelsquirrel dot com.
Application security, change management and detection: Christoffer Jerkeby, christoffer at jerkeby dot se.
