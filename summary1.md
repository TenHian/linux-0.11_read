# 第一部分总结
## 第一章
第一章从将启动区代码加载到内存   

![bios](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/bios.png)  

到将全部的操作系统代码移入内存  

![total_setup](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/total_setup.png)  

我们进行了调整内存，将操作系统代码移入内存的过程  

## 第二章
第二章我们首先读取并存储了一些重要的参数  

内存地址|长度（字节）|名称  
-|-|-|
0x90000|2|光标位置
0x90002|2|扩展内存数
0x90004|2|显示页面
0x90006|1|显示模式
0x90007|1|字符列数
0x90008|2|未知
0x9000A|1|显示内存
0x9000B|1|显示状态
0x9000C|2|显卡特性参数
0x9000E|1|屏幕行数
0x9000F|1|屏幕列数
0x90080|16|硬盘1参数表
0x90090|16|硬盘2参数表
0x901FC|2|根设备号  

并且调整的内存布局，使之变化到一个相对稳定的状态，这以后我们不会对内存进行大的改变了  

![25](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/25.png)  
![26](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/26.png)  

## 第三章
我们设置了gdt以为进入保护模式与开启分页机制做准备

```
gdt:
    ;空
    .word   0,0,0,0     ; dummy

    ;代码段描述符
    .word   0x07FF      ; 8Mb - limit=2047 (2048*4096=8Mb)
    .word   0x0000      ; base address=0
    .word   0x9A00      ; code read/exec
    .word   0x00C0      ; granularity=4096, 386

    ;数据段描述符
    .word   0x07FF      ; 8Mb - limit=2047 (2048*4096=8Mb)
    .word   0x0000      ; base address=0
    .word   0x9200      ; data read/write
    .word   0x00C0      ; granularity=4096, 386
```   
## 第四章
我们开启了A20地址线，使我们的寻址能够扩展到32位  

```
; that was painless, now we enable A20

	call	empty_8042
	mov	al,#0xD1		; command write
	out	#0x64,al
	call	empty_8042
	mov	al,#0xDF		; A20 on
	out	#0x60,al
	call	empty_8042
```  

并且对CPU中断进行重新编程，也就是设置了idt

PIC 请求号 | 中断号 | 用途
-|-|-
IRQ0 | 0x20 | 时钟中断
IRQ1 | 0x21 | 键盘中断
IRQ2 | 0x22 | 接连从芯片
IRQ3 | 0x23 | 串口2
IRQ4 | 0x24 | 串口1
IRQ5|0x25	|并口2
IRQ6|	0x26|	软盘驱动器
IRQ7|	0x27|	并口1
IRQ8|	0x28|	实时钟中断
IRQ9|	0x29|	保留
IRQ10|	0x2a|	保留
IRQ11|	0x2b|	保留
IRQ12|	0x2c|	鼠标中断
IRQ13|	0x2d|	数学协处理器
IRQ14|	0x2e|	硬盘中断
IRQ15|	0x2f|	保留  

改变cr0寄存器最后一位，进入了保护模式

![33](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/33.png)  

于是内存就变成了这样  

![32](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/32.png)  

## 第五章
我们重新设置了gdt和idt，使其重新指向我们现在的内存  

![35](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/35.png)  

## 第六章
我们开启了分页机制，并着重讲解了它，这里最好回去大略看一下，复习一下  
[开启分页机制--head.s](head_1.md)  

## 第七章
我们将main压栈，最后程序会跳转到main函数这里执行