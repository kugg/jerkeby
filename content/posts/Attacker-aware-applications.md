title: "Attack aware application"
date: "2022-01-07"
draft: true
---

Last year ended in a rush to patch the Log4j vulnerability.
Most organisations lack controls to prevent or even detect exploitation of log4j.
What if there is a portable approach to detecting anomalous application behaviour?
Introducing the attack aware application.


Setting up applicaiton level SLA, time to detect and fix vulnerabilities.
Reducing the investigation time.
Reduce the amount of decisions taken for running applications.

# The risk aware application
Wiliam Jardine from F-Secure wrote an article on how to implement [attack aware applicaitons in practice](https://labs.f-secure.com/blog/application-level-purple-teaming)
Most blueteamers/SoC aim their focus on plattforms security and known [IoC's with high confidence](https://twitter.com/Kuggofficial/status/1476542086178562052)

The Attack Aware application pattern applies to all kinds of applications while most public research of today is aimed towards web-applications.
Attack aware application patterns are currently applied to payment solutions, insurance and any type of machine learning.
Here are some examples:
- In insurance it's used to detect fraud in large datasets.
  - Predictive models and heuristics are used to find hidden relationships and aanomalies.
- In finance/banking/payment it's mandated to detect and react to risk behaviour such as money laundering, support of terrorism and tax avoidance.
  - Testpoints are, type of service, clientele and distribution channel as well as geographical aspects.
- In the automotive industry it is used to detect faulty parts and risk of collision.
  - Sensordata from runtime operations and self diagnostics.

These are all domain specific usages of risk aware appliction patterns. In all of these cases manual oversight of the decision making process is very expensive. The data load is too high and the time to process too slow if humans are involved.

The goal of using this pattern is to:
1. Reduce alarm fatigue for investigators
2. Create automatic action from events
3. Improve log coherence and remove blockers from the investigation

The purpose of this method is to make use of applicaiton logs for improving the software security and quality. 

The pattern:
1. Qualify one or multiple testpoints for one risk event indicator.
2. Collect risk event indicators in a centralized function (Risk API).
3. Assign a confidence level and risk score to the event. 
4. Group risk events together and send to central log function.
5. Choose an action relevant for the given event, riskscore and confidence.

In a microservice arcitecture each application is too small to make their own risk risk decisions therefore a centralized function called a Risk API is used.

## Feature specific risks and threat modelling
In threat modelling, controls are typically used to mitigate a risks. Often risks cannot be fully mitigated using one single control. Using the risk aware application pattern multiple detections points and actions can be applied to one single risk. The purpose is to create security in depth.

During the threat modelling exercice the confidence level of each detection point for a given risk is proposed, but the same score can be adjusted later to make risk scoring coherent across the system.

## Generic risks
The Owasp Application Sensor framework present a few generic detection points and actions for web applications. 

### The detection point
Application sensor offer a list of [detection points](https://owasp.org/www-project-appsensor/#div-detection_points)

The first two detection points are `RequestExceptions`. They are intended to describe the event of a browser using the wrong verb or command, for instance using OPTIONS instead of GET in the HTTP request.

### The actions
A proposed action is matched to the detection event. There are four action types and in most cases more then one action is needed per detection event.

Here is a proposed list of [response actions](https://owasp.org/www-project-appsensor/#div-response_actions) from OWASP.

In this list I try to summarize the action types and action examples:

- Silent: User(s) unaware of any application change
  - Log the event
  - Alert an event and highlight the logged entry
  - Toggle loglevel in the application
- Passive: Process altered, but user(s) may still continue to process completion
  - User notification
  - Force the client to respond to a captcha
  - Change timing (see [Kvicksand](https://www.jerkeby.se/newsletter/posts/capturing-robots-with-kvicksand/))
- Active: Functionality reduced or disabled
  - Reject invalid input
  - Account lockout
  - Disable functionality
- Intrusive: Non-malicious action on user's system
  - Collect data from user using client-side code

## Feature specific risks and threat modelling
In threat modelling, controls are typically used to mitigate a risks. Often risks cannot be fully mitigated using one single control. Using the risk aware application pattern multiple detections points and actions can be applied to one single risk. The purpose is to create security in depth.

During the threat modelling exercice the confidence level of each detection point for a given risk is proposed, but the same score can be adjusted later to make risk scoring coherent across the system.

## Advanced events and actions
The detection points described by OWASP are user centric and generic, they do not venture out to platform limitations or exceptions caught by the applicaiton.

As the developers are adjusting to a risk aware mindset it will become more apparent to them how to write code that reduce risk of vulnerabilities and detect risks.

Once you have applied detections points for generic risks and started exploring feature specific risks, you have acquired capacity to detecting unknown risk (aka zero day risks). 

## Platform security in the application
A vast majority of the tools that do intrusion detection reside on a platform level. They may have the capacity to rudimentarily inspect packets but have little understanding of application state (WAF and IDS).

However to an application it may be a clear indication of a intrusion if an ImageMagick library throws IO-Exceptions (Remote Code Execution). Similarly if a PDF renderer get a "Connection Timeout", this must mean that the PDF renderer was tricked in to trying to reach a website (SSRF).

Adding platform security detection capacity to the application layer improves portability, the environment security should not rely on using a specific cloud provider.

Exceptions to look out for:
- Permission errors
  - May occur if an attacker attempts to write to a directory where they don't have permission (Path Traversal).
- Connection Refused and Connection Timeout
  - Occurs if an attacker manages to create a request from the server to a destination that is not available (Server Side Request Forgery).
- Out Of Memory
  - An attacker attempts to load additional functionality in to the application and cause a crash (Remote Code Execution).
- Illegal Argument Exception
  - If this error occurs after input validation the input validation has been bypassed by the attacker.
- Outlier Round Trip Time
  - If an attacker is able to execute arbitary code or use existing functionality to extract sizeable amount of information from the server the request tend to produce longer response times.
- Database errors
  - The database reports a syntax error in a query. This is a clear indicator that a user has been able to alter a database query (SQL injection).

## Manual Actions
Some actions will require manual invervention. It's vital that such events contain qualifying information. In this graduation process the event must automatically collect information from the SIEM solution based on a criteria defined by the Secrutity operation Center (SoC).
The qualificaiton criteria is a clearly defined list of information points the SoC person needs to take action.

Here is an example:
```
{
   "EventId": String,
   "EventName": String,
   "EventType": String,
   "EventTime": Int,
   "RiskScore": Int,
   "RiskProbability": Int,
   "Breach": Bool,
   "Request": String,
   "ChildEvent": String,
   "Description": String,
   "Action": String,
   "Instruction": String,
   "EventOwner": String
}
```
Security Operation Center staff are used to working with clear Indicators of Compromise. The SoC tend to focus on platform security indicators based on the mitre attack framework. The IoC's produced by the risk aware applicaitons must be equally mature to be accepted in to the SoC.

There are two types of queries, production queries and investigation queries. A produciton query must be graduated using the qualification process. A production query should only ever produce at most one unique result per request and its indicators may include an event hash/identifier to find further information. The production query must result in an event with a high RiskProbability.

An investigaiton query is a tool to simplify the process of a manual step in the investigation. This can be a query that collect all available information about a given client for deeper scrutiny. Perhaps a query to find other accounts with similar criteria as an offending account.
The investigation query can have a low reliability, it can have manual steps and is typically used to find similarities where there can be multiple results.

## Harmonizing the logs
A pre-reuisite for being able to do automatic decisions based on logging is that the log methodology and fomrat is harmonised. This means that a central decision needs to be made on a method of log transportation.

Some prefer to log to stdout, others use logging frameworks such as syslog-ng for their logs. The main thing here is to use one single modus for every application.

The time on each server must be synchronised so that log events canbe traced across multiple machines.

The log events must be formated in the same manner regardless of the application layer. Using a structured serialisation format such as JSON may improve structure drasticly. The NCSA format used by Apache is quite common among traditional Unix applications.  

Application errors tend to vary in format between architectures, attempting to structure the output of backtraces may save time in a later stage. Immagine that you will want to do fuzzy matching in similar field types in defferent applications for exceptions of a certain type. The fuzzy match will be easier to perform if the fileds data use the same encoding and naming schemes.

Last but not least, logs have a severity indicator or loglevel. These are "FATAL", "WARNING", "ERROR", "INFO" and "DEBUG". Adhering to a logging policy means that the right type of log message is grouped or visible at the right time. It's possible that the effective production loglevel can toggle over time for a given application if the Risk API decide that such actions are needed.

Detecting the log4shell vulnerability was easier on an application running with loglevel "DEBUG" compared to one running on "INFO".

## Roadmap

Here is an example of a roadmap to introduce application aware logging:
1. Harmonize the log format accross the application
2. Add rudimentary detection query in RISK API or logging service (Splunk/ELK)
3. Introduce application risk awareness in to the threat model
4. Add common generic detection points for new features
5. Add feature specific detections for mature developers
6. Qualify events in Risk API/ Logging service for production alarms (to SoC)
7. Create actions for automatic ticket creation

## Thank you

For this issue I want to thank Lars Gråmark and Alexander Mohlin for taking the time to discuss applicatino logging potentials.

## Future work

Investigate how to use [tracing](https://opentracing.io) and applicaiton profiling to speed up the investigation phase.
