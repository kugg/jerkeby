---
title: "History of the ROP"
date: "2021-11-02"
draft: false
---


# From 0 to eternity

Hi buddies! I'm now celebrating one month down in my one man megacorp. I have decided to take some time off from client work to study and write about the future of memory corruption vulnerabilities. I want to help you understand the risks, and known controls for C programs. I think it's easier to remember all of this in a story context. This newsletter is divided in two. This one covers the history of memory corruption attacks as I remember it. The second covers an evaluation of new mitigations.

## The threat of memory corruption

I vividly remember the spring of 1997. I had decided I was going to be a hacker. Started lending programming books about C in our local library and was reading up on the mailing list Bugtraq.
But most vividly I remember getting a hold of and printing the Phrack file by Aleph One "Smashing The Stack For Fun And Profit" that had been released during the winter of 1996 (http://phrack.org/issues/49/14.html).
A complete gamechanger to hacking. From a generation of desperately brute forcing teenagers arose a tribe of warriors ready to smash that stack. We started learning how compilers and assembler work (still lots to learn there). "Smashing The Stack..." explained how to overwrite the memory of a computer program with your own instructions using unexpectedly long inputs. Back then there was only one known memory corruption vulnerability: the buffer overflow. Since then plenty of new memory corruption vulnerability classes have surfaced.

By the summer of 1997 stack smashing and buffer overflows was on everybody's lips, at least if you were part of the Bugtraq community (https://seclists.org/bugtraq/1997/Apr/125). A hacker named Solar Designer posted a Linux kernel patch to the kernel community mailing list for adding non-executable stack support (http://lkml.iu.edu/hypermail/linux/kernel/9706.0/0341.html) The same hacker diligently pointed out that the patch could be circumvented using a new technique called return to libc http://lkml.iu.edu/hypermail/linux/kernel/9706.0/0234.html. The response to the non-executable stack patch was that even though you can mark the stack as non-executable and attackers can use already existing memory and libraries and jump to existing code, that would be executable and run from there. Solar Designer then wrote a proof of concept exploit and published the first return to libc attack in August to the public "full disclosure" mailing list Bugtraq (https://seclists.org/bugtraq/1997/Aug/63).

Without knowing it, Tim Newsham describes the ROP attack on Bugtraq while providing criticism towards the Non-eXecutable (NX) stack approach: https://seclists.org/bugtraq/1997/Apr/129
```
...
The ability to overwrite the stack with arbitrary data
is very powerful.   Besides return addresses the stack
is also used to save register values and to hold variables.
Most programs have segments of code that look like:

     restore some registers from the stack
     return from subroutine

If an attacker knows the address of such code, he can
provide register contents on the stack and set the return
address to point to this code.  When the next return
happens, registers are set with whatever values he put
on the stack, another return is done pulling another
address off the stack.  Say the next return address on
the stack pointed to code that trapped to the system call
vector.  We just put arbitrary values in registers and
then trapped to the system - we have the ability to
do arbitrary system calls.  All the code that was executed
was from the code segment.

By controlling the stack, an attacker can cause execution
to thread through segments of existing code with a great
degree of freedom.  The attacks have to accurately compute
the location of stack positions and code addresses so
the attack is definitely a lot harder than the cookie-cutter
stack overflows that you see today, but its still
``just a simple matter of coding''.
...
```

## The ROP chain

In late 2001 a Phrack article by Nergal (http://phrack.org/issues/58/4.html) dealt with recent mitigations and the fully featured ROP attack is born. This time the Solar Designer idea of "return to libc" had been refined with a chain of calls to various libraries. This is the birth of the ROP chain, which is the actual topic of this article.

Today producing a ROP chain means to scan through a binary using a ROP chain search tool and having it generated automatically. The ROP search tool looks through each offset of the program's `.text` section and tries to find patterns that look like assembler instructions that could be useful for an attack. All patterns end with the assembler instructions CALL, JUMP or RET. These are called gadgets. For now we will only focus on the most popular one the RET gadgets. RET/Return is the last thing that happens in a function. After that the program looks for the next value on the stack and uses that address to skip back to wherever the function was called from. This stored address that RET looks for is called a base pointer. The attacker will pile a bunch of addresses on the stack to be used as false base pointers.

## Mitigations

In 2000-2001 two mitigations against the buffer overflow attack were devised. First came the Non-eXecutable (NX) stack as suggested already back in 1997 and after that the Address Space Layout Randomization (ASLR) from the PaX project in 2000. PaX for Linux was initially written by a Hungarian hacker named pipacs and the PaX Team. Read more about pipacs backstory here: http://phrack.org/issues/66/2.html#article.
A few years later the kernel security mitigation space was dominated by another researcher: Brad Spengler, aka Spender. Spender made a patch set for Linux called grsec. It supported an array of defense and detection mechanisms. Spender also worked on improving and packaging PaX as part of grsecurity. His patches were considered well written, stable, secure and very inventive. The Linux kernel community however was not keen on hardening at all, because it meant making feature tradeoffs and it would create complications. (Read more about that story here https://lwn.net/Articles/721848/)

The idea is that the addresses used in the `.data` section of a binary program can be randomized so that an attacker that has been able to create a buffer overflow would not know where their code would end up. 

Things are moving slowly in the Linux world due to personal conflicts between Spender and parts of the Linux kernel community. When Intel announced the Itanium ia64 architecture they had introduced hardware support for the NX bit proposed back in 1997 by Solar Designer. Suddenly the kernel world sprung to action with the support from Intel. The two security features from the PaX project were merged in 2003-2004 (in kernel version 2.6.8). As a comparison, Windows integrated support for the same security features in October 2009 with the release of Windows 7.

In '97 the Bugtraq people summarized the mitigations. These were the solutions proposed (https://seclists.org/bugtraq/1997/Apr/125):

1. Kernel exec stack permission (NX): Proposed by Solar Designer
2. Function wrapping, a method that replaces one function with another at compile or runtime to limit the damage. By Massimo at IBM.
3. Automatic code review tools: Brian Marick's GCT (Generic Coverage Tool), CodeCenter and Purify
4. Compiler modifications, bounds checking (https://web.archive.org/web/20160326081542/https://www.doc.ic.ac.uk/~phjk/BoundsChecking.html)
5. Controlled interpreters like Lisp and Ada (what we today call scripting languages). This is done in Rust in the Linux kernel today.
6. "The problem is not that privileged code is insecure. The problem is that there is too much privileged code." Today this is adressed with micro kernel initiatives and Memory Space Separation on MMU level
7. Stack canaries in GCC introduced as a patch-set by StackGuard (https://www.usenix.org/legacy/publications/library/proceedings/sec98/full_papers/cowan/cowan.pdf)

All of the above methods have been tried and all of them have contributed to limiting the impact of buffer overflow attacks. 
I mentioned that ASLR was added in 2004. Buffer overflows continued to be a successful attack vector due to a new circumvention technique called the NOP sled (https://en.wikipedia.org/wiki/Buffer_overflow#NOP_sled_technique). The stack canary became default in modern compilers - a method that checks that the stack is intact by checking a random value before executing a function and then checking that the same intact value is intact just after returning. This method has been circumvented by using format string vulnerabilities to leak canary values. On Windows an attacker can cause exceptions that trigger the System Interrupt Handler that does not use stack canaries.

## Architecture specific approaches

At this point in time hardware supported methods are not being discussed by the software focused security community. Earlier attempts had been made towards building memory tagged MMU's and cryptography into memory management.
These research concepts are picked up today by ARM who use pointer authentication in the M1 chips from Apple which limits ROP significantly. (https://www.usenix.org/system/files/login/articles/login_summer19_03_serebryany.pdf) My concern with this is that security benefits will only be served on certain architectures where approaches differ from one architecture to another. A responsibility shift that separates platforms to the extent that it's impossible to determine risk for software or code alone. As an example the Rowhammer attack was mitigated by using ECC memory checksums to protect RAM from the attack but Intel recently removed the feature again (https://www.realworldtech.com/forum/?threadid=198497&curpostid=198647). On the other hand using every available means for protection can help the end user and organisation, but then ties are made to one hardware architecture.

## Path of most reward

It's notable that the resistance against hardening has led to a slow pace for bug class mitigation and an increasing interest in finding new bugs.
A Linux kernel developer recently complained to me that there are six security researchers looking for bugs on every developer trying to fix and mitigate the bugs. It's all in the incentive and rewards. We see a rapid development where there are short term incentives and goals for hardware related security. Just look at how swiftly the bugs Spectre and Meltdown were addressed with the support from the architecture companies. The problem of attention and reward in software warrants it's own newsletter post.

## A long way back to safety

When we try to eradicate a bug class we try to attack the root cause if possible but sometimes that means taking away the core functionality of a language and in this case memory corruption in itself is a core issue in the C programming language that probably never will be solved. Even though you don't write C code you still use C optimizations in all available frameworks and languages. First, C is used in core functionality of languages like PHP, Perl, Java, Python, Erlang etc. Its used to write compilers, memory managers and time critical operations like precise floating point math. My main concern is that operating system kernels, virtualization environments and web servers are written in C, a language that has so many inherent risks that despite 20 years of active mitigation work has only been able to partly address the issues found in '97.

I want to give a final mention to the mitigation method RELRO that maps the `.data` and `.bss` sections are mapped as read-only. This means that a buffer overflow can no longer overwrite the execution stack.
Find out more about RELRO here: http://phrack.org/issues/66/2.html#article
Unfortunately, today buffer overflow is no longer the only fish in the "memory corruption sea". Integer overflow, heap based overflows, double free and null pointer dereference are a few new methods to cause trouble when the stack is read-only.

## Perfect is the enemy of good

The above mentioned protection mechanisms have all served in reducing the probability of a successful memory coruption attack. This newsletter post is about assessing and introducing yet another measure for lowering the impact of ROP attacks. 

Perfect is the enemy of good, meaning that if we would have expected to find the perfect solution and introduced it first we wouldn't have had any mitigations (which is bad). Instead I think it's a better approach to attempt a layered security by experimenting and improving on existing solutions. However this approach requires historic knowledge and experience to understand the motive behind every little knob on the control board.

I want to give a final mention to the mitigation method RELRO that maps the `.data` and `.bss` sections are mapped as read-only. This means that a buffer overflow can no longer overwrite the execution stack.
These are the basic options in GCC taken from https://wiki.debian.org/Hardening#Notes_on_Memory_Corruption_Mitigation_Methods. They constitute a foundation of the compiler controlled hardening features.
Understanding and using these options drastically improves security in C programs in various ways.
```
-Wall -Wextra
  Turn on all warnings.
-Wconversion -Wsign-conversion
  Warn on unsign/sign conversions.
-Wformat­security
  Warn about uses of format functions that represent possible security problems
-Werror
  Turns all warnings into errors.
-arch x86_64
  Compile for 64-bit to take max advantage of address space (important for ASLR; more virtual address space to chose from when randomising layout).
-fstack-protector-all -Wstack-protector --param ssp-buffer-size=4
  Your choice of "-fstack-protector" does not protect all functions (see comments). You need -fstack-protector-all to guarantee guards are applied to all functions, although this will likely incur a performance penalty. Consider -fstack-protector-strong as a middle ground.
  The -Wstack-protector flag here gives warnings for any functions that aren't going to get protected.
-pie -fPIE
  Position independent executable this is an enable for ASLR memory mapping by the kernel
-ftrapv
  Generates traps for signed overflow
-D_FORTIFY_SOURCE=2 ­O2
  Buffer overflow checks. See also difference between =2 and =1
­-Wl,-z,relro,-z,now
  RELRO (read-only relocation). The options relro & now specified together are known as "Full RELRO". You can specify "Partial RELRO" by omitting the now flag. RELRO marks various ELF memory sections read­only (E.g. the GOT)
-Wl,dynamicbase
  Tell linker to use ASLR protection
-Wl,nxcompat
  Tell linker to use DEP protection
```

## return 0

I have now talked about how the ROP attack came to be and the reasons for the slow mitigations in early 00s. In the next newsletter I'll introduce a new ROP mitigation technique released with GCC 11 that you may see in your distros from next year. The GCC option is called zero-call-used-regs and I want to share some risks as well as the benefits of using it. I have collected some statistics of its performance.

For this edition I would like to thank Linus Walleij who explained the social aspects of the kernel development community in the early 2000's. And Calle Svensson (https://twitter.com/ZetaTwo) for following along and arguing for various ROP chain options. I want to thank Laban (https://twitter.com/LabanSkoller) who corrected my spelling again. Andreas (https://twitter.com/hyyyy) who pointed out that I was wrong about ASLR and gave some more context to RELRO.

Stay tuned for the follow-up!

