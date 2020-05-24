---
layout: post
title: "Details Of Hello World in macOS Catalina"
categories: [blog, C, assembly, macOS, gotchas]
tags: []
---
# details_of_helloWorld
# In depth look at details of C hello world in macOS Catalina

Our basic program (ignore the size and nmemb nonsense, I was just playing around):
```C
#include <stdio.h>
int main() {
    fwrite("Hello, world!\n", 128, 15, stdout);
    return 0;
}
```
Using `gcc -S hello.c -o hello.s` will give us
```M
	.section	__TEXT,__text,regular,pure_instructions
	.build_version macos, 10, 15	sdk_version 10, 15, 4
	.globl	_main                   ## -- Begin function main
	.p2align	4, 0x90
_main:                                  ## @main
	.cfi_startproc
## %bb.0:
	pushq	%rbp
	.cfi_def_cfa_offset 16
	.cfi_offset %rbp, -16
	movq	%rsp, %rbp
	.cfi_def_cfa_register %rbp
	subq	$16, %rsp
	movq	___stdoutp@GOTPCREL(%rip), %rax
	movl	$0, -4(%rbp)
	movq	(%rax), %rcx
	leaq	L_.str(%rip), %rdi
	movl	$128, %esi
	movl	$15, %edx
	callq	_fwrite
	xorl	%r8d, %r8d
	movq	%rax, -16(%rbp)         ## 8-byte Spill
	movl	%r8d, %eax
	addq	$16, %rsp
	popq	%rbp
	retq
	.cfi_endproc
                                        ## -- End function
	.section	__TEXT,__cstring,cstring_literals
L_.str:                                 ## @.str
	.asciz	"Hello, world!\n"

.subsections_via_symbols
```

The first few lines are pretty basic. 
```
.section	__TEXT,__text,regular,pure_instructions
.build_version macos, 10, 15	sdk_version 10, 15, 4
.globl	_main                   ## -- Begin function main
.p2align	4, 0x90
```
`.section __TEXT,__text` is Apple's way of defining a `.text` 
[section](https://developer.apple.com/library/archive/documentation/DeveloperTools/Reference/Assembler/040-Assembler_Directives/asm_directives.html#//apple_ref/doc/uid/TP30000823-SW1).
>regular (S_REGULAR)
A regular section may contain any kind of data and gets no special processing from the link editor. This is the default section type. Examples of regular sections include program instructions or initialized data.

The next line is just some build information about OS version, etc.
`.build_version macos, 10, 15	sdk_version 10, 15, 4`.
The `.globl _main` declaration is used for making the `_main` symbol external so
the linker knows what to look for when marking the entry point. The align piece
simply explains where the next align-expression boundary is. In this case it's 
every 16-byte alignments with 0x90 as the filling between the current location
and the next alignment.

Finally we get onto the actual program.
```
pushq	%rbp
.cfi_def_cfa_offset 16
.cfi_offset %rbp, -16
movq	%rsp, %rbp
```
We begin by saving the frame pointer by pushing quadword `%rbp` onto the stack.
We then move the stack pointer `%rsp` into the `%rbp` register. The `q` in these
instructions means a quadword operation (64bit).

```
subq	$16, %rsp
movq	___stdoutp@GOTPCREL(%rip), %rax
movl	$0, -4(%rbp)
movq	(%rax), %rcx
leaq	L_.str(%rip), %rdi

```
We then pad the stack with 16 bytes. I'm assuming because we defined p2align as 2^4 = 16.
We then use the Global Offset Table to get stdout and place it into our result register.
Then we store the immediate value 0 at 4 bytes from our stack pointer. Then we 
place the stdout reference in the 4th argument register, `%rcx`. We then load up
our string into the first argument register (`%rdi`). 
```
movl	$128, %esi
movl	$15, %edx
callq	_fwrite
```
We push our other arguments of 128 and 15 into their respective registers, and call
our print function.
```
xorl	%r8d, %r8d
```
The XOR simply zeros out the `%r8d` register. 
```
movq	%rax, -16(%rbp)         ## 8-byte Spill
movl	%r8d, %eax
addq	$16, %rsp
popq	%rbp
retq
```
We store the result register at another 16byte offset. We then zero out the
`%eax` register; reset our stack pointer back to the beginning, pop off our
frame pointer and return. Our hello world program is now done.
