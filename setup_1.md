# 解决x86历史遗留问题
## 全局描述符表（gdt）和gdtr 寄存器
<font size=4>我们继续阅读setup.s剩下的内容:</font>  
```
lidt  idt_48      ; load idt with 0,0
lgdt  gdt_48      ; load gdt with whatever appropriate

idt_48:
    .word   0     ; idt limit=0
    .word   0,0   ; idt base=0L
```  
<font size=4>通过讲解这段代码，我们将理解实模式和保护模式的第一个区别，我们首先来回忆一下实模式下CPU计算物理地址的方式:</font>  
> 段基址左移四位，再加上偏移地址:
> ds:3 段寄存器ds左移四位，再加上偏移地址3  

<font size=4>当切换成保护模式后，CPU寻址方式也会发生改变：</font>  
> ds 寄存器里存储的值，在实模式下叫做段基址，在保护模式下叫段选择子。段选择子里存储着段描述符的索引,如图：
> ![27](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/27.png)  
> 通过段描述符索引，可以从全局描述符表 gdt 中找到一个段描述符，段描述符里存储着段基址。
> ![28](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/28.png)
> 段基址取出来，再和偏移地址相加，就得到了物理地址，整个过程如下。
> ![29](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/29.png)
> 总结：段寄存器（比如 ds、ss、cs）里存储的是段选择子，段选择子去全局描述符表中寻找段描述符，从中取出段基址

<font size=4>那么问题来了，**全局描述符表（gdt）**它在哪？CPU如何寻找它？</font>
> gdt在内存中  
> 由操作系统把gdt位置信息存储在gdtr 的寄存器中，就是这条指令：`lgdt    gdt_48`

```
lgdt    gdt_48
```  
<font size=4>lgdt 就表示把后面的值（gdt_48）放在 gdtr 寄存器中，我们来看一下gtd_48标签</font>
```
gdt_48:
    .word   0x800       ; gdt limit=2048, 256 GDT entries
    .word   512+gdt,0x9 ; gdt base = 0X9xxxx
```  
<font size=4>可以看到这个标签位置处表示一个 48 位的数据，其中高 32 位存储着的正是全局描述符表 gdt 的内存地址</font>
> 低16位0x800
> 中16位512+gdt=0x0200+gdt
> 高16位0x9=0x0009
> 高32位:0x90200 + gdt  

<font size=4>gdt 是个标签，表示在本文件内的偏移量，而本文件是 setup.s，编译后是放在 0x90200 这个内存地址的，所以要加上 0x90200 这个值<br>所以我们看出gdtr寄存器高32位就是0x90200 + gdt，即gdt内存起始位置，所以我们只需要找到setup.s中gdt标签的位置看一下它的值就行了，如图：</font>
![30](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/30.png)
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
<font size=4>我们拿来前面的**段描述符**结构对照看一下</font>  
![28](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/28.png)  
<font size=4>我们摘取重要信息，可以看出目前全局描述符表有三个段描述符，第一个为**空**，第二个是**代码段描述符**（type=code），第三个是**数据段描述符**（type=data），**第二个和第三个段描述符的段基址都是 0**，也就是之后在逻辑地址转换物理地址的时候，通过段选择子查找到无论是代码段还是数据段，取出的段基址都是 0，那么物理地址将直接等于程序员给出的逻辑地址（准确说是逻辑地址中的偏移地址）。<br>现在的**全局描述符表**状态：</font>  
![31](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/31.png)  
<font size=4>再接下来我们看看内存布局:(看看0在哪？)</font>
![32](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/32.png)
<font size=4>这里我把 **idtr 寄存器**也画出来了，这个是**中断描述符表**，其原理和全局描述符表一样。全局描述符表是让段选择子去里面寻找段描述符用的，**而中断描述符表是用来在发生中断时，CPU 拿着中断号去中断描述符表中寻找中断处理程序的地址，找到后就跳到相应的中断程序中去执行**，具体我们后面遇到了再说。</font>