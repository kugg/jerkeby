---
title: "Gadget reduction using zero-call-user-regs"
date: "2021-11-08"
draft: true
---

TODO: Decide if we should focus on used-gpr or make a full comparison
TODO: Redo the bash experiment but with busybox
TODO: Write something about maxploits attack and how it relates risk wise
TODO: Attempt to reduce length by using external references

# Gadget reduction using zero-call-user-regs

## Introduction

If you are unfamiliar with the concepte of ROP, you may read up in https://www.jerkeby.se/newsletter/posts/history-of-rop/

I have decided to cover x64 intel assembler because thats the platforms I have available.

As I was writing this article the Linux Kernel v5.15 was released including the feature discussed.

Maxploit released a new remote heap buffer overflow in that affect the 5.15 kernel in the TIPC protocol that can lead to kernel RCE if the attacker has access to appropriate ROP gadgets.
https://www.sentinelone.com/labs/tipc-remote-linux-kernel-heap-overflow-allows-arbitrary-code-execution/

## The gadget
In 2007 a researcher named Hovav Shacham published a paper with the title "The Geometry of Innocent Flesh on the Bone" that attempts to refine the return-to-libc attack by defining an algorithm for finding "gadgets" in legitemate code.
https://hovav.net/ucsd/dist/geometry.pdf

The paper is the first to coin the phrase ROP attack but it also does what had been described by Tim Newsham as "a matter of simple coding" ten years earlier.
The technique was not novel, but the implementation was. Before we move in to gadget land let it sink in. The idea of how to do automated rop searches took ten years to be implemented. This is a reminder that our future may be implementations of our greatest ideas of today.

Back again to the gadgets. The compiler of a c program is a machine code generator that create machine code according to the format appropriate for the target CPU architecture and operatingsystem environment. IN a dynamically linked program, building blocks make use of compiled code libraries that are linked from external sources. In a statically linked program all logical functionalities is contained within the program. The Linux kernel is typically statically linked because it cannot depend on a specific outside library or component.

From the Shacham paper:
```
To understand how there exist code sequences in libc that were not placed there by the compiler,
consider an analogy to English. English words vary in length, and there is no particular position on
the page where a word must end and another start. Intel x86 code is like English written without
punctuation or spaces, so that the words all run together.2 The processor knows where to start
reading and, continuing forward, is able to recover the individual words and make out the sentence,
as it were. At the same time, one can make out more words on the page than were intentionally
placed there. Some words will be suffixes of other words, as “dress” is a suffix of “address”; others
will consist of the end of one word and the beginning of the next, as “head” can be found in “the
address”; and so on.

Two instructions in the entrypoint ecb_crypt are encoded as follows:

f7 c7 07 00 00 00	test $0x00000007, %edi
0f 95 45 c3		setnzb -61(%ebp)

Starting one byte later, the attacker instead obtains:

c7 07 00 00 00 0f	movl $0x0f000000, (%edi)
95			xchg %ebp, %eax
45			inc %ebp
c3			ret

How frequently such things occur depends on the characteristics of the language in question, what
we call its geometry. And the x86 ISA is extremely dense, meaning that a random byte stream can
be interpreted as a series of valid instructions with high probability [3]. Thus for x86 code it is quite
easy to find not just unintended words but entire unintended sequences of words. For a sequence
to be potentially useful in our attacks, it need only end in a return instruction, represented by the
byte c3

```

## My experiment with zero-call-user-regs

In the upcoming release of GCC 11 there is a feature that does cleans up registers used by a function prior to returning. One usecase for the feture can be to reduce ROP attacks.
I want to determine what risks are actually mitigated by the -fzero-call-user-regs option in gcc. The intent of the patch is to reduce the probability of a successful ROP attack.

### The options zero-call-user-regs

This is the GCC patch we want to use:
https://github.com/gcc-mirror/gcc/commit/d10f3e900b0377b4760a090b0f90371bcef01686

### The comparison

I decided to use a rop gadget search tool to compare a binary with and without this feature.
The concept behind the mitigation is to clear all used registers prior to calling return in a given function.

Essentially this is a comparison of the quality of ROP gadgets in software compiled with and without -fzero-call-user-regs.

Downloading my experiment stuff:
1. rp++: `git clone https://github.com/0vercl0k/rp`
2. new gcc: `git clone --depth 1 --branch releases/gcc-11.2.0 https://github.com/gcc-mirror/gcc`
3. checksec: `apt-get install checksec`
4. ROPchains: `pip install ropchains` Another ROP search tool that also builds chains

Checksec provide a list of security features that the compiler have used to protect this binary from memory corruption vulnerabilitites.
RP++ searches a binary for patterns that can be used in a ROP chain.
GCC version 11.2.0 is a version of gcc that has support for -fzero-call-used-regs.

Building GCC 11.2.0 with GCC 9.3.0 for 64bit
```
./configure --disable-multilib
make
```
Compiling GCC may take some time, while that is running we can make a comparison with a gcc that is not compiled with any ROP protections.

If you are reading this in late 2022 or later the "-fzero-call-used-regs" feature will already be in all recently stable distirbutions. If you type gcc --version check to see if your compiler is above version.

If you already have a gcc -fzero-call-used-regs option try this:
```
cd bash
./configure CFLAG=-fzero-call-used-regs=skip && make
```
or
if you dont have a gcc with -fzero-call-used-regs just build gcc without that option:
```
cd bash
./configure && make
```

Compile GCC and install it in my your home folder using
```
mkdir ~/gcc
make DESTDIR=~/gcc install
```
Then use that gcc version to compile a target
```
./configure CC=~/gcc/usr/local/bin/gcc CFLAGS=-fzero-call-used-regs=used-arg
```


```
cd bash
./configure CFLAGS=-fzero-call-used-regs=used-arg && make
cp ./bash ./bash-wiped-regs
./rp-lin-x64 -f bash-wiped-regs --unique --rop=8 | head

```

I did not expect the results I got instead I compiled a small program with only one function

```
int test(int r) {
   return r;
}
int main() {
   return test(0);
}
```

In gdb I run disas test to check how test is compiled in to asembler:
A comparison on compiler explorer: https://godbolt.org/z/Tden7oKq7
In compiler explorer we can compare the output assembler from diferent gcc versions and their respective options.
Dump of assembler code for function test compiled with used-arg:
```
   0x0000000000401106 <+0>:	push   %rbp
   0x0000000000401107 <+1>:	mov    %rsp,%rbp
   0x000000000040110a <+4>:	mov    %edi,-0x4(%rbp)
   0x000000000040110d <+7>:	mov    -0x4(%rbp),%eax
   0x0000000000401110 <+10>:	pop    %rbp
   0x0000000000401111 <+11>:	xor    %edi,%edi
   0x0000000000401113 <+13>:	retq  
```

Without protection it looks like this:
```
Dump of assembler code for function test:
   0x0000000000401106 <+0>:	push   %rbp
   0x0000000000401107 <+1>:	mov    %rsp,%rbp
   0x000000000040110a <+4>:	mov    %edi,-0x4(%rbp)
   0x000000000040110d <+7>:	mov    -0x4(%rbp),%eax
   0x0000000000401110 <+10>:	pop    %rbp
   0x0000000000401111 <+11>:	retq   
End of assembler dump.
```

We can see that the compiler is doing xor reg, reg beore ret and thereby ruiining the registers for anyone calling that gadget.

This makes things significantly more difficult for anyone who is building a rop attack. But a few problems remain.

### Callee-saved-register / Non-volatile 
Callee-saved registers (AKA non-volatile registers, or call-preserved) are used to hold long-lived values that should be preserved across calls.

When the caller makes a procedure call, it can expect that those registers will hold the same value after the callee returns, making it the responsibility of the callee to save them and restore them before returning to the caller. Or to not touch them.

x64 ABI define thse args as non-volatile: r12, r13, r14, r15, rbx, rsp and rbp

#### Caller saved registers
volatile registers, or call-clobbered registers. , it is the caller's responsibility to push these registers onto the stack or copy them somewhere else if it wants to restore this value after a procedure call. It's normal to let a call destroy temporary values in these registers.

### Jump Oriented Programming

https://www.comp.nus.edu.sg/~liangzk/papers/asiaccs11.pdf
```
The x86 stack is managed by two dedicated CPU registers: the esp “stack pointer” register, which points to the top of the stack, and the ebp “base
pointer” register, which points to the bottom of the current stack frame. Because the stack grows downward, i.e., grows in the direction of decreasing addresses, esp ≤ ebp. Each
stack frame stores each function call’s parameters, return address, previous stack frame pointer, and automatic (local) variables, if any. The stack content or pointers can be
manipulated directly via the two stack registers, or implicitly through a variety of CPU opcodes, such as push and
pop. The instruction set includes opcodes for making function calls (call) and returning from them (ret). The call
instruction pushes the address of the next instruction (the return address) onto the stack. Conversely, the ret instruction pops the stack into eip, resuming execution directly after the call.
An attacker can exploit a buffer overflow vulnerability or other flaw to overwrite part of the stack, such as replacing the current frame’s return address with a supplied value. In
the traditional return-into-libc approach, this new value is a pointer to a function in libc chosen by the attacker. After the victim program uses the new value and enters the function, the memory cells next to the overwritten return address are interpreted as parameters by the function, allowing the execution of an arbitrary function with attacker-specified
parameters. By chaining these malicious stack frames together, a sequence of functions can be executed.
```

### Missaligned offsets


### Other circumvention
om du har en enda pop edx-gadget så kan du ju styra edx,ecx,esi,edi,r8,r9,10 och r11 genom att anropa dem med minskande offset


### The Linux kernel target

The option has been added to the Linux kernel version 5.15 under  `> Security options > Kernel hardening options > Memory initialization`

```
  │ CONFIG_ZERO_CALL_USED_REGS:                                                                                                                                                                        │  
  │                                                                                                                                                                                                    │  
  │ At the end of functions, always zero any caller-used register                                                                                                                                      │  
  │ contents. This helps ensure that temporary values are not                                                                                                                                          │  
  │ leaked beyond the function boundary. This means that register                                                                                                                                      │  
  │ contents are less likely to be available for side channels                                                                                                                                         │  
  │ and information exposures. Additionally, this helps reduce the                                                                                                                                     │  
  │ number of useful ROP gadgets by about 20% (and removes compiler                                                                                                                                    │  
  │ generated "write-what-where" gadgets) in the resulting kernel                                                                                                                                      │  
  │ image. This has a less than 1% performance impact on most                                                                                                                                          │  
  │ workloads. Image size growth depends on architecture, and should                                                                                                                                   │  
  │ be evaluated for suitability. For example, x86_64 grows by less                                                                                                                                    │  
  │ than 1%, and arm64 grows by about 5%.                                                                                                                                                              │  
  │                                                                                                                                                                                                    │  
  │ Symbol: ZERO_CALL_USED_REGS [=y]                                                                                                                                                                   │  
  │ Type  : bool                                                                                                                                                                                       │  
  │ Defined at security/Kconfig.hardening:235                                                                                                                                                          │  
  │   Prompt: Enable register zeroing on function exit                                                                                                                                                 │  
  │   Depends on: CC_HAS_ZERO_CALL_USED_REGS [=y]                                                                                                                                                      │  
  │   Location:                                                                                                                                                                                        │  
  │     -> Security options                                                                                                                                                                            │  
  │       -> Kernel hardening options                                                                                                                                                                  │  
  │         -> Memory initialization 
```
This kernel option corresponds to adding the gcc argument -fzero-call-used-regs=used-gpr.


Ok this pretty much sums up what I was going to investigate but for the sake of it, lets compare rop gadgets!

Next lets compile a linux kernel to see if we can reduce attack surface there:
```
root@32acc84fe1d0:/linux-5.6.9# make KCFLAGS="-fzero-call-used-regs=used-arg" -j4

Trying to open 'vmlinux-used-args'..
Loading ELF information..
FileFormat: Elf, Arch: Ia64
Using the Nasm syntax..

Wait a few seconds, rp++ is looking for gadgets..
in LOAD
1001509 found.

in LOAD
39205 found.

A total of 1040714 gadgets found.
You decided to keep only the unique ones, 236588 unique gadgets found.
```

Without protection:
```
Trying to open 'vmlinux-skip'..
Loading ELF information..
FileFormat: Elf, Arch: Ia64
Using the Nasm syntax..

Wait a few seconds, rp++ is looking for gadgets..
in LOAD
815024 found.

in LOAD
31006 found.

A total of 846030 gadgets found.
You decided to keep only the unique ones, 304366 unique gadgets found.
```

Here we can see an improvement!

If we go down the paranoid route and use all-gpr (all general purpose regiters)
```
Trying to open 'vmlinux-all-gpr'..
Loading ELF information..
FileFormat: Elf, Arch: Ia64
Using the Nasm syntax..

Wait a few seconds, rp++ is looking for gadgets..
in LOAD
1219506 found.

in LOAD
49995 found.

A total of 1269501 gadgets found.
You decided to keep only the unique ones, 115263 unique gadgets found.
```

Based on the findings made by the tool ROPgadget the vast majority of the remaining gadgets are JOP

```
user@lab:~/devel/rop$ ROPgadget --nojop --ropchain --binary vmlinux-used-args > vmlinux-used-args-ROPgadget_nojop
user@lab:~/devel/rop$ wc -l vmlinux-used-args-ROPgadget_nojop 
118539 vmlinux-used-args-ROPgadget_nojop
user@lab:~/devel/rop$ wc -l vmlinux-used-args-ROPgadget
709659 vmlinux-used-args-ROPgadget
user@lab:~/devel/rop$ bc
bc 1.07.1
Copyright 1991-1994, 1997, 1998, 2000, 2004, 2006, 2008, 2012-2017 Free Software Foundation, Inc.
This is free software with ABSOLUTELY NO WARRANTY.
For details type `warranty'. 
709659 - 118539
591120
```

We have not adressed the issue of ROP attacks but Jump Oriented Programming remains.
```
user@lab:~/devel/rop$ wc -l vmlinux-5.15-zero-regs-rp++-rop
249527 vmlinux-5.15-zero-regs-rp++-rop
user@lab:~/devel/rop$ wc -l vmlinux-5.15-skip-rp++-rop
326214 vmlinux-5.15-skip-rp++-rop
bc 1.07.1
Copyright 1991-1994, 1997, 1998, 2000, 2004, 2006, 2008, 2012-2017 Free Software Foundation, Inc.
This is free software with ABSOLUTELY NO WARRANTY.
For details type `warranty'. 
326214 - 249527
76687
```

Lets take peak in to the future of major distirbutions. First, lets look at the mental part of debian experimental https://packages.debian.org/experimental/ia64/linux-kbuild-5.15
The bubild depends on gcc11. We can find the package here: http://ftp.de.debian.org/debian-ports//pool-ia64/main/l/linux/linux-kbuild-5.15_5.15-1~exp1_ia64.deb

http://ftp.de.debian.org/debian-ports//pool-ia64/main/l/linux/linux-config-5.15_5.15-1~exp1_ia64.deb

We are interested in what going on here: https://packages.debian.org/experimental/kernel/linux-config-5.15

Today in early November 2021 there has been no discussions on https://lists.debian.org/debian-kernel/2021/11/threads.html to enable the Zero-used-regs feature.

I decided to post a whishlist bugreport:

```
Subject: linux: Please enable ZERO_CALL_USED_REGS to reduce ROP probability
Source: linux
Version: 5.15-1~exp1
Severity: wishlist

Hi, the option ZERO_CALL_USED_REGS will improve kernel security by
reducing the amount of available ROP gadgets by 20% on average in
the Linux kernel. Currently the option is not enabled in Debians
experimental kernel config. Please enable it if you consider build
size to be reasonable on all architectures.

The option requires building with GCC11 or a compiler that support
-fzero-call-user-regs.

Here is a comparison between the amount of unique ROP gadgets found
compared between a kernel build without CALL_USED_REGS in two
different ROP gadget scanning tools.

rp++ is a popular ROP scanning tool due to its ability to find many
different gadgets.

$ wc -l vmlinux-5.15-zero-regs-rp++-rop
249527 vmlinux-5.15-zero-regs-rp++-rop

$ wc -l vmlinux-5.15-skip-rp++-rop
326214 vmlinux-5.15-skip-rp++-rop

The tool ROPgadget is popular due to its ability to automatically
build ROP chains for a statically linked target.

vmlinux-5.15-zero-regs:
Unique gadgets found: 136014
No automatic chain building possible.

vmlinux-5.15-skip:
Unique gadgets found: 214104
Automatich chain building of gadgets possible.
```
## Acknowledgements
I want to thank:

* Linus Walleij, for answering questions about the Linux kernel community
* Kees Cook, for adding `CONFIG_ZERO_CALL_USED_REGS` to the Linux kernel
* Qing Zhao, engineer from oracle who wrote the zero--call-user-regs patch
* Alve Björk, errata base pointer, instruction pointer correction
* Odd Stranne, errata base pointer, DEP in Windows 2004, SafeSEH and CFG
* Laban Sköllermark, grammar and spelling
* Dick Svensson, feedback
* Andreas Kling, kernel experience and feedback from Serenity OS
