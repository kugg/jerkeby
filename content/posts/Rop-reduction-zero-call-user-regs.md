---
title: "Gadget reduction using zero-call-user-regs"
date: "2021-11-26"
draft: false
---
## Introduction
In my previous article ["History of the ROP"](https://www.jerkeby.se/newsletter/posts/history-of-rop/), I cover the fundamental mitigation techniques and methods to circumvent them. In this article I'll introduce you to yet another security feature in GCC and where it's effective. This is a new feature and I think it's important to investigate its potential.

## Understanding the risks
When I find memory corruption vulnerabilities during pentests, clients and vendors have a tendency to dispute the impact of the vulnerabilities. The reason why the impact of memory corruption vulnerabilities are disputed is that it's very difficult to evaluate if the various mitigations are effective.

Determining the exploitability of a memory corruption bug is called triaging. In this process the researcher determines how the vulnerability can be exploited and to what degree. Modern mitigations aim to reduce the probability of a successful exploit.
Back in F-Secure I made a YouTube live stream with a colleague covering the process of triaging memory corruption vulnerabilities, check out the recording [here](https://www.youtube.com/watch?v=SiFWF1i5Quc).

## Current events
Last week my old colleague [Maxploit published a new remote heap buffer overflow](https://www.sentinelone.com/labs/tipc-remote-linux-kernel-heap-overflow-allows-arbitrary-code-execution/) that affects the 5.15 kernel in the TIPC protocol. The vulnerability can lead to kernel Remote Code Execution (RCE) if the attacker has access to "appropriate ROP gadgets".

Edit: [The vulnerability can be exploited with only one single ROP gadget](https://haxx.in/posts/pwning-tipc/).

## The ROP gadget

In 2007 a researcher named Hovav Shacham published a paper with the title ["The Geometry of Innocent Flesh on the Bone"](https://hovav.net/ucsd/dist/geometry.pdf) that attempts to refine the return-to-libc attack by defining an algorithm for finding "gadgets" in legitimate code.

The paper is the first to coin the phrase ROP attack but it also does what had been described by Tim Newsham as "a matter of simple coding" ten years earlier in the Bugtraq mailing list.

The technique was not novel, but the implementation was. Before we move in to "gadget-land", let it sink in. The idea of how to do automated ROP searches took ten years to be implemented. This is a reminder that our future in ten years may be implementations of our greatest ideas of today.

Back again to the gadgets. The compiler of a C program is a machine code generator that create machine code according to the format appropriate for the target CPU architecture and file format of the operating system environment.

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

## Background on register zeroing
In 2018 Graham Markall suggested the idea of doing stack and registry erasure prior to exit during [GNU Tools Cauldron 2018](https://gcc.gnu.org/wiki/cauldron2018#secure)
The suggestion is connected to a few challenges: `Stack and register erasure. Ensuring that on return from a function, no data is left lying on the stack or in registers. Particular challenges are in dealing with inlining, shrink wrapping and caching.`

The same year, Zelin Rong, Peidai Xie, Jingyuan Wang, Shenglin Xu, Yongjun Wang published a paper called [Clean the Scratch Registers:A Way to Mitigate
Return-Oriented Programming Attacks](https://dustri.org/b/files/papers/7c3954f7d19392056f072837b3a3775e_rop_reg_clean.pdf). The paper shows results from an experiment where [Intel PIN](https://www.intel.com/content/www/us/en/developer/articles/tool/pin-a-dynamic-binary-instrumentation-tool.html) is used in Justin In Time mode to add scratch register zeroing to the end of every function. This instrumentation method is used on known vulnerable CTF (Capture The Flag) challenges where we know that the target is exploitable using ROP. In the five experiments presented in in the paper all of them became unsolvable using ROP after using "register zeroing".

## Register zeroing
An assembler function typically stores its temporary variables in "registers". When the function is finished it runs the instruction "ret" (return). A good ROP gadget does something to a register and ends with ret, just like a function.

Here is a function that returns its only argument written in C.
```
int test(int r) {
   return r;
}
```
Here is the assembler that the compiler produced:
```
test(int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     eax, DWORD PTR [rbp-4]
        pop     rbp
        ret
```
The `edi` register is used in this function to store the return value.
Zeroing the `edi` register means that its contents is set to zero before returning from the function. The quickest way to do this is using the instruction `xor edi, edi`.

This is what the same function looks like after zeroing `edi`
```
test(int):
        push    rbp
        mov     rbp, rsp
        mov     DWORD PTR [rbp-4], edi
        mov     eax, DWORD PTR [rbp-4]
        pop     rbp
        xor     edi, edi
        ret
```

As a result this function is now less usable as a ROP gadget because the value of the general purpose register `edi` is Zeroed before return.

## Experiment with GCC zero-call-user-regs
In 2018 Victor Rodriguez was quick to add a patch to the [CLEAR Linux project GCC branch](https://github.com/clearlinux-pkgs/gcc/blob/master/0001-x86-Add-mzero-caller.patch) following up on the idea of zeroing registers. The patch is so far only used within the CLEAR Linux projects internal GCC branch. But it is a reference that helps the GCC group move forward on accepting an upstream patch in 2020.

In the upcoming release of GCC 11 there is a feature written by (Qing Zhao from Oracle) that does register zeroing prior to returning. One use-case for the feature can be to reduce the probability of usable ROP gadgets just as suggested in the "Clean the Scratch registers article". The feature also impacts the "speculative execution" bug class, but let's talk about that elsewhere.
Qing Zhao writes a rationale for the overall design on the [GCC mailing list](https://gcc.gnu.org/pipermail/gcc-patches/2020-August/552262.html). He motivates why zeroing is better than randomization and which registers to erase. The concept is motivated by mitigating ROP and reducing information leakage from registers on exit.

I want to determine to what extent ROP is mitigated by the `-fzero-call-user-regs` option in GCC.

#### Callee-saved-register / Non-volatile
Callee-saved registers (AKA non-volatile registers, or call-preserved) are used to hold long-lived values that should be preserved across calls.

When the caller makes a procedure call, it can expect that those registers will hold the same value after the callee returns, making it the responsibility of the callee to save them and restore them before returning to the caller. Or to not touch them.

x64 ABI define these args as non-volatile: `r12, r13, r14, r15, rbx, rsp and rbp`.

#### Caller saved registers and shrink wrapping
Caller saved, volatile registers are also called call-clobbered registers. It is the caller's responsibility to push these registers onto the stack or copy them somewhere else if it wants to restore this value after a procedure call. It's normal to let a call destroy temporary values in these registers.

As Graham Markall feared back in 2018 there might also be problems with ["shrink wrapping"](https://gcc.gnu.org/onlinedocs/gcc-7.1.0/gccint/Shrink-wrapping-separate-components.html) when zeroing "all registers". This is [addressed by making all zeroing based on only call-used registers](https://gcc.gnu.org/pipermail/gcc-patches/2020-August/551448.html).

### Definition of call-used
We now know, that this patch only zeroes `call-used` registers. Let's find out what the definition of a `call-used` register is.
```
    A "call-used" register is a register whose contents can be changed
     by a function call; therefore, a caller cannot assume that the
     register has the same contents on return from the function as it
     had before calling the function.  Such registers are also called
     "call-clobbered", "caller-saved", or "volatile".
```
So `call used` does not mean that the register is used in the call, it means that the register "can" be used in the call.
The term `used` however means that the register is actually used in the function.

### All the options
Let's unfold all the other feature options.
The [reasoning from Qing on the GCC mailinglist](https://www.mail-archive.com/gcc-patches@gcc.gnu.org/msg247451.html) for having an extended feature option list is this:


```
     In order to satisfy users with different security needs and control
     the run-time overhead at the same time, GCC provides a flexible way
     to choose the subset of the call-used registers to be zeroed.

     The three basic values of CHOICE are:

        * 'skip' doesn't zero any call-used registers.

        * 'used' only zeros call-used registers that are used in the
          function.  A "used" register is one whose content has been set
          or referenced in the function.

        * 'all' zeros all call-used registers.

     In addition to these three basic choices, it is possible to modify
     'used' or 'all' as follows:

        * Adding '-gpr' restricts the zeroing to general-purpose
          registers.

        * Adding '-arg' restricts the zeroing to registers that can
          sometimes be used to pass function arguments.  This includes
          all argument registers defined by the platform's calling
          conversion, regardless of whether the function uses those
          registers for function arguments or not.

     The modifiers can be used individually or together.  If they are
     used together, they must appear in the order above.

     The full list of CHOICEs is therefore:

        * 'skip' doesn't zero any call-used register.

        * 'used' only zeros call-used registers that are used in the
          function.

        * 'all' zeros all call-used registers.

        * 'used-arg' only zeros used call-used registers that pass
          arguments.

        * 'used-gpr' only zeros used call-used general purpose
          registers.

        * 'used-gpr-arg' only zeros used call-used general purpose
          registers that pass arguments.

        * 'all-gpr-arg' zeros all call-used general purpose registers
          that pass arguments.

        * 'all-arg' zeros all call-used registers that pass arguments.

        * 'all-gpr' zeros all call-used general purpose registers.

     Among this list, 'used-gpr-arg', 'used-arg', 'all-gpr-arg', and
     'all-arg' are mainly used for ROP mitigation.
```

The feature options that have `used` in the name does something clever. After initial compilation the option reads every function backwards from return and looks where a register has ended upon the right hand side of an instruction (Example: `mov 0, eax` have [clobbered](http://zvon.org/comp/r/ref-Jargon_file.html#Terms~clobber) eax so it will need zeroing.). It lists all the "used registers" and adds them in the end for zeroing. "GPR" stands for general purpose registers and by choosing to only zero them the risk of causing damage to the program is reduced.

From `gcc/function.c`:
```
/*
 * If only_gpr is true, only zero call-used registers that are
 * general-purpose registers; if only_used is true, only zero
 * call-used registers that are used in the current function;
 * if only_arg is true, only zero call-used registers that pass
 * parameters defined by the flatform's calling conversion.
 */
```

The options called "all" does not distinguish if the register is used in the function. This argument could turn out to be interesting for experimentation as all-gpr can reduce information leakage. For now this option is unlikely to be used.
To be clear, even if you compile you program with `all` the non-volatile registers will not be touched by this feature.

That is: `r12, r13, r14, r15, rbx, rsp and rbp`.


#### Linux kernel v5.15

While I was writing this article the Linux kernel released version 5.15 which include a [configure option](https://github.com/KSPP/linux/issues/84) for using `-fzero-call-user-regs`. The Linux kernel community under supervision of Kees Cook decided to use the `used-gpr` option.

`> Security options > Kernel hardening options > Memory initialization`

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
This kernel option corresponds to adding the `gcc` argument `-fzero-call-used-regs=used-gpr`.

#### Other operating systems

The SerenityOS project have been using the zero-call-used-regs=used-gpr compiler option since July this year [without reporting any negative impact](https://github.com/SerenityOS/serenity/pull/8950).

### The comparison

I decided to use a ROP gadget search tool to compare a binary with and without this feature.
The concept behind the mitigation is to clear all used registers prior to calling return in a given function.
In the comparison I want to focus on pure ROP gadgets as much as possible and exclude counting of [JOP and syscall gadgets]((https://www.comp.nus.edu.sg/~liangzk/papers/asiaccs11.pdf).

Essentially this is a comparison of the quality of ROP gadgets in software compiled with and without `-fzero-call-user-regs`.

Downloading my experiment stuff in case you want to tag along:
1. rp++: `git clone https://github.com/0vercl0k/rp`
2. new GCC: `git clone --depth 1 --branch releases/gcc-11.2.0 https://github.com/gcc-mirror/gcc`
3. ROPchains: `pip install ropchains` Another ROP search tool that also builds chains

RP++ searches a binary for patterns that can be used in a ROP chain.
GCC version 11.2.0 is a version of GCC that has support for `-fzero-call-used-regs`.

If you are reading this in late 2022 or later the `-fzero-call-used-regs` feature will already be in all recently stable distributions. Use `gcc --version` check to see if your compiler is above version.

Compile GCC 11.2 and install it in the home dir like this:
```
./configure --disable-multilib
make
mkdir ~/gcc
make DESTDIR=~/gcc install
```

Once you have GCC 11, use a line like this one to compile your target:
```
./configure CC=~/gcc/usr/local/bin/gcc CFLAGS=-fzero-call-used-regs=used-gpr
```

I compiled a small program with only two functions
```
int test(int r) {
   return r;
}
int main() {
   return test(0);
}
```

In gdb I run disas test to check how test is compiled into assembler:
A comparison on compiler explorer: https://godbolt.org/z/Tden7oKq7

In compiler explorer we can compare the output assembler from different GCC versions and their respective options.
Dump of assembler code for function test compiled with used-gpr:
```
   0x0000000000401106 <+0>:	push   %rbp
   0x0000000000401107 <+1>:	mov    %rsp,%rbp
   0x000000000040110a <+4>:	mov    %edi,-0x4(%rbp)
   0x000000000040110d <+7>:	mov    -0x4(%rbp),%eax
   0x0000000000401110 <+10>:	pop    %rbp
   0x0000000000401111 <+11>:	xor    %edi,%edi
   0x0000000000401113 <+13>:	retq
```

Without protection (skip) it looks like this:
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

We can see that the compiler is doing `xor reg, reg` before ret and thereby ruining the registers for anyone calling that gadget.

This makes things significantly more difficult for anyone who is building a ROP attack. But a few problems remain.

### Experiment setup
In this experiment we are deliberately excluding results that are JOP or syscalls.

### The Busybox target

Busybox was originally a shell used in embedded devices. Nowadays a full environment containing everything you may want for debugging a IoT device. Busybox is compiled as a static by default which is exactly the type of target where ROP attacks are useful.
Like the Linux kernel the configure stage of the compilation is done using `make menuconfig` inside the compilation options menu in `Settings` you can configure the `CFLAGS`. Initially I'll compile using `-fzero-call-used-regs=skip` it will simply compile without any ROP protections.

No protections:
```
./rp-lin-x64 -f ./busybox_skip --unique --rop=8
A total of 40624 gadgets found.
You decided to keep only the unique ones, 18050 unique gadgets found.
```

Used-gpr:
```
A total of 49648 gadgets found.
You decided to keep only the unique ones, 15262 unique gadgets found.
```
Interesting! The total amount of gadgets increase on the Busybox target, but the amount of unique gadgets decrease somewhat. Remember that for exploitation, we are interested in unique gadgets. We will address the gadget increase later in the Kernel section.

The ROPgadget tool finds the following results.

All gadgets:
```
ROPgadget --ropchain --binary busybox_skip
Unique gadgets found: 57972
```

Only ROP gadgets:
```
ROPgadget --nosys --nojop --ropchain --binary busybox_skip
Unique gadgets found: 11214
```

On the used-gpr compiled version.
All gadgets:
```
ROPgadget --ropchain --binary busybox_used-gpr
Unique gadgets found: 54225
```

Only ROP gadgets:
```
ROPgadget --nosys --nojop --ropchain --binary busybox_used-gpr
Unique gadgets found: 8254
```

This means that a vast majority of the gadgets found after used-gpr are JOP gadgets (and a few syscall gadgets).
The ROP quantity is reduced by 26.39%.

### The Linux kernel target

Let's compile a Linux kernel to see if we can reduce attack surface there.

#### ROPgadget
In this section I have compiled the Linux kernel using the Kees Cook option to zero regs found in 5.15:
Here is a reference run with the skip option (no protections).

All ROP gadgets on a skipped kernel:
```
ROPgadget --ropchain --binary vmlinux-5.15-skip
Unique gadgets found: 777954
```

Only ROP gadgets on a skipped kernel:
```
ROPgadget --nosys --nojop --ropchain --binary vmlinux-5.15-skip
Unique gadgets found: 213884
```
The number to keep in mind here is `213884` this is the amount of unique usable ROP gadgets.
What is noteworthy about this run is that ROPgadget is able to auto-generate a full ROP chain. That is about to change.

In this section we compile the kernel using the `-fzero-call-used-regs=used-gpr`

All ROP gadgets on a used-gpr kernel:
```
ROPgadget --ropchain --binary vmlinux-5.15-zero-regs
Unique gadgets found: 776216
```
The quantity of ROP gadgets found using ROPgadget has gone down by a `1738`.

Only ROP gadgets on a used-gpr kernel:
```
ROPgadget --nosys --nojop --ropchain --binary vmlinux-5.15-zero-regs
Unique gadgets found: 135814
```

In reality the amount of ROP gadgets reduced are `78070` that is a unique ROP gadget reduction of `36,5%`.
The automatic ROP chain generation fails on the early stages, because there are no good GPR write gadgets.

So why is there an increase in JOP gadgets when the `used-gpr` protection is added?
The used-gpr kernel had `640402` JOP and SYS gadgets, while the skip kernel had `564070`, this is an increase in `13%`.
When the Linux kernel informative text tells you that this option reduces ROP gadgets by 20%, it lowers the ROP gadgets but increase the JOP gadgets.

I think the explanation is "misaligned offset instructions". Remember how the Sacham paper quoted earlier explained that the word `dress` is a part of `address`?
Introducing more compiled code also increase the candidates for misaligned instructions.

### Gadget quality

The ROPgadget search tool works in 5 steps these are the first four:

1. Write-what-where gadgets

In this step the search tool will try to find a list of gadgets to populate the registers for a future syscall.
This is memory segments that POP GPR registers some data from the stack and then does return.

Here is an example of such gadget: `pop rax ; ret`
This gadget reside on an address that the search tool tries to find and write to the ROP chain.
The registers should point to the address of the arguments later passed to a syscall. Typically we are looking for the address of a string like "/bin/sh".

2. Init syscall number gadgets

Identify a group of gadgets to control the
Make the `eax/rax` register contain the number of the syscall instruction (0x80) or set it to a controlled value by finding a `xor eax, eax` / `xor rax,rax`.
After using `used-gpr` there are thousands of gadgets for this purpose available in the kernel.

3. Init syscall arguments gadgets

The searchtool will try to find gadgets to configurae the arguments to a [syscall](https://filippo.io/linux-syscall-table/). Typically the desired syscall is `execve` which syscall number `59`.

4. Syscall gadget

This is done by calling a gadget that increments `eax` or `rax` by one and run it 80 times.
The tool will search for a gadget that can be used to trigger a syscall.

5. Assemble the ROP chain

When the search tool has assembled all the gadgets the addresses of the gadgets are stacked in the payload.

The concept of `-fzero-call-used-regs=used-gpr` is that it effectively removes all gadgets usable to perform `Step 1.`. Every time a GPR registry is popped, it is always zeroed using the xor technique prior to `ret`.

### Debian
Let's take a peak into the future of major distributions. First, let's look at Debian experimental https://packages.debian.org/experimental/ia64/linux-kbuild-5.15

The build depends on gcc11, a good start! We can find the package here: http://ftp.de.debian.org/debian-ports//pool-ia64/main/l/linux/linux-kbuild-5.15_5.15-1~exp1_ia64.deb

It's likely that GCC 11.2 will be the candidate for the next stable Debian release.

In [this file we can find the current experimental Kernel configuration.](http://ftp.de.debian.org/debian-ports//pool-ia64/main/l/linux/linux-config-5.15_5.15-1~exp1_ia64.deb)

In early November 2021 there has been no discussions on https://lists.debian.org/debian-kernel/2021/11/threads.html to enable the Zero-used-regs feature.

I decided to post a [whishlist bugreport](https://lists.debian.org/debian-kernel/2021/11/msg00057.html) to improve kernel security in the Debian derivatives.


## Conclusions
1. The gcc `-fzero-call-used-regs` is most commonly used with the feature option `used-gpr`, it was recently added  as a non default configura option to the Linux Kernel.
2. The option will identify volatile registers that has been touched by a function and zero them prior to return.
3. Adding the option introduce new JOP gadgets by 13% and reduce the ROP gadgets by 36% in the Linux kernel.
4. Commonly used ROP chain generators fail to produce chains automatically when the feature is enabled.
5. Circumventions to the method exist using misaligned offsets that still produce usable gadgets.
6. The gadgets addressed are registry altering gadgets.

## Acknowledgements
I want to thank:

* Linus Walleij, for answering questions about the Linux kernel community
* Kees Cook, for adding `CONFIG_ZERO_CALL_USED_REGS` to the Linux kernel
* Qing Zhao, engineer from Oracle who wrote the zero--call-user-regs patch
* Alve Björk, errata base pointer, instruction pointer correction
* Odd Stranne, errata base pointer, DEP in Windows 2004, SafeSEH and CFG
* Laban Sköllermark, grammar and spelling
* Dick Svensson, feedback
* Andreas Kling, kernel experience and feedback from SerenityOS
* Calle Svensson, technical guidance and understanding of circumvention methods
* Andreas from Romab gave some comforting words that cheered me up when I was sick

## Future research
Adding support for "unsigned overflow protection" in GCC would reduce the risk of overwriting the GOT table (and circumventing RELRO) using global variables.
Zero init instead of stack erasure. https://gcc.gnu.org/pipermail/gcc-patches/2020-August/551444.html

## Caveats
This is a work in progress section to be filled in with circumventions and potential issues with `fzero-call-used-regs`. Hopefully these sections will qualify to be part of the main post.

### Jump Oriented Programming
We have now extensively addressed the ROP problem. However there is also a problem surrounding JOP, this is a problem that is not addressed by this patch. However JOP gadgets are rarely useful without the combination of a good ROP gadget.

Extract from a paper on the [JOP attack pattern](https://www.comp.nus.edu.sg/~liangzk/papers/asiaccs11.pdf)
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

### Misaligned offsets
The GCC11 zero-call-used-regs feateure didn't take misaligned offsets in to consideration.

### Added gadgets
Adding code to each function also adds gadgets, an attacker wants to have unique gadgets but these are pretty much all the same.

### Elastic objects to defeat KASLR
Bypassing kernel protections using elastic objects by Yueqi Chen, Zhenpeng Lin, Xinyu Xin [Video](https://www.youtube.com/watch?v=yXhH0IJAxkE) read the [paper](https://zplin.me/papers/ELOISE.pdf).

### Other circumvention
If you have a single pop edx gadget (or rdx) you can control `edx, ecx, esi, edi, r8, r9, r10 and r11` by calling the main register with a decreasing offset.
