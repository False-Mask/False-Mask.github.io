---
title: 机器码编码解析
tags:
cover:
---



# 机器码编码解析





# Arm



## armv7a





## armv8a

https://developer.arm.com/documentation/ddi0406/latest/



0x39536008

ldrb w8, [x0, #0x4d8]

00111001 01010011 01100000 00001000



> case1

op0 - 0

01

op1- 1100 

1 01010011 01100000 00001000



> case2

op0 - 0011

1

op1 - 0

0

op2 - 1 01010011 011000

00 00001000



> case 3

size - 00

111

VR - 0

01 

opc - 01

imm12 - 010011 011000

Rn - 00 000

Rt - 01000





0100 11011000



## armv9a





## armv7a



ldrble r4



  5ac700: f9426000     	ldr	x0, [x0, #0x4c0]

  90e170: 39536008     	ldrb	w8, [x0, #0x4d8]



00111001 01010011 01100000 00001000









00000000005ac6e0 <_ZNK3art7Runtime20GetSystemClassLoaderEv>:
  5ac6e0: d100c3ff     	sub	sp, sp, #0x30
  5ac6e4: a9017bfd     	stp	x29, x30, [sp, #0x10]
  5ac6e8: a9024ff4     	stp	x20, x19, [sp, #0x20]
  5ac6ec: 910043fd     	add	x29, sp, #0x10
  5ac6f0: d53bd054     	mrs	x20, TPIDR_EL0
  5ac6f4: aa0003f3     	mov	x19, x0
  5ac6f8: f9401688     	ldr	x8, [x20, #0x28]
  5ac6fc: f90007e8     	str	x8, [sp, #0x8]
  5ac700: f9426000     	ldr	x0, [x0, #0x4c0]
  5ac704: b4000120     	cbz	x0, 0x5ac728 <_ZNK3art7Runtime20GetSystemClassLoaderEv+0x48>
  5ac708: f9401688     	ldr	x8, [x20, #0x28]
  5ac70c: f94007e9     	ldr	x9, [sp, #0x8]
  5ac710: eb09011f     	cmp	x8, x9
  5ac714: 54000501     	b.ne	0x5ac7b4 <_ZNK3art7Runtime20GetSystemClassLoaderEv+0xd4>
  5ac718: a9424ff4     	ldp	x20, x19, [sp, #0x20]
  5ac71c: a9417bfd     	ldp	x29, x30, [sp, #0x10]
  5ac720: 9100c3ff     	add	sp, sp, #0x30
  5ac724: d65f03c0     	ret
  5ac728: f9414268     	ldr	x8, [x19, #0x280]
  5ac72c: b50000a8     	cbnz	x8, 0x5ac740 <_ZNK3art7Runtime20GetSystemClassLoaderEv+0x60>
  5ac730: f9403268     	ldr	x8, [x19, #0x60]
  5ac734: b40000c8     	cbz	x8, 0x5ac74c <_ZNK3art7Runtime20GetSystemClassLoaderEv+0x6c>
  5ac738: aa1f03e0     	mov	x0, xzr
  5ac73c: 17fffff3     	b	0x5ac708 <_ZNK3art7Runtime20GetSystemClassLoaderEv+0x28>
  5ac740: f9400908     	ldr	x8, [x8, #0x10]
  5ac744: 39400108     	ldrb	w8, [x8]
  5ac748: 34ffff48     	cbz	w8, 0x5ac730 <_ZNK3art7Runtime20GetSystemClassLoaderEv+0x50>
  5ac74c: d0ffd561     	adrp	x1, 0x5a000
  5ac750: 910e9c21     	add	x1, x1, #0x3a7



000000000090e170 <_ZNK3art7Runtime22IsVerificationSoftFailEv>:
  90e170: 39536008     	ldrb	w8, [x0, #0x4d8]
  90e174: 7100091f     	cmp	w8, #0x2
  90e178: 1a9f17e0     	cset	w0, eq
  90e17c: d65f03c0     	ret





# X86









