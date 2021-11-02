---
title: "Coffee Time"
date: "2021-10-11"
draft: false
---
I'm writing this from a cafe in Malmö. I traveled down south to work with a client and meet some old friends. On my newly won free time I have been reading up on the new [issue of phrack](http://phrack.org/issues/70/1.html), specifically I have had my eyes on "*[Twenty years of escaping the java sandbox](http://phrack.org/issues/70/7.html#article)*".

Whats noticeable about the new issue of Phrack is that a majority of the articles cover various methods for breaking out of confined execution environments rather than increasing privilege or gaining remote access. From 2010 when I used to reviewed articles for Phrack, the security tactics are now leaning heavy on compartmentalisation.

An early adopter of this strategy was Java. Unfortunately Java security is notoriously difficult to risk assess. The CVE format doesn't do it justice, [vulnerabilities with vague descriptions](https://cwiki.apache.org/confluence/display/WW/Version+Notes+2.3.35) break the Internet and critical flaws go unnoticed. Hands on experience and domain specific knowledge is needed to safely navigate these waters.

I started researching one specific Java vulnerability this summer that I'd like to share with you. Sandbox escape vulnerabilities are exploited by attackers who have control over a Java application or library in the target environment. This could be an app running on a mobile device, a sim-card or a popular third party library that is embedded in to a legitimate web-application. The target is to break out of the Java Virtual Machine and run any code as the java process.

| ![](https://dim.mcusercontent.com/cs/ddd7ae486331199e51e491d97/images/2bdde76d-8f2c-e44d-bd1b-7e0448d00752.png?w=1091&dpr=2) |

You better sit down for this
============================

Java runs in a virtual machine and converts the meaning of bytecode blocks in to assembler instructions. In the Phrack article they mention two types of Java sandbox escapes one is aimed towards the Java Class Library (JCL) the other one towards the Java Virtual Machine (JVM). These two attack techniques attempts to corrupt memory through type confusion, integer overlfows and confused deputy attacks. All of these attack categories have known mitigations that are being employed in recent versions of the Java JRE.

A new type of vulnerability was discovered by Huixin Ma from Tencent who found a vulnerability in the Java Hotspot Just In Time (JIT) compiler c1. It was fixed quite silently this summer, so I decided to find out more. The vulnerability is called [CVE-2021-2388](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-2388), it uses a malformed byte-code stream to create a heap overflow in compile time. Redhat has been my best friend in finding out more about Java vulnerabilities, [the patch details contain a link to bugzilla and a Oracle issue id](https://access.redhat.com/errata/RHSA-2021:2784). From bugzilla we find out that:

"*A flaw was found in the way the Hotspot component of OpenJDK performed range check elimitation. An untrusted Java application or applet could use this flaw to bypass Java sandbox restrictions.*"

The oracle id 8264066 can be found in the [OpenJDK github repository](https://github.com/openjdk/jdk/commit/e1051ae0695f14802f192a5aa58ff2365a5ef753). The discrete commit message "*8264066: Enhance compiler validation*". The commit contains a quite simple fix that indicate where the vulnerability can be found.

![](https://dim.mcusercontent.com/cs/ddd7ae486331199e51e491d97/images/18e5c322-775e-d99e-15f1-af20f834b959.png?w=564&dpr=2)

The byte-code is interpreted by c1 as a linked list of structures where the last block use NULL as the last block indicator. The attacker can create a malformed byte-code block to corrupt the memory of the c1 compiler.

Previous attack methods on JIT compilers have abused the JIT by preparing a heap through [JIT spraying](https://en.wikipedia.org/wiki/JIT_spraying). This preparation method can aid an attacker to defeat Address Space Layout Randomisation (ASLR).

This is significant because it means that we must understand the risks we take when we introduce a new JRE environment in to an organisation. The possibility of this type of attack indicate that pre-compiled third party components must be carefully checked, not only for its classes and intents but also for its byte-code structure, before being allowed in to the environment.
