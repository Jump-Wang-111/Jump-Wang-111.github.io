---
layout: post
title: "Bootsector：利用bios例程打印hex"
date: 2023-11-15
tags: [Bootsector, os]
comments: true
toc: true
author: JumpWang
---

​		有些时候，仅仅打印字符串不足以帮助我们debug，毕竟除了内存里除了可打印字符还有许多不可打印的字符，因此，打印hex时十分重要的

### 很简单的逻辑

- 设置一个打印字符表，包括`0-9A-F`
- 遍历要打印的每个字节
- 分别取出其中的高4位和低4位的值
- 用这个值加上打印字符表的基址，就可以获得要打印的字符
- 每两个字符打印一个空格

### 很难看的实现

​		打印hello world的hex

```assembly
[org 0x7c00]
;test code to print hex
start:
	mov bp, 0x8000
	mov sp, 0x8000
	mov bx, hello
	mov ax, 20
	call print_hex
	jmp $

; param
; bx: address
; ax: num of bytes
print_hex:
	xor cx, cx
L3:	push ax
	push bx
	mov ah, 0x0e
	
	mov bl, [bx]	; only bx can be used as index reg
	xor bh, bh
	mov dl, bl
	shr bl, 4
	add bx, hexTable
	mov al, [bx]
	int 0x10

	xor bx, bx
	mov bl, dl
	and bl, 0xf
	add bx, hexTable
	mov al, [bx]
	int 0x10

	mov al, 0x20
	int 0x10

	pop bx
	pop ax
	inc cx
	inc bx
	cmp cx, ax
	jne L3
	ret
	
hexTable db '0123456789ABCDEF'
hello db 'hello world!', 0ah, 0		  ; str to print
times 510 - ($ - $$) db 0			 ; filled with 0x00
dw 0xaa55							; magic num of bootsecto

```

### 很欣慰的运行

![image-20231116194445306](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/202311191715300.png)
