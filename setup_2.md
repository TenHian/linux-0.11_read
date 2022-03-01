# 进入保护模式
## 打开A20地址线
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
<font size=4>这段代码的作用是打开A20地址线。什么是A20地址线？这里我们参考  《linux内核完全注释》--赵炯  来说一下。</font>
> 大家都知道8086/8088处理器有20根地址线，能够访问1^20 = 1M内存，实模式访存采用seg x 4 + off（seg为段寄存器，off为段内偏移地址）的方式，访问范围是0（0x0000:0x0000）---10ffef（0xffff:0xffff），对于超过1M的寻址地址将自动环绕（wrap around）到0x0ffef，比如访问地址0x100001时将访问0x0ffef + 1 = 0xfff0物理地址。随后intel发明了80286，地址线有24位，访问地址空间是1^24=16M，并且有一个和8086/8088相互兼容的实模式，如果超过了1M的内存，80286不会回滚到1M内的地址，为了实现完全兼容，intel的工程师使用一个开关来标志是否回滚，打开开关则回滚，关闭则不回滚。当时在主板的8042键盘控制器上刚好有一个空闲的端口，于是就使用该端口来设置这一标志位。发送信号设置该标志位为0则地址线20位及以上的位全被设0，从而实现了回滚，该信号被称为A20，用于设置A20地址线。  
> 
> 8042键盘控制器有4个8位的寄存器，它们是状态寄存器、输出缓冲器、输入缓冲器和控制寄存器。对8042键盘控制器操作使用60h 和 64h这两个端口，0x60是数据端口， 0x64 是命令端口。
>   
> 状态寄存器每一位作用如下：
> > Bit7：从键盘获得的数据奇偶校验错误  
> > Bit6：接收超时，置1  
>> Bit5：发送超时，置1  
>> Bit4：为1，键盘没有被禁止。为0，键盘被禁止  
>> Bit3：为1，输入缓冲器中的内容为命令，为0，输入缓冲器中的内容为数据  
>> Bit2：系统标志，加电启动置0，自检通过后置1  
>> Bit1：输入缓冲器满置1，i8042 取走后置0  
>> Bit0：输出缓冲器满置1，CPU读取后置0  
>
> 控制寄存器各位定义如下：
>> Bit7: 保留
>> Bit6: 将第二套扫描码翻译为第一套
>> Bit5: 置1，禁止鼠标
>> Bit4: 置1，禁止键盘
>> Bit3: 置1，忽略状态寄存器中的 Bit4
>> Bit2: 设置状态寄存器中的 Bit2
>> Bit1: 置1，enable 鼠标中断
>> Bit0: 置1，enable 键盘中断

<font size>完整代码如下</font>
```
org 0x7c00
    mov   ax,cs
    mov   ds,ax
    mov   es,ax
    mov   ss,ax
    mov   sp,0xffff
    jmp   enable_a20    
enable_a20:                      
    call  empty_8042
    mov   al,0xd1
    out   0x64,al
    call  empty_8042
    mov   al,0xdf
    out   0x60,al
    call  empty_8042    
print_msg:
    mov   cx,13                                    
    mov   dx,0x1004                                 
    mov   bx,0x000c                        
    mov   bp,msg_a20_enable                                   
    mov   ax,0x1301                                
    int   0x10     
loop:
    jmp   loop                                      
msg_a20_enable:
    DB "A20 is enable"        
empty_8042:
    dw    00ebh,00ebh            
    in    al,64h            
    test  al,2
    jnz   empty_8042
    ret
    
times 510-($-$$) db 0 
boot_flag:
    DW 0xAA55
```  
<font size=4>mov ss,ax和mov sp,0xffff设置堆栈，不然call xxx, ret这样的子程序调用不能使用，enable_a20子程序通过向0x60、0x64端口写入信号来开启A20地址线。empty_8042子程序判断控制器命令队列是否为空，来检查发出的信号是否被处理。<br>说白了就是将让20位的地址线，通过多余的寄存器，扩展成为能传递32位地址的地址线，以适配保护模式。这段代码现在俨然过时，大家只要理解它大体的扩展原理就行，不必深究。</font>  
## 对可编程中断控制器 8259 芯片进行的编程
<font size=4>下面这段代码也是，我会给你们总结的：</font>
```
; well, that went ok, I hope. Now we have to reprogram the interrupts :-(
; we put them right after the intel-reserved hardware interrupts, at
; int 0x20-0x2F. There they won't mess up anything. Sadly IBM really
; messed this up with the original PC, and they haven't been able to
; rectify it afterwards. Thus the bios puts interrupts at 0x08-0x0f,
; which is used for the internal hardware interrupts as well. We just
; have to reprogram the 8259's, and it isn't fun.

    mov al,#0x11        ; initialization sequence
    out #0x20,al        ; send it to 8259A-1
    .word   0x00eb,0x00eb       ; jmp $+2, jmp $+2
    out #0xA0,al        ; and to 8259A-2
    .word   0x00eb,0x00eb
    mov al,#0x20        ; start of hardware int's (0x20)
    out #0x21,al
    .word   0x00eb,0x00eb
    mov al,#0x28        ; start of hardware int's 2 (0x28)
    out #0xA1,al
    .word   0x00eb,0x00eb
    mov al,#0x04        ; 8259-1 is master
    out #0x21,al
    .word   0x00eb,0x00eb
    mov al,#0x02        ; 8259-2 is slave
    out #0xA1,al
    .word   0x00eb,0x00eb
    mov al,#0x01        ; 8086 mode for both
    out #0x21,al
    .word   0x00eb,0x00eb
    out #0xA1,al
    .word   0x00eb,0x00eb
    mov al,#0xFF        ; mask off all interrupts for now
    out #0x21,al
    .word   0x00eb,0x00eb
    out #0xA1,al
```  
<font size=4>因为中断号是不能冲突的， Intel 把 0 到 0x19 号中断都作为**保留中断**，比如 0 号中断就规定为**除零异常**，软件自定义的中断都应该放在这之后，但是 IBM 在原 PC 机中搞砸了，跟保留中断号发生了冲突，以后也没有纠正过来，所以我们得重新对其进行编程，不得不做，却又一点意思也没有。这是 Linus 在上面注释上的原话。</font>  
<font size=4>所以我们也不必在意，只要知道重新编程之后，8259 这个芯片的引脚与中断号的对应关系，变成了如下的样子就好。</font>  
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

## 切换模式
<font size=4>接下来就是真正的切换模式了</font>
```
mov ax,#0x0001  ; protected mode (PE) bit
lmsw ax      ; This is it;
jmpi 0,8     ; jmp offset 0 of segment 8 (cs)
```  
<font size=4>前两行，将ax赋值0x0001，再将这个值 lmsw --加载机器状态字，也就是改变cr0寄存器的值，末位改成1，如图：</font>
![33](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/33.png)  
<font size=4>所以真正的模式切换十分简单，重要的是之前做的准备工作。</font>  
<font size=4>再往后，又是一个段间跳转指令 jmpi，后面的 8 表示 cs（代码段寄存器）的值，0 表示偏移地址。请注意，此时已经是保护模式了，之前也说过，保护模式下内存寻址方式变了，段寄存器里的值被当做段选择子。<br>回顾一下段选择子的结构</font>  
![27](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/27.png)  
<font size=4>将8转换成2进制00000,0000,0000,1000则得到**描述符索引**值为1，TI为0，RPL为0。</font>  
<font size=4>我们拿着**描述符索引**去**全局描述符表（gdt）**中找第一项段描述符</font>  
```
gdt:
    .word   0,0,0,0     ; dummy

    .word   0x07FF      ; 8Mb - limit=2047 (2048*4096=8Mb)
    .word   0x0000      ; base address=0
    .word   0x9A00      ; code read/exec
    .word   0x00C0      ; granularity=4096, 386

    .word   0x07FF      ; 8Mb - limit=2047 (2048*4096=8Mb)
    .word   0x0000      ; base address=0
    .word   0x9200      ; data read/write
    .word   0x00C0      ; granularity=4096, 386
```  
<font size=4>第 0 项是空值，第一项被表示为代码段描述符，是个可读可执行的段，第二项为数据段描述符，是个可读可写段，不过他们的段基址都是 0。既然段基址为0，偏移地址也为0，那就表示跳到物理地址0处执行，这里回顾一下之前的内存布局图：</font>  
![32](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/32.png)  
<font size=4>就是操作系统全部代码的 system 这个大模块，system 模块怎么生成的呢？由 Makefile 文件可知，是由 head.s 和 main.c 以及其余各模块的操作系统代码合并来的，可以理解为操作系统的全部核心代码编译后的结果。</font>  
```
tools/system: boot/head.o init/main.o \
    $(ARCHIVES) $(DRIVERS) $(MATH) $(LIBS)
    $(LD) $(LDFLAGS) boot/head.o init/main.o \
    $(ARCHIVES) \
    $(DRIVERS) \
    $(MATH) \
    $(LIBS) \
    -o tools/system > System.map
```  
<font size=4>到这里setup.s就讲完了，我们下一章会继续讲head.s，这是最后的汇编代码，之后就是由C写成的main.c了</font>