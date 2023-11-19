---
layout: post
title: "Bootsector:利用bios例程打印string"
date: 2023-11-10
tags: [Bootsector, os]
comments: true
toc: true
author: JumpWang
---

这里以MBR启动扇区为例，写一个启动扇区（参考os-dev）

接下来几篇博客应该都是关于bootsector的

~~如果之后有时间的话，可能复现一个早期的linux~~

### boot后的内存布局

可以看到，启动扇区会被放置在`0x7c00-0x7e00`的位置，因此我们在汇编代码中，需要告诉机器，把代码放到`0x7c00`的位置

```assembly
[org 0x7c00]
```

后续加载硬盘也应该尽量加载到空闲的区域，不覆盖还需要使用的部分

![image-20231024154600458](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/202311191715602.png)

### 主要内容

想要完成任何程序都需要先有调试的能力，对于这么底层的程序来说更是了

因此，想要完成一个启动扇区，首先要完成字符串打印到屏幕的工作

打印字符串就是最基本的调试能力

```assembly
[org 0x7c00]
; use bios interrupt to show ascii
; 0x10 is about screen
; setting ah=0x0e means tele-type mode
; setting al="ascii will be type"
start:
	mov bp, 0x8000
	mov sp, 0x8000
	mov bx, hello
	call print_str
	; just wait and do nothing
	; so we could know program is here 
	jmp $

; param:bx
print_str:
	mov ah, 0x0e
L1:	cmp byte [bx], 0
	je 	L2
	mov al, [bx]
	int 0x10
	inc bx
	jmp L1
L2:	ret

hello db 'hello world!', 0ah, 0		; str to print
times 510 - ($ - $$) db 0			; filled with 0x00
dw 0xaa55							; magic num of bootsecto

```

编译生成二进制，交给qemu，qemu会把文件的第一个扇区（512bytes）作为引导扇区引导操作系统

当然，我们这里的引导扇区只打印了hello world，然后死循环，仅仅测试一下字符串打印

别急，面包会有的，牛奶也会有的，操作系统也会有的，先一步步来吧！

```shell
nasm print_str.asm -f bin -o print_str.bin
qemu-system-i386 print_str.bin 
```

hello world总会给人以希望，不是吗？

![image-20231024152350899](https://picgo-111.oss-cn-beijing.aliyuncs.com/img/202311191715950.png)

~~我发现了，在这个短视频时代，博客也不能太长，不然没人看~~

~~又给自己水博客找了个借口，真好啊~~
