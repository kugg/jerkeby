---
title: "History of the ROP"
date: "2021-11-02"
draft: false
---


# From 0 to eternity

Hi buddies! I'm now celebrating one month down in my one man megacorp. I have decided to take some time off from client work to study and write about the future of memory corruption vulnerabilities. I want to help you understand the risks, and known controls for C programs. I think it's easier to remember all of this in a story context. I got in to this type of hacking in just about the right time. 
This newsletter is divided in two. This one covers the history of memory corruption attacks as I remmber it, the second cover an evaluation of new mitigations.

## The threat of memory corruption

I vividly remember the spring of 1997. I had deciced I was going to be a hacker. Started lending programming books about C in our local library and was reading up on the mailing list Bugtraq.
But most vividly I remember getting a hold of and printing the phrack issue by Aleph one "Smashing the stack for fun and profit" that had been released during the winter of 1996 (http://phrack.org/issues/49/14.html).
A complete gamechanger to hacking. From a generation of desperately bruteforcing teenagers arose a tribe of warriors ready to smash that stack. We started learning how compilers and assembler works (still lots to learn there). Smashing the stack explained how to overwrite memory of a computer program with your own instructions using unexpectedly long inputs. Back then there was only one known memory corruption vulnerability: the buffer overflow. Since then plenty of new memory corruption vulnerability classes have surfaced.

By the summer of 1997 stack smashing and buffer overflows was on everybodys lips, at least if you were part of the bugtraq community (https://seclists.org/bugtraq/1997/Apr/125). A hacker named Solar Designer Linux posted a Linux Kernel patch to the kernel community mailing list for adding executable stack support (http://lkml.iu.edu/hypermail/linux/kernel/9706.0/0341.html) The same hacker diligently pointed out that the patch could be circumvented using a new technique called return to libc http://lkml.iu.edu/hypermail/linux/kernel/9706.0/0234.html. The response to the non executable stack patch was that even though you can mark the stack as non executable and attacker can use already existing memory and libraries and jump to existing code that would be executable and run from there. Solar Designer then wrote a proof of concept exploit and published the first return to libc attack in August to the public "full disclosure" mailininglist bugtraq (https://seclists.org/bugtraq/1997/Aug/63).

Without knowing it Tim Newsham describes the ROP attack on bugtraq while providing critiscism towards the NX approach: https://seclists.org/bugtraq/1997/Apr/129
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

In late 2001 a Phrack article by Nergal (http://phrack.org/issues/58/4.html) deal with recent mitigations and the fully featured ROP attack is born. This time the Solar Designer idea of "return to libc" had been refined with a chain of calls to various libraries. This is the birth of the ROP chain, which is the actual topic of this article.

Today producing a ROP chain means to scan through a binary using a ROP chain search tool and having it generated automatically. The ROP search tool looks through each offset of the programs .text seciton and tries to find patterns that looks like assembler instructions that could be useful for an attack. All patterns end with the assembler instructions CALL, JUMP or RET. These are called gadgets. For now we will only focus on the most popular one the RET gadgets. RET/Return is the last thing that happens in a function. After that the program looks for the next value on the stack and use that address to skip back to wherever the function was called from is. This stored address that RET looks for is called a Base Pointer. The attacker will pile a bunch of addresses on the stack to be used as false basepointers.

## Mitigations

In 2000-2001 two mitigations angainst the buffer overflow attack was deviced. First came the Non eXecutable (NX) stack as suggested already back in 1997 and closely after the Address Space Layout Randomization from the PaX project.
At that time the kernel security mitigation space was dominated by one single researcher: Brad Spengler, aka Spender. Spender made a patch set for Linux called the grsec & PaX patches. Hes patches were well written stable, secur and very inventive. The Linux kernel community however was not keen on hardening at all, because it meant making feature tradeoffs and it would create complications. (Read more about that story here https://lwn.net/Articles/721848/)

The idea is that the addresses used in the .data section of a binary program can be randomized so that an attacker that has been able to create a buffer overflow would not know where their code would end up. 

Things are moving slowly in the Linux world due to perosnal conflicts between Spender and parts of the Linux Kernel community. When Intel announced the ia64 architecture they had introduced hardware support for the NX bit proposed back in 1997 by Solar Designer. Suddenly the kernel world sprung to action with the support from Intel the two security features from the PaX project was merged in 2003-2004 (2.6.8). As a comparison Windows integrated support for the same security features in October 2009 with the release of Windows 7.

In 97 the bugtraq people summarized the mitigations, these were the solutions proposed (https://seclists.org/bugtraq/1997/Apr/125):

1. Kernel exec stack permission (NX): Proposed by Solar Designer
2. Function wrapping, a method that replace one function with another at compile or runtime to limit the damage. Massimo at IBM
3. Automatic code review tools : Brian Marick's GCT (Generic Coverage Tool), CodeCenter and Purify
4. Compiler modifications, bounds checking, (https://web.archive.org/web/20160326081542/https://www.doc.ic.ac.uk/~phjk/BoundsChecking.html)
5. Controlled intepretters like LISP and ADA (what we today call scripting languages). This is done in RUST in the Linux kernel today
6. "The problem is not that privileged code is insecure. The problem is that there is too much privileged code.". Today this is adressed with Microkernel intitiatives and Memory Space Separation on MMU level
7. Stack canaries in gcc introduced as a patch-set by StackGuard (https://www.usenix.org/legacy/publications/library/proceedings/sec98/full_papers/cowan/cowan.pdf)

All of the above methods have been tried and all of them have contributed to limiting the impact of buffer overflow attacks. 
I mentioned that ASLR was added in 2004. Buffer overflows continued to be a successfull attack vector due to a new circumvention technique called the NOP sled (https://en.wikipedia.org/wiki/Buffer_overflow#NOP_sled_technique). The Stack canary bcame default in modern compilers, a method that checked that function was intact by comparing a random value before executing a function and the same intact value just after returning. This method has been circumvented by using format string vulnerabilitites to leak canary values, on Windows an attacker can cause Exceptions that trigger the System Interup Handler that does not use stack canaries.

## Architecture specific approaches

At this point in time hardware supported methods are not being discussed by the software focused security community. Earlier attempts had been made towards building memory tagged MMU's and cryptography in to memory management.
These research concepts research is picked up today by ARM who use pointer authentication in the M1 chips from Apple which limits ROP significantly. (https://www.usenix.org/system/files/login/articles/login_summer19_03_serebryany.pdf) My concern with this is that security benefits will only be served on certain architectures where approaches differ from one architecture to another. A responsibility shift that separate platforms to the extent that it's impossible to determine risk for software or code alone. As an example the Rowhammer attack was mitigated by using ECC memory checksums to protect ram from the attack but recently removed the feature again (https://www.realworldtech.com/forum/?threadid=198497&curpostid=198647). On the other hand using every available means for protection can help the end user and organisation, but then ties are made to one hardware architecture.

## Path of most reward

It's notable that the reistance against hardening have led to a slow pace for bug class mitigation and an increasing interest in finding new bugs.
A Linux Kernel recently complained to me that thre are 6 security researchers looking for bugs on 1 developer trying to fix and mitigate the bugs. It's all in the incentive and rewards. We see a rapid development where there are short term incentives and goals for hardware related security. Just look at how swiftly the bugs Spectre and Meltdown was addressed with the support from the architecture companies. The problem of attention and reward in software warrants it's own newsletter post.

## A long way back to safety

When we try to erradicate a bug class we try to attack the root cause if possible but sometimes that means taking away the core functionality of a language and in this case menmory corruption in itself is a core issue in the C programming language that probably never will be solved. Even though you dont write C code you still use C optimizations in all available frameworks and languages. First, C is used in core functionality of languages like PHP, Perl, Java, Python, Erlang etc. Its used to write compilers, memory managers and time critical operations like precise floating point math. My main concern is that operating system kernels, virtualization environments and web servers are written in C. A language that has so many inherent risks that despite 20 years of active mitigation work have only been able to partly address the issues found in 97. 

## Perfect is the enemy of the good

The above mentioned protection mechanisms have all served in reducing the probability of a successful memory corurption attack. This newsletter post is about assessing and introducing yet another measure for lowering the impact of ROP attacks. 

Perfect is the enemy of the good meaning that if we would have expected to find the perfect solution and introduced it first we wouldnt have had any mitigations (which is bad). Instead I think it's a better approach to attempt a layered security by experimenting and improving on existing solutions. However this approach requires historic knowledge and experience to understand the motive behind every little knob on the controlboard.

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
  For ASLR
-ftrapv
  Generates traps for signed overflow (currently bugged in gcc)
-D_FORTIFY_SOURCE=2 ­O2
  Buffer overflow checks. See also difference between =2 and =1
­-Wl,-z,relro,-z,now
  RELRO (read-only relocation). The options relro & now specified together are known as "Full RELRO". You can specify "Partial RELRO" by omitting the now flag. RELRO marks various ELF memory sections read­only (E.g. the GOT)
-Wl,dynamicbase
  Tell linker to use ASLR protection
-Wl,nxcompat
  Tell linker to use DEP protection
```
These are the basic options in GCC taken from https://wiki.debian.org/Hardening#Notes_on_Memory_Corruption_Mitigation_Methods

## return 0

I have now talked about how the ROP attack came to be and the reasons for the slow mitigations in early 00s. In the next newsletter I'll introduce a new ROP mitigation technique released with GCC 11 that you may see in your distros from next year. The GCC option is called zero-call-user-reg and I want to share some risks as well as the benefits of using it. I have collected some statistics of it's performance.

For this edition I would like to thank Linus Walleij who explained the social aspects of kernel development community in the early 00s. And Calle Svensson (@ZetaTwo) for following along and arguing for vaious ROP chain options.

Stay tuned for the follow up!

