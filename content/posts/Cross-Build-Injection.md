---
title: "Cross Build Injection: Supplychain backdoors and mitigations"
date: "2025-02-18"
draft: true
---
## Introduction

The rise of software supplychain attacks have shown us that ensuring the integrity of build processes is crucial. Cross Build Injection occurs when one compilation output contaminates or manipulates builds to introduce backdoors. These attacks are subtle and can be difficult to detect, making them a serious security concern. In this post, we explore various cross build injection techniques and discuss architectural mitigations to protect build environments.

## Cross Build Injection Techniques

### 1. Build Time Contamination

Build helpers or scripts can modify compile-time artifacts, introducing unexpected modifications.
[Example: Reflections on Trust in the Software Supply Chain by Jeremy Long](https://www.youtube.com/watch?v=GXdHoxDgAYI)

### 2. Hostile Tests

Malicious or misconfigured test cases can write files that affect the resulting build artifact, leading to unintended behavior in production.

[Example: XZ backdoor announcement on oss-security by Andres Freund](https://www.openwall.com/lists/oss-security/2024/03/29/4)

### 3. Evil Dependencies

A build may unknowingly depend on a shared library that contains backdoors or malicious functionality due to unvetted or compromised dependencies.

[Example: boltdb typosquat backdoor library found by Socket](https://socket.dev/blog/malicious-package-exploits-go-module-proxy-caching-for-persistence)

### 4. Cache Poisoning

Build caches or shared repositories (e.g., `~/.m2/repository` for Maven) can be contaminated with manipulated build outputs, leading to the propagation of compromised artifacts in future builds.

[Example: Christoffer Jerkeby & Jonny W1th4d - Playing hide and seek in Java land](https://www.youtube.com/watch?v=Wcz-Gvm-468)

## Mitigation Strategies

### 1. Build in Isolated Environments

To prevent cross-contamination, each component should be built in an isolated environment, such as a dedicated Docker container. This mitigates:

- **Build Time Contamination** by ensuring that build helpers cannot modify previous build artifacts.
- **Cache Poisoning** by ensuring that each build starts with a clean slate and cannot access or overwrite previous builds.

### 2. Recreate the Build Environment for Every Build

A clean build environment must be re-created from scratch each time. Avoid storing cached builds or sharing persistent states across builds. This counters:

- **Build Time Contamination** by preventing lingering modifications from previous builds.
- **Cache Poisoning** by ensuring a fresh build process free from tampered files.

### 3. Running Test Steps Separately

Test execution should be isolated, and the test environment should be built separately from the main build process. This mitigates:

- **Hostile Tests** by ensuring that test cases cannot influence the final production artifact.

### 4. Reduce the amount of dependencies

When designing software, coordinate dependency qualification and use libraries with [high credibility in OpenSSF](https://securityscorecards.dev/)
This reduces the risk of any type of Cross Build Injection.

### 5. Cryptographic Signing for Shared Libraries

While cryptographic signing can help verify dependencies, it has limited impact on build time contamination. Given that many external packages already implement signing, introducing it internally adds overhead without fully mitigating risks from manipulated build processes. Instead, focus should be placed on strict environment isolation and monitoring.


## Conclusion

Cross build injection presents a significant security risk, but by enforcing strict isolation and ensuring fresh builds, organizations can protect their software supply chain. These mitigations help maintain the integrity of software builds, reducing the risk of backdoor introduction and ensuring the security of deployed applications.

## Shouts

Jonny W1th4d for relentless R&D in [JarPlant](https://github.com/w1th4d/JarPlant)
