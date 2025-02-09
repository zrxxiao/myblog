---
title: 保护模式_part1
cover: false
toc: true
mathjax: true
password:
tags:
- 汇编
- 操作系统
categories:
- 动手写操作系统
---



在XOS下新建chapter3/a文件夹，并把下面内容写到文件pmtest1.asm中

```assembly
; ==========================================
; pmtest1.asm
; 编译方法：nasm pmtest1.asm -o pmtest1.bin
; ==========================================

%include	"pm.inc"	; 常量, 宏, 以及一些说明

org	07c00h
	jmp	LABEL_BEGIN

[SECTION .gdt]
; GDT
;                              段基址,       段界限     , 属性
LABEL_GDT:	   Descriptor       0,                0, 0           ; 空描述符
LABEL_DESC_CODE32: Descriptor       0, SegCode32Len - 1, DA_C + DA_32; 非一致代码段
LABEL_DESC_VIDEO:  Descriptor 0B8000h,           0ffffh, DA_DRW	     ; 显存首地址
; GDT 结束

GdtLen		equ	$ - LABEL_GDT	; GDT长度
GdtPtr		dw	GdtLen - 1	; GDT界限
		dd	0		; GDT基地址

; GDT 选择子
SelectorCode32		equ	LABEL_DESC_CODE32	- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT
; END of [SECTION .gdt]

[SECTION .s16]
[BITS	16]
LABEL_BEGIN:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, 0100h

	; 初始化 32 位代码段描述符
	xor	eax, eax
	mov	ax, cs
	shl	eax, 4
	add	eax, LABEL_SEG_CODE32
	mov	word [LABEL_DESC_CODE32 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE32 + 4], al
	mov	byte [LABEL_DESC_CODE32 + 7], ah
	
	; 为加载 GDTR 作准备
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_GDT		; eax <- gdt 基地址
	mov	dword [GdtPtr + 2], eax	; [GdtPtr + 2] <- gdt 基地址
	
	; 加载 GDTR
	lgdt	[GdtPtr]
	
	; 关中断
	cli
	
	; 打开地址线A20
	in	al, 92h
	or	al, 00000010b
	out	92h, al
	
	; 准备切换到保护模式
	mov	eax, cr0
	or	eax, 1
	mov	cr0, eax
	
	; 真正进入保护模式
	jmp	dword SelectorCode32:0	; 执行这一句会把 SelectorCode32 装入 cs,
					; 并跳转到 Code32Selector:0  处

; END of [SECTION .s16]


[SECTION .s32]; 32 位代码段. 由实模式跳入.
[BITS	32]

LABEL_SEG_CODE32:
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子(目的)

	mov	edi, (80 * 11 + 79) * 2	; 屏幕第 11 行, 第 79 列。
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	al, 'P'
	mov	[gs:edi], ax
	
	; 到此停止
	jmp	$

SegCode32Len	equ	$ - LABEL_SEG_CODE32
; END of [SECTION .s32] 
```

编译
```bash
nasm pmtest1.asm -o pmtest1.bin
```

  pm.inc

```assembly
; 描述符图示

; 图示一
;
;  ------ ┏━━┳━━┓高地址
;         ┃ 7  ┃ 段 ┃
;         ┣━━┫    ┃
;                  基
;  字节 7 ┆    ┆    ┆
;                  址
;         ┣━━┫ ② ┃
;         ┃ 0  ┃    ┃
;  ------ ┣━━╋━━┫
;         ┃ 7  ┃ G  ┃
;         ┣━━╉──┨
;         ┃ 6  ┃ D  ┃
;         ┣━━╉──┨
;         ┃ 5  ┃ 0  ┃
;         ┣━━╉──┨
;         ┃ 4  ┃ AVL┃
;  字节 6 ┣━━╉──┨
;         ┃ 3  ┃    ┃
;         ┣━━┫ 段 ┃
;         ┃ 2  ┃ 界 ┃
;         ┣━━┫ 限 ┃
;         ┃ 1  ┃    ┃
;         ┣━━┫ ② ┃
;         ┃ 0  ┃    ┃
;  ------ ┣━━╋━━┫
;         ┃ 7  ┃ P  ┃
;         ┣━━╉──┨
;         ┃ 6  ┃    ┃
;         ┣━━┫ DPL┃
;         ┃ 5  ┃    ┃
;         ┣━━╉──┨
;         ┃ 4  ┃ S  ┃
;  字节 5 ┣━━╉──┨
;         ┃ 3  ┃    ┃
;         ┣━━┫ T  ┃
;         ┃ 2  ┃ Y  ┃
;         ┣━━┫ P  ┃
;         ┃ 1  ┃ E  ┃
;         ┣━━┫    ┃
;         ┃ 0  ┃    ┃
;  ------ ┣━━╋━━┫
;         ┃ 23 ┃    ┃
;         ┣━━┫    ┃
;         ┃ 22 ┃    ┃
;         ┣━━┫ 段 ┃
;
;   字节  ┆    ┆ 基 ┆
; 2, 3, 4
;         ┣━━┫ 址 ┃
;         ┃ 1  ┃ ① ┃
;         ┣━━┫    ┃
;         ┃ 0  ┃    ┃
;  ------ ┣━━╋━━┫
;         ┃ 15 ┃    ┃
;         ┣━━┫    ┃
;         ┃ 14 ┃    ┃
;         ┣━━┫ 段 ┃
;
; 字节 0,1┆    ┆ 界 ┆
;
;         ┣━━┫ 限 ┃
;         ┃ 1  ┃ ① ┃
;         ┣━━┫    ┃
;         ┃ 0  ┃    ┃
;  ------ ┗━━┻━━┛低地址
;


; 图示二

; 高地址………………………………………………………………………低地址

; |   7   |   6   |   5   |   4   |   3   |   2   |   1   |   0    |
; |7654321076543210765432107654321076543210765432107654321076543210|	<- 共 8 字节
; |--------========--------========--------========--------========|
; ┏━━━┳━━━━━━━┳━━━━━━━━━━━┳━━━━━━━┓
; ┃31..24┃   (见下图)   ┃     段基址(23..0)    ┃ 段界限(15..0)┃
; ┃      ┃              ┃                      ┃              ┃
; ┃ 基址2┃③│②│    ①┃基址1b│   基址1a     ┃    段界限1   ┃
; ┣━━━╋━━━┳━━━╋━━━━━━━━━━━╋━━━━━━━┫
; ┃   %6 ┃  %5  ┃  %4  ┃  %3  ┃     %2       ┃       %1     ┃
; ┗━━━┻━━━┻━━━┻━━━┻━━━━━━━┻━━━━━━━┛
;         │                \_________
;         │                          \__________________
;         │                                             \________________________________________________
;         │                                                                                              \
;         ┏━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┓
;         ┃ 7  ┃ 6  ┃ 5  ┃ 4  ┃ 3  ┃ 2  ┃ 1  ┃ 0  ┃ 7  ┃ 6  ┃ 5  ┃ 4  ┃ 3  ┃ 2  ┃ 1  ┃ 0  ┃
;         ┣━━╋━━╋━━╋━━╋━━┻━━┻━━┻━━╋━━╋━━┻━━╋━━╋━━┻━━┻━━┻━━┫
;         ┃ G  ┃ D  ┃ 0  ┃ AVL┃   段界限 2 (19..16)  ┃  P ┃   DPL    ┃ S  ┃       TYPE           ┃
;         ┣━━┻━━┻━━┻━━╋━━━━━━━━━━━╋━━┻━━━━━┻━━┻━━━━━━━━━━━┫
;         ┃      ③: 属性 2      ┃    ②: 段界限 2      ┃                   ①: 属性1                  ┃
;         ┗━━━━━━━━━━━┻━━━━━━━━━━━┻━━━━━━━━━━━━━━━━━━━━━━━┛
;       高地址                                                                                          低地址
;
;

; 说明:
;
; (1) P:    存在(Present)位。
;		P=1 表示描述符对地址转换是有效的，或者说该描述符所描述的段存在，即在内存中；
;		P=0 表示描述符对地址转换无效，即该段不存在。使用该描述符进行内存访问时会引起异常。
;
; (2) DPL:  表示描述符特权级(Descriptor Privilege level)，共2位。它规定了所描述段的特权级，用于特权检查，以决定对该段能否访问。 
;
; (3) S:   说明描述符的类型。
;		对于存储段描述符而言，S=1，以区别与系统段描述符和门描述符(S=0)。 
;
; (4) TYPE: 说明存储段描述符所描述的存储段的具体属性。
;
;		 
;	数据段类型	类型值		说明
;			----------------------------------
;			0		只读 
;			1		只读、已访问 
;			2		读/写 
;			3		读/写、已访问 
;			4		只读、向下扩展 
;			5		只读、向下扩展、已访问 
;			6		读/写、向下扩展 
;			7		读/写、向下扩展、已访问 
;
;		
;			类型值		说明
;	代码段类型	----------------------------------
;			8		只执行 
;			9		只执行、已访问 
;			A		执行/读 
;			B		执行/读、已访问 
;			C		只执行、一致码段 
;			D		只执行、一致码段、已访问 
;			E		执行/读、一致码段 
;			F		执行/读、一致码段、已访问 
;
;		
;	系统段类型	类型编码	说明
;			----------------------------------
;			0		<未定义>
;			1		可用286TSS
;			2		LDT
;			3		忙的286TSS
;			4		286调用门
;			5		任务门
;			6		286中断门
;			7		286陷阱门
;			8		未定义
;			9		可用386TSS
;			A		<未定义>
;			B		忙的386TSS
;			C		386调用门
;			D		<未定义>
;			E		386中断门
;			F		386陷阱门
;
; (5) G:    段界限粒度(Granularity)位。
;		G=0 表示界限粒度为字节；
;		G=1 表示界限粒度为4K 字节。
;           注意，界限粒度只对段界限有效，对段基地址无效，段基地址总是以字节为单位。 
;
; (6) D:    D位是一个很特殊的位，在描述可执行段、向下扩展数据段或由SS寄存器寻址的段(通常是堆栈段)的三种描述符中的意义各不相同。 
;           ⑴ 在描述可执行段的描述符中，D位决定了指令使用的地址及操作数所默认的大小。
;		① D=1表示默认情况下指令使用32位地址及32位或8位操作数，这样的代码段也称为32位代码段；
;		② D=0 表示默认情况下，使用16位地址及16位或8位操作数，这样的代码段也称为16位代码段，它与80286兼容。可以使用地址大小前缀和操作数大小前缀分别改变默认的地址或操作数的大小。 
;           ⑵ 在向下扩展数据段的描述符中，D位决定段的上部边界。
;		① D=1表示段的上部界限为4G；
;		② D=0表示段的上部界限为64K，这是为了与80286兼容。 
;           ⑶ 在描述由SS寄存器寻址的段描述符中，D位决定隐式的堆栈访问指令(如PUSH和POP指令)使用何种堆栈指针寄存器。
;		① D=1表示使用32位堆栈指针寄存器ESP；
;		② D=0表示使用16位堆栈指针寄存器SP，这与80286兼容。 
;
; (7) AVL:  软件可利用位。80386对该位的使用未左规定，Intel公司也保证今后开发生产的处理器只要与80386兼容，就不会对该位的使用做任何定义或规定。 
;


;----------------------------------------------------------------------------
; 在下列类型值命名中：
;       DA_  : Descriptor Attribute
;       D    : 数据段
;       C    : 代码段
;       S    : 系统段
;       R    : 只读
;       RW   : 读写
;       A    : 已访问
;       其它 : 可按照字面意思理解
;----------------------------------------------------------------------------

; 描述符类型
DA_32		EQU	4000h	; 32 位段

DA_DPL0		EQU	  00h	; DPL = 0
DA_DPL1		EQU	  20h	; DPL = 1
DA_DPL2		EQU	  40h	; DPL = 2
DA_DPL3		EQU	  60h	; DPL = 3

; 存储段描述符类型
DA_DR		EQU	90h	; 存在的只读数据段类型值
DA_DRW		EQU	92h	; 存在的可读写数据段属性值
DA_DRWA		EQU	93h	; 存在的已访问可读写数据段类型值
DA_C		EQU	98h	; 存在的只执行代码段属性值
DA_CR		EQU	9Ah	; 存在的可执行可读代码段属性值
DA_CCO		EQU	9Ch	; 存在的只执行一致代码段属性值
DA_CCOR		EQU	9Eh	; 存在的可执行可读一致代码段属性值

; 系统段描述符类型
DA_LDT		EQU	  82h	; 局部描述符表段类型值
DA_TaskGate	EQU	  85h	; 任务门类型值
DA_386TSS	EQU	  89h	; 可用 386 任务状态段类型值
DA_386CGate	EQU	  8Ch	; 386 调用门类型值
DA_386IGate	EQU	  8Eh	; 386 中断门类型值
DA_386TGate	EQU	  8Fh	; 386 陷阱门类型值


; 选择子图示:
;         ┏━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┳━━┓
;         ┃ 15 ┃ 14 ┃ 13 ┃ 12 ┃ 11 ┃ 10 ┃ 9  ┃ 8  ┃ 7  ┃ 6  ┃ 5  ┃ 4  ┃ 3  ┃ 2  ┃ 1  ┃ 0  ┃
;         ┣━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━┻━━╋━━╋━━┻━━┫
;         ┃                                 描述符索引                                 ┃ TI ┃   RPL    ┃
;         ┗━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━┻━━┻━━━━━┛
;
; RPL(Requested Privilege Level): 请求特权级，用于特权检查。
;
; TI(Table Indicator): 引用描述符表指示位
;	TI=0 指示从全局描述符表GDT中读取描述符；
;	TI=1 指示从局部描述符表LDT中读取描述符。
;

;----------------------------------------------------------------------------
; 选择子类型值说明
; 其中:
;       SA_  : Selector Attribute

SA_RPL0		EQU	0	; ┓
SA_RPL1		EQU	1	; ┣ RPL
SA_RPL2		EQU	2	; ┃
SA_RPL3		EQU	3	; ┛

SA_TIG		EQU	0	; ┓TI
SA_TIL		EQU	4	; ┛
;----------------------------------------------------------------------------



; 宏 ------------------------------------------------------------------------------------------------------
;
; 描述符
; usage: Descriptor Base, Limit, Attr
;        Base:  dd
;        Limit: dd (low 20 bits available)
;        Attr:  dw (lower 4 bits of higher byte are always 0)
%macro Descriptor 3
	dw	%2 & 0FFFFh				; 段界限1
	dw	%1 & 0FFFFh				; 段基址1
	db	(%1 >> 16) & 0FFh			; 段基址2
	dw	((%2 >> 8) & 0F00h) | (%3 & 0F0FFh)	; 属性1 + 段界限2 + 属性2
	db	(%1 >> 24) & 0FFh			; 段基址3
%endmacro ; 共 8 字节
;
; 门
; usage: Gate Selector, Offset, DCount, Attr
;        Selector:  dw
;        Offset:    dd
;        DCount:    db
;        Attr:      db
%macro Gate 4
	dw	(%2 & 0FFFFh)				; 偏移1
	dw	%1					; 选择子
	dw	(%3 & 1Fh) | ((%4 << 8) & 0FF00h)	; 属性
	dw	((%2 >> 16) & 0FFFFh)			; 偏移2
%endmacro ; 共 8 字节
; ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
```

将第二章的a.img和bochsrc复制过来，并执行下面命令生成软盘映像

```bash
dd if=pmtest1.bin of=a.img bs=512 count=1 conv=notrunc  
```

运行Bochs,可以看到， 在屏幕中部右侧， 出现了一个红色的字母“ P” ， 然后再也不动了。  

第二部分

在chapter3下新建b文件夹，复制a文件夹中的bochsrc，pm.inc，pmtest1.asm

1. 到Bochs官方网站下载一个FreeDos。 解压后将其中的a.img复制到我们的工作目录中， 并改名为freedos.img。
2. 用bximage生成一个软盘映像， 起名为pm.img。
3. 修改我们的bochsrc， 确保其中有以下三行：
```配置
floppya: 1_44=freedos.img, status=inserted
floppyb: 1_44=pm.img, status=inserted
boot: a 
```
4.启动Bochs， 待FreeDos启动完毕后用下面命令格式化B:盘   
```
format b:
```
5. 将代码3.1的第8行中的07c00h改为0100h， 并重新编译：
```bash
nasm pmtest1.asm -o pmtest1.com
```
6. 将pmtest1.com复制到虚拟软盘pm.img上：
注意按照书中命令，进行虚拟软盘的挂载时
```bash
sudo mount -o loop pm.img /mnt/floppy
```
会出现了错误
```bash
mount point /mnt/floppy does not exist
```
解决办法：
先用mkdir指令在mnt目录下生成一个floppy 然后执行下面命令：
```bash
sudo losetup /dev/loop0 pm.img
sudo mount /dev/loop0 /mnt/floppy
sudo cp pmtest1.com /mnt/floppy
sudo umount /mnt/floppy
```
7. 到FreeDos中执行如下命令：
```
dir b:
b:pmtest1.com
```

这样pmtest1.com就运行起来了, 一个红色的字母“ P” 出现了。

第三部分
在b文件夹下，新建pmtest2.asm，并把下面内容写到文件中

```assembly
; ==========================================
; pmtest2.asm
; 编译方法：nasm pmtest2.asm -o pmtest2.com
; ==========================================

%include	"pm.inc"	; 常量, 宏, 以及一些说明

org	0100h
	jmp	LABEL_BEGIN

[SECTION .gdt]
; GDT
;                            段基址,        段界限 , 属性
LABEL_GDT:         Descriptor    0,              0, 0         ; 空描述符
LABEL_DESC_NORMAL: Descriptor    0,         0ffffh, DA_DRW    ; Normal 描述符
LABEL_DESC_CODE32: Descriptor    0, SegCode32Len-1, DA_C+DA_32; 非一致代码段, 32
LABEL_DESC_CODE16: Descriptor    0,         0ffffh, DA_C      ; 非一致代码段, 16
LABEL_DESC_DATA:   Descriptor    0,      DataLen-1, DA_DRW    ; Data
LABEL_DESC_STACK:  Descriptor    0,     TopOfStack, DA_DRWA+DA_32; Stack, 32 位
LABEL_DESC_TEST:   Descriptor 0500000h,     0ffffh, DA_DRW
LABEL_DESC_VIDEO:  Descriptor  0B8000h,     0ffffh, DA_DRW    ; 显存首地址
; GDT 结束

GdtLen		equ	$ - LABEL_GDT	; GDT长度
GdtPtr		dw	GdtLen - 1	; GDT界限
		dd	0		; GDT基地址

; GDT 选择子
SelectorNormal		equ	LABEL_DESC_NORMAL	- LABEL_GDT
SelectorCode32		equ	LABEL_DESC_CODE32	- LABEL_GDT
SelectorCode16		equ	LABEL_DESC_CODE16	- LABEL_GDT
SelectorData		equ	LABEL_DESC_DATA		- LABEL_GDT
SelectorStack		equ	LABEL_DESC_STACK	- LABEL_GDT
SelectorTest		equ	LABEL_DESC_TEST		- LABEL_GDT
SelectorVideo		equ	LABEL_DESC_VIDEO	- LABEL_GDT
; END of [SECTION .gdt]

[SECTION .data1]	 ; 数据段
ALIGN	32
[BITS	32]
LABEL_DATA:
SPValueInRealMode	dw	0
; 字符串
PMMessage:		db	"In Protect Mode now. ^-^", 0	; 在保护模式中显示
OffsetPMMessage		equ	PMMessage - $$
StrTest:		db	"ABCDEFGHIJKLMNOPQRSTUVWXYZ", 0
OffsetStrTest		equ	StrTest - $$
DataLen			equ	$ - LABEL_DATA
; END of [SECTION .data1]


; 全局堆栈段
[SECTION .gs]
ALIGN	32
[BITS	32]
LABEL_STACK:
	times 512 db 0

TopOfStack	equ	$ - LABEL_STACK - 1

; END of [SECTION .gs]


[SECTION .s16]
[BITS	16]
LABEL_BEGIN:
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax
	mov	sp, 0100h

	mov	[LABEL_GO_BACK_TO_REAL+3], ax
	mov	[SPValueInRealMode], sp
	
	; 初始化 16 位代码段描述符
	mov	ax, cs
	movzx	eax, ax
	shl	eax, 4
	add	eax, LABEL_SEG_CODE16
	mov	word [LABEL_DESC_CODE16 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE16 + 4], al
	mov	byte [LABEL_DESC_CODE16 + 7], ah
	
	; 初始化 32 位代码段描述符
	xor	eax, eax
	mov	ax, cs
	shl	eax, 4
	add	eax, LABEL_SEG_CODE32
	mov	word [LABEL_DESC_CODE32 + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_CODE32 + 4], al
	mov	byte [LABEL_DESC_CODE32 + 7], ah
	
	; 初始化数据段描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_DATA
	mov	word [LABEL_DESC_DATA + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_DATA + 4], al
	mov	byte [LABEL_DESC_DATA + 7], ah
	
	; 初始化堆栈段描述符
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_STACK
	mov	word [LABEL_DESC_STACK + 2], ax
	shr	eax, 16
	mov	byte [LABEL_DESC_STACK + 4], al
	mov	byte [LABEL_DESC_STACK + 7], ah
	
	; 为加载 GDTR 作准备
	xor	eax, eax
	mov	ax, ds
	shl	eax, 4
	add	eax, LABEL_GDT		; eax <- gdt 基地址
	mov	dword [GdtPtr + 2], eax	; [GdtPtr + 2] <- gdt 基地址
	
	; 加载 GDTR
	lgdt	[GdtPtr]
	
	; 关中断
	cli
	
	; 打开地址线A20
	in	al, 92h
	or	al, 00000010b
	out	92h, al
	
	; 准备切换到保护模式
	mov	eax, cr0
	or	eax, 1
	mov	cr0, eax
	
	; 真正进入保护模式
	jmp	dword SelectorCode32:0	; 执行这一句会把 SelectorCode32 装入 cs, 并跳转到 Code32Selector:0  处

;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;;

LABEL_REAL_ENTRY:		; 从保护模式跳回到实模式就到了这里
	mov	ax, cs
	mov	ds, ax
	mov	es, ax
	mov	ss, ax

	mov	sp, [SPValueInRealMode]
	
	in	al, 92h		; `.
	and	al, 11111101b	;  | 关闭 A20 地址线
	out	92h, al		; /
	
	sti			; 开中断
	
	mov	ax, 4c00h	; `.
	int	21h		; /  回到 DOS

; END of [SECTION .s16]


[SECTION .s32]; 32 位代码段. 由实模式跳入.
[BITS	32]

LABEL_SEG_CODE32:
	mov	ax, SelectorData
	mov	ds, ax			; 数据段选择子
	mov	ax, SelectorTest
	mov	es, ax			; 测试段选择子
	mov	ax, SelectorVideo
	mov	gs, ax			; 视频段选择子

	mov	ax, SelectorStack
	mov	ss, ax			; 堆栈段选择子
	
	mov	esp, TopOfStack


	; 下面显示一个字符串
	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	xor	esi, esi
	xor	edi, edi
	mov	esi, OffsetPMMessage	; 源数据偏移
	mov	edi, (80 * 10 + 0) * 2	; 目的数据偏移。屏幕第 10 行, 第 0 列。
	cld

.1:
	lodsb
	test	al, al
	jz	.2
	mov	[gs:edi], ax
	add	edi, 2
	jmp	.1
.2:	; 显示完毕

	call	DispReturn
	
	call	TestRead
	call	TestWrite
	call	TestRead
	
	; 到此停止
	jmp	SelectorCode16:0

; ------------------------------------------------------------------------
TestRead:
	xor	esi, esi
	mov	ecx, 8
.loop:
	mov	al, [es:esi]
	call	DispAL
	inc	esi
	loop	.loop

	call	DispReturn
	
	ret

; TestRead 结束-----------------------------------------------------------


; ------------------------------------------------------------------------
TestWrite:
	push	esi
	push	edi
	xor	esi, esi
	xor	edi, edi
	mov	esi, OffsetStrTest	; 源数据偏移
	cld
.1:
	lodsb
	test	al, al
	jz	.2
	mov	[es:edi], al
	inc	edi
	jmp	.1
.2:

	pop	edi
	pop	esi
	
	ret

; TestWrite 结束----------------------------------------------------------


; ------------------------------------------------------------------------
; 显示 AL 中的数字
; 默认地:
;	数字已经存在 AL 中
;	edi 始终指向要显示的下一个字符的位置
; 被改变的寄存器:
;	ax, edi
; ------------------------------------------------------------------------
DispAL:
	push	ecx
	push	edx

	mov	ah, 0Ch			; 0000: 黑底    1100: 红字
	mov	dl, al
	shr	al, 4
	mov	ecx, 2

.begin:
	and	al, 01111b
	cmp	al, 9
	ja	.1
	add	al, '0'
	jmp	.2
.1:
	sub	al, 0Ah
	add	al, 'A'
.2:
	mov	[gs:edi], ax
	add	edi, 2

	mov	al, dl
	loop	.begin
	add	edi, 2
	
	pop	edx
	pop	ecx
	
	ret

; DispAL 结束-------------------------------------------------------------


; ------------------------------------------------------------------------
DispReturn:
	push	eax
	push	ebx
	mov	eax, edi
	mov	bl, 160
	div	bl
	and	eax, 0FFh
	inc	eax
	mov	bl, 160
	mul	bl
	mov	edi, eax
	pop	ebx
	pop	eax

	ret

; DispReturn 结束---------------------------------------------------------

SegCode32Len	equ	$ - LABEL_SEG_CODE32
; END of [SECTION .s32]


; 16 位代码段. 由 32 位代码段跳入, 跳出后到实模式
[SECTION .s16code]
ALIGN	32
[BITS	16]
LABEL_SEG_CODE16:
	; 跳回实模式:
	mov	ax, SelectorNormal
	mov	ds, ax
	mov	es, ax
	mov	fs, ax
	mov	gs, ax
	mov	ss, ax

	mov	eax, cr0
	and	al, 11111110b
	mov	cr0, eax

LABEL_GO_BACK_TO_REAL:
	jmp	0:LABEL_REAL_ENTRY	; 段地址会在程序开始处被设置成正确的值

Code16Len	equ	$ - LABEL_SEG_CODE16

; END of [SECTION .s16code]
```

编译：
```bash
nasm pmtest2.asm -o pmtest2.com
```

运行

我们清晰地看到， 程序打印出两行数字， 第一行全部是零， 说明开始内存地址5MB处都是0， 而下一行已经变成了41 42 43…，说明写操作成功。 十六进制的41、 42、 43、 …、 48正是A、 B、 C、 …、 H。同时看到， 程序执行结束后不再像上一个程序那样进入死循环， 而是重新出现了DOS提示符。 这说明我们重新回到了实模式下的DOS。

