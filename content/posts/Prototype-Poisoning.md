---
title: "Prototype poisoning, it's everywhere"
date: "2022-08-06"
draft: true
authors:
  - Christoffer Jerkeby
  - Anton Linné
---
# Prototype poisoning, it's everywhere
Prototype poisoning happens when an object inherits a prototype from a parent object at the assignment.
Prototype mutation is a JavaScript feature that can be exploited by an attacker using a "`__proto__`" key in structured input. The value of the "`__proto__`" key overwrites the prototype of the destination object and its members.

## Example
The [now patched](https://github.com/ljharb/qs/pull/428) qs input library used by [body-parser](https://github.com/expressjs/body-parser/releases/tag/1.19.2) for parsing HTTP messages in Express.

Server code:
```
let input = JSON.parse(req.body);
```

Request:
```
POST / HTTP/2
Host: example.com
Content-Type: application/JSON
Content-Length: 45

{
    "value": 1,
    "__proto__": {
        "toString": null
    }
}
```

The attacker overwrites the prototype members of the `req.body` object. The pollution occurs when the request body is parsed and assigned to an object without sanitizing the "`__proto__`" key.


The `toString` function is no longer a function. Instead, `toString` is a variable with the value `null`.

## Impact
To interact with an object, the consumer use member functions and variables. The object prototype is a list of member functions and variables added to the object if the same member name isn't already present.

Running `req.body.toString()` results in a `TypeError`. The server will respond to the request with a `500 Internal Error message`.

Using the prototype poisoning technique, the attacker has replaced that function with `null`, causing a `TypeError`.
The application never logs the value of the `req.body`.

The typical attack avenues for this bug class are:

* Adding implicitly called functions with executable code. Function overwriting may be possible if the input format supports it. `JSON` does not support function bodies as a data type.
* Remote Code Execution, through code evaluation, when any prototype value is consumed by an `eval()` statement.
* Property injection by appending member variables such as "`debug: true`" can cause drastic alterations in application flow.
* Appending member values used as arguments to third-party libraries causes changes in application behaviour.
* Overwrite prototype members to bypass input validation schemas.

## Identifiction
This section covers guidance on how to get started with building prototype poisoning detections.
The static code analysis (SAST) approach uses a code scanning tool such as SemGrep or CodeQL.
A more dynamic approach (DAST), by tests the target application inputs with a `Prototype Poisoning Polyglot`.

### DAST (Prototype Poisoning Polyglot)
A verification string value must be appended to a structurally valid JSON input containing the expected fields of the target application. The input will get more coverage in the target application with a valid base request.

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
   '__proto__': {
      'hasOwnProperty':null,
      'isPropertyOf':null,
      'propertyIsEnumerable':null,
      'toLocaleString':null,
      'toSource':null,
      'toString':null,
      'valueOf':null
   }
}
```

The modified discovery string replaces the most common object prototypes with null. Access to source code can contribute to a more comprehensive list of possible implicit prototype members. The expected outcome of a successful poisoning attack is `500 Internal Server Error` or a connection Timeout. The application log should have 'TypeError' message.

### SAST (CodeQL )
A CodeQL query allows researchers to search for a defined behaviour through large codebases. The query to find prototype poisoning vulnerabilities must define a relevant source and sink. The source is a user-controlled input with hierarchical structures, such as an HTTP field containing named arrays, dictionaries or `JSON`. The sink is a reference to an inherited prototype member.
The implicit prototype functions inherited from `Object` are:

* hasOwnProperty()
* isPrototypeOf()
* propertyIsEnumerable()
* toLocaleString()
* toSource()
* toString()
* valueOf()

There may be other implicit members in the target environment. A well-crafted CodeQL query could first detect all
implicit prototype members then search for their use.

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
can affect the parent Object prototype by chaining multiple "`__proto__.__proto__`" to affect the `dunder object prototype`.

Prototype pollution typically occurs when the attacker can control the name of the array field and its value.

```
variable[NAME] = VALUE
```

In this example the attacker can control both `NAME` and `VALUE` the attacker can choose to set `NAME` to "`__proto__`" or even "`__proto__.__proto__`" and `VALUE` to "`{'toString': null}`". Thus overwriting the prototype or even the parent object prototype with any structure or array of members.

Prototype pollution occurs in deep merge functions where a local Object merges with a user input Object. Read more [here](https://github.com/HoLyVieR/prototype-pollution-nsec18/blob/master/paper/JavaScript_prototype_pollution_attack_in_NodeJS.pdf)

### The early days (2018)
[Bryan English](https://github.com/bengl) writes in 2018 about the [prototype poisoning labeled reports](https://medium.com/intrinsic-blog/javascript-prototype-poisoning-vulnerabilities-in-the-wild-7bc15347c96) recevied at [HackerOne](https://hackerone.com/nodejs-ecosystem) to the [Node.JS Securety Working Group](https://github.com/nodejs/security-wg).
Bryan describes an attack scenario where the attacker uses a prototype pollution method to append a prototype member called `admin:true` to all objects during a deep copy. The appended prototype is added to all new Objects created after the pollution, thus altering implicit prototype values in foreign objects.

### Issues in Joi, Hapi and Bourn (2019)
Joi is a schema-based input validation framework closely associated with the web framework Hapi.
[Eran Hammer](https://github.com/hueniverse) writes a [lengthy article](https://www.fastify.io/docs/latest/Guides/Prototype-Poisoning/) about the devastating effect of prototype poisoning (at assignment initialization) in Joi and Hapi. Eran is, at the time, working with the Hapi community to research and act on a report from the engineering team at [Lob](https://www.lob.com/category/engineering). In the report, Lob engineering showed that they could bypass the schema validation in Joi using a client-supplied "`__proto__`" to sneak in values.

This vulnerability is still at large in Hapi and Joi. The recommended solution is to wrap all incoming input in a "safer" JSON parser called [Bourne](https://www.npmjs.com/package/@hapi/bourne).

In a [hapi GitHub issue](https://github.com/hapijs/hapi/issues/3916) Eran explains that:
> If you use `onCredentials` or `onPostAuth` in your code, or if you use the `base64json` cookie encoding format, review your handling of `request.payload` and `request.state` objects to ensure your current (pre-patched) code is not at risk.

[Bourne](https://github.com/hapijs/bourne), is a wrapper to filter out "`__proto__`" strings from user input to Joi. [Hapi documentation](https://hapi.dev/#security) doesn't mention that Bourne filtering is mandatory on all input. This is likely a reason why Bourne adaptation is not the norm.

### The body-parser (2019)
[Body-parser](https://github.com/ljharb/qs/pull/428) is a popular library used for HTTP field parsing in Express.
The `body-parser` library use `qs` (query string) that [was patched against prototype poisoning]((https://github.com/ljharb/qs/pull/428)) in February 2022.

The Express [body-parser](https://github.com/expressjs/body-parser/releases/tag/1.19.2) library has an Open issue from 2019 discussing the issue of [input containing "`__proto__`"](https://github.com/expressjs/body-parser/issues/347).

# Contact

For media, consulting and guidance contact:

JavaScript security consulting: Anton Linné, anton at intelsquirrel dot com.
Application security, change management and detection: Christoffer Jerkeby, christoffer at jerkeby dot se.
