---
title: "Introduction. Fibonacci compiled!"
tags: []
date: 2023-02-03T15:02:19+04:00
draft: true
---

Hello reader!

I’m happy to see you here, and I hope you’ll find something for yourself here.
This will be a series of posts about making a brand-new programming language and its compiler.
I’ve been doing that for quite a while, so a few milestones have passed.
But today, I want to share a new achievement.

Today I managed to compile a function call.
And not even that, but the function is recursive.

The test program is simple (in the repo)

```
package main

func main(a int) (r int) {
    return fib(a)
}

func fib(n int) (r int) {
    if n <= 1 {
        return n
    }

    return fib(n-1) + fib(n-2)
}
```

It looks like a Go code. It is not a coincidence.
I’m coding the compiler in Go, and I use its stdlib parser instead of my own.
My priority, for now, is the compiler backend (ir -> asm).
The middle part (ast -> ir) goes next.
And the frontend (text -> ast) is the last in the queue.

I’ve managed register allocation before. Which was a hard fight, but I’ll tell you about it next time.
The algorithm assigns a limited set of registers to subexpressions calculated in the function.

So now I needed to deal with the fact the function call is an unusual expression.
It requires fixed registers assigned to function arguments and their results.

<!--
Why? Because a function can potentially be called from anywhere.
And in any case, the callee function expects arguments in some specific location. Register X0, for example.
So the caller must put the function arguments to these locations and take function results from somewhere too.
-->

After a few failed ideas I decided to reserve argument registers from assigning to any expression which is alive at the function call instruction.
Now I could put arguments and take results from the callee. But that wasn’t the end.

The compiled program didn’t succeed on any input more than 1. The problem was that I passed arguments, but the callee function (the same one) was using the same registers.
So we loose the intermediate results after called function returned.

I can't assign the registers and hope the same function called again will not corrupt them.
That is where the Stack comes in. Each function down the recursion call needs its own personal storage.

<!--
There are two approaches. The first is when the caller saves the needed registers before the call and restores them later.
The second is when the callee saves registers it will use at the beginning of the function and restores them before returning.
Both approaches have pros and cons, and they are both used simultaneously. Some registers are caller saved, and some are callee saved.
Everything I'm describing here is called the Calling Convention, which was invented long ago.
-->

I'm using the most straightforward approach I imagined I could implement. The first eight registers are dedicated to arguments and returning values.
And it wasn't obvious to me that I may not use them all during the function call since the callee function may call any other and corrupt the 5th argument register event if I passed only two arguments.
So I reserved all of them in register allocation.

The rest of the registers are callee saved because I can do that only once in the function prolog and restore in the epilogue.
After the register allocation step, before code generation, I find the highest used register and save all from the 8th to the highest to the stack.
A similar process is done in the epilogue but in reverse.

Now the function works and makes me so much happy.

This is the assembler text my compiler generates.

```
.global _fib
.align 4
_fib:
	STP     FP, LR, [SP, #-16]!
	MOV     FP, SP
	STP     X8, X9, [SP, #-16]!

	// reg 0  args 1  expr 7
	MOV	X1, #1	// expr 9
	CMP	X0, X1	// expr 10
	B.LE	L.3
	B	L.2.fix.3.1
L.3:
	B	L.1
L.2.fix.3.1:
	// permutate [[9 0]]
	MOV	X9, X0
	B	L.2
L.2:
	MOV	X0, #1		// expr 16
	SUB	X0, X9, X0	// expr 17
	BL	_fib		// func call  17 (X0) -> 18 (X8)
	MOV	X8, X0		// func res 0 fix
	MOV	X0, #2		// expr 19
	SUB	X0, X9, X0	// expr 20
	BL	_fib		// func call  20 (X0) -> 21 (X0)
	ADD	X0, X8, X0	// expr 22
	B	L.1
L.1:
	// reg 0  res 0  expr 24

	LDP     X8, X9, [SP], #16
	LDP     FP, LR, [SP], #16
	RET
```

Which I then compile by llvm assembler. And this is what the result looks like.

```
0000000100003f60 <_fib>:
100003f60: fd 7b bf a9 	stp	x29, x30, [sp, #-16]!
100003f64: fd 03 00 91 	mov	x29, sp
100003f68: e8 27 bf a9 	stp	x8, x9, [sp, #-16]!
100003f6c: 21 00 80 d2 	mov	x1, #1
100003f70: 1f 00 01 eb 	cmp	x0, x1
100003f74: 4d 00 00 54 	b.le	0x100003f7c <_fib+0x1c>
100003f78: 02 00 00 14 	b	0x100003f80 <_fib+0x20>
100003f7c: 0c 00 00 14 	b	0x100003fac <_fib+0x4c>
100003f80: e9 03 00 aa 	mov	x9, x0
100003f84: 01 00 00 14 	b	0x100003f88 <_fib+0x28>
100003f88: 20 00 80 d2 	mov	x0, #1
100003f8c: 20 01 00 cb 	sub	x0, x9, x0
100003f90: f4 ff ff 97 	bl	0x100003f60 <_fib>
100003f94: e8 03 00 aa 	mov	x8, x0
100003f98: 40 00 80 d2 	mov	x0, #2
100003f9c: 20 01 00 cb 	sub	x0, x9, x0
100003fa0: f0 ff ff 97 	bl	0x100003f60 <_fib>
100003fa4: 00 01 00 8b 	add	x0, x8, x0
100003fa8: 01 00 00 14 	b	0x100003fac <_fib+0x4c>
100003fac: e8 27 c1 a8 	ldp	x8, x9, [sp], #16
100003fb0: fd 7b c1 a8 	ldp	x29, x30, [sp], #16
100003fb4: c0 03 5f d6 	ret
```

It's funny; it's not much worse than the clang output of the same program written in C.

```
0000000100003f5c <_fib>:
100003f5c: f4 4f be a9 	stp	x20, x19, [sp, #-32]!
100003f60: fd 7b 01 a9 	stp	x29, x30, [sp, #16]
100003f64: fd 43 00 91 	add	x29, sp, #16
100003f68: 1f 08 00 71 	cmp	w0, #2
100003f6c: 6a 00 00 54 	b.ge	0x100003f78 <_fib+0x1c>
100003f70: 13 00 80 52 	mov	w19, #0
100003f74: 0b 00 00 14 	b	0x100003fa0 <_fib+0x44>
100003f78: 13 00 80 52 	mov	w19, #0
100003f7c: f4 03 00 aa 	mov	x20, x0
100003f80: 80 06 00 51 	sub	w0, w20, #1
100003f84: f6 ff ff 97 	bl	0x100003f5c <_fib>
100003f88: e8 03 00 aa 	mov	x8, x0
100003f8c: 80 0a 00 51 	sub	w0, w20, #2
100003f90: 13 01 13 0b 	add	w19, w8, w19
100003f94: 9f 0e 00 71 	cmp	w20, #3
100003f98: f4 03 00 aa 	mov	x20, x0
100003f9c: 2c ff ff 54 	b.gt	0x100003f80 <_fib+0x24>
100003fa0: 00 00 13 0b 	add	w0, w0, w19
100003fa4: fd 7b 41 a9 	ldp	x29, x30, [sp, #16]
100003fa8: f4 4f c2 a8 	ldp	x20, x19, [sp], #32
100003fac: c0 03 5f d6 	ret
```

Clang unwinded the recursive tail call, but it uses the same 32 bytes of the stack, and the code size is only one instruction shorter.

[commit](https://github.com/slowlang/slow/commit/ae712d5b363cdfd950cf4526e284f1973ecc48ed)
