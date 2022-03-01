# head.s
## 回顾
<font size=4>上一章我们配置了全局描述符表 gdt 和中断描述符表 idt</font>
```
lidt  idt_48
lgdt  gdt_48
```  
<font size=4>打开了A20地址线</font>
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
<font size=4>然后更改 cr0 寄存器开启保护模式</font>
```
mov ax,#0x0001
lmsw ax
```  
<font size=4>跳转到内存地址 0 处开始执行代码</font>
```
jmpi 0,8
```  
<font size=4>0 位置处存储着操作系统全部核心代码，是由 head.s 和 main.c 以及后面的无数源代码文件编译并链接在一起而成的 system 模块</font>  
![32](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/32.png)    
<font size=4>这章，我们要开始讲head.s</font>  
## head.s
```
_pg_dir:
_startup_32:
    mov eax,0x10
    mov ds,ax
    mov es,ax
    mov fs,ax
    mov gs,ax
    lss esp,_stack_start
```  
<font size=4>我们注意开头的标号 _pg_dir ,这个表示**页目录**，之后在设置分页机制时，页目录会存放在这里，也会覆盖这里的代码。</font>  
<font size=4>再往下连续五个 mov 操作，分别给 ds、es、fs、gs 这几个段寄存器赋值为 0x10，根据段描述符结构解析，表示这几个段寄存器的值为指向全局描述符表中的第二个段描述符，也就是数据段描述符。</font>  
<font size=4>最后 lss 指令相当于让 ss:esp 这个栈顶指针指向了 _stack_start 这个标号的位置。(原来栈顶指向0x9FF00)<br>stack_start 标号定义在了很久之后才会讲到的 sched.c 里，我们提取部分讲一下</font>
```C  
long user_stack [ PAGE_SIZE>>2 ] ;

struct {
	long * a;
	short b;
	} stack_start = { & user_stack [PAGE_SIZE>>2] , 0x10 };
```  
<font size=4>stack_start 结构中的高位 8 字节是 0x10，将会赋值给 ss 栈段寄存器，低位 16 字节是 user_stack 这个数组的最后一个元素的地址值，将其赋值给 esp 寄存器。<br>赋值给 ss 的 0x10 仍然按照保护模式下的**段选择子**去解读，其指向的是全局描述符表中的第二个段描述符（数据段描述符），段基址是 0<br>赋值给 esp 寄存器的就是 user_stack 数组的最后一个元素的内存地址值，那最终的栈顶地址，也指向了这里（user_stack + 0），后面的压栈操作，就是往这个新的栈顶地址处压。</font>     
```
call setup_idt ;设置中断描述符表
call setup_gdt ;设置全局描述符表
mov eax,10h
mov ds,ax
mov es,ax
mov fs,ax
mov gs,ax
lss esp,_stack_start
```  
<font size=4>先设置了 idt 和 gdt，然后又重新执行了一遍刚刚执行过的代码<br>为什么要重新设置这些段寄存器呢？因为上面修改了 gdt，所以要重新设置一遍以刷新才能生效。那我们接下来就把目光放到设置 idt 和 gdt 上。<br>中断描述符表 idt 我们之前没讲过，我们现在来看一下。</font>  
```
setup_idt:
    lea edx,ignore_int
    mov eax,00080000h
    mov ax,dx
    mov dx,8E00h
    lea edi,_idt
    mov ecx,256
rp_sidt:
    mov [edi],eax
    mov [edi+4],edx
    add edi,8
    dec ecx
    jne rp_sidt
    lidt fword ptr idt_descr
    ret

idt_descr:
    dw 256*8-1
    dd _idt

_idt:
    DQ 256 dup(0)
```  
<font size=4>中断描述符表 idt 里面存储着一个个中断描述符，每一个中断号就对应着一个中断描述符，而中断描述符里面存储着主要是中断程序的地址，这样一个中断号过来后，CPU 就会自动寻找相应的中断程序，然后去执行它。<br>
那这段程序的作用就是，**设置了 256 个中断描述符**，并且让每一个中断描述符中的中断程序例程都指向一个**ignore_int**的函数地址，这个是个**默认的中断处理程序**，之后会逐渐被各个具体的中断程序所覆盖。比如之后键盘模块会将自己的键盘中断处理程序，覆盖过去。<br>那现在，产生任何中断都会指向这个默认的函数 **ignore_int**，也就是说现在这个阶段你按键盘还不好使。
<br>设置中断描述符表**setup_idt**说完了，那接下来 **setup_gdt** 就同理了。我们就直接看设置好后的新的全局描述符表长什么样吧？</font>
```
_gdt:
    DQ 0000000000000000h    ;/* NULL descriptor */
    DQ 00c09a0000000fffh    ;/* 16Mb */
    DQ 00c0920000000fffh    ;/* 16Mb */
    DQ 0000000000000000h    ;/* TEMPORARY - don't use */
    DQ 252 dup(0)
```  
<font size=4>其实和我们原先设置好的 gdt 一模一样。</font>  
<font size=4>也是第一段描述符为**空**接着**代码段描述符**和**数据段描述符**，然后第四项**系统段描述符**并没有用到，不用管。最后还留了 252 项的空间，这些空间后面会用来放置**任务状态段描述符 TSS** 和**局部描述符 LDT**，这个后面再说。</font>  
![34](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/34.png)  
<font size=4>也就是将原本在setup程序中的idt、gdt移动到head程序中，因为以后原setup程序在内存中的位置要被新程序覆盖掉</font>  
![35](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/35.png)  
```
jmp after_page_tables
...
after_page_tables:
    push 0
    push 0
    push 0
    push L6
    push _main
    jmp setup_paging
L6:
    jmp L6
```  
<font size=4>开启分页机制，并且跳转到 main 函数</font>