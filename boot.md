# 开机，将内核移入到内存中
## BIOS进行代码的复制
<font size=4>BIOS 会将硬盘中**启动区的 512 字节**的数据，原封不动复制到**内存中的 0x7c00** 这个位置，并跳转到那个位置进行执行。</font>  
> 什么是启动区？
> 硬盘中的 0 盘 0 道 1 扇区的 512 个字节的最后两个字节分别是 0x55 和 0xaa，那么 BIOS 就会认为它是个启动区。<br>
> 作为操作系统的开发人员，仅仅需要把操作系统最开始的那段代码，编译并存储在硬盘的 0 盘 0 道 1 扇区即可。之后 BIOS 会帮我们把它放到内存里，并且跳过去执行。  

![bios](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/bios.png)
<font size=4>Linux-0.11 的最开始的代码，就是这个用汇编语言写的 bootsect.s，路径`/boot/bootsect.s`。通过编译，这个 bootsect.s 会被编译成二进制文件，存放在启动区的第一扇区。<br>于是我们开始解读`/boot/bootsect.s`</font>
```x86
mov ax,0x07c0
mov ds,ax
```
<font size=4>将ds数据段寄存器赋为`0x07c0`。<br>我们知道x86汇编语言的数据寻址方式是 `ds << 4位` 再加上`偏移地址`，执行以上代码后，CPU下次取数据就是从 0x07c0 << 4 ==0x7c00 这个地址取。</font>  

```x86
mov ax,0x07c0
mov ds,ax
mov ax,0x9000
mov es,ax
mov cx,#256
sub si,si
sub di,di
rep movw
```
<font size=4>`mov ax,0x9000; mov es,ax`将es(附加寄存器)赋为`0x9000`；`sub si,si; 
sub di,di`将si和di清空<br>现在各寄存器的值如下：</font>
> ds = 0x07c0  
> es = 0x9000  
> cx = 256  
> si = 0  
> di = 0  

<font size=4>而`rep movw`这个命令就是重复执行(rep)复制一个**字**(16位)256次(cx寄存器的值);从`ds:si`处复制到`es:di`处。<br>也就是说将从0x7c00处开始的512字节数据复制到0x90000处</font>  
```x86
jmpi go,0x9000
go: 
  mov ax,cs
  mov ds,ax
```  
<font size=4>`jmpi go,0x9000`,跳转到0x9000:go处执行，go是一个标签，它表示go后第一行代码在编译后的二进制文件的地址，如果`mov ax,cs`在编译后的二进制文件中的地址为0x08，则`jmpi go,0x9000`就跳转到0x90008处执行。<br>也就是说上面的操作就是把启动区的512字节的数据从硬盘移动到内存0x7c000处，再移动到0x90000处</font>  
```x86
go: mov ax,cs
    mov ds,ax
    mov es,ax
    mov ss,ax
    mov sp,#0xFF00
```  
<font size=4>把cs(代码段寄存器)的值赋给ds(数据段寄存器)、es(附加段寄存器)、ss(堆栈段寄存器)、把0xFF00赋给sp(堆栈指针)<br>由于我们在执行go之前，执行了一条跳转指令`jmpi go,0x9000`，所以在go开始时cs的值为0x9000不变，ip(指令指针)的值为go标签的偏移地址<br>ss是堆栈段寄存器，结合sp(堆栈指针)组成ss:sp指向0x9FF00，即栈顶地址<br>到此为止，我们完成了一些最最基本的准备工作:</font>  
1. 代码从硬盘移到内存，又从内存挪了个地方，放在了 0x90000 处
2. 数据段寄存器 ds 和代码段寄存器 cs 此时都被设置为了 0x9000，也就为跳转代码和访问内存数据，奠定了同一个内存的基址地址，方便了跳转和内存访问，我们仅仅需要指定偏移地址即可了
3. 栈顶地址被设置为了 0x9FF00，具体表现为栈段寄存器 ss 为 0x9000，栈基址寄存器 sp 为 0xFF00。栈是向下发展的，这个栈顶地址 0x9FF00 要远远大于此时代码所在的位置 0x90000，所以栈向下发展就很难撞见代码所在的位置，也就比较安全。这也是为什么给栈顶地址设置为这个值的原因，其实只需要离代码的位置远远的即可  

![准备工作流程图](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/%E5%87%86%E5%A4%87%E5%B7%A5%E4%BD%9C%E6%B5%81%E7%A8%8B%E5%9B%BE.png)
![寄存器(1)](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/%E5%AF%84%E5%AD%98%E5%99%A8(1).png)  
## 复制setup.s到内存  
```x86
load_setup:
    mov dx,#0x0000      ; drive 0, head 0
    mov cx,#0x0002      ; sector 2, track 0
    mov bx,#0x0200      ; address = 512, in 0x9000
    mov ax,#0x0200+4    ; service 2, nr of sectors
    int 0x13            ; read it
    jnc ok_load_setup       ; ok - continue
    mov dx,#0x0000
    mov ax,#0x0000      ; reset the diskette
    int 0x13
    jmp load_setup
```  
<font size=4>这段代码的注释写的很明确，将0磁盘0磁道的0个磁头2扇区开始的数据加载到内存0x90200处，共加载4个扇区。形象化则如图所示：</font>
![load_setup](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/load_setup.png)  
<font size=4>如果加载成功，则跳转到ok_load_setup:执行；若不成功，则重试这一步骤<br>有关`int 0x13`所引发的13号中断，读取磁盘，详情见：</font>  
## 复制head.s到内存
```x86
ok_load_setup:
    ...
    mov ax,#0x1000
    mov es,ax       ; segment of 0x10000
    call read_it
    ...
    jmpi 0,0x9020
```  
<font size=4>这里省略了许多不重要的代码，如打印"Loading system ..."等。这段代码的作用和上面的`load_setup`大同小异，就是把磁盘从6扇区往后的240个扇区，通过`read_it`函数，加载到内存0x10000处。<br>至此，整个操作系统的全部代码都已从硬盘中搬运到内存里来了。然后我们会跳到0x90200处开始执行，也就是移动到内存中来的磁盘2扇区开始处的内容。</font>  
![ok_load_setup](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/ok_load_setup.png)

<font size=4>好的,在此章结束之处我们谈一些额外的:</font>
1. 上文我们说的把代码从硬盘搬运到内存等等的操作，并不是搬运我们写的汇编代码，而是编译后的机器码；众所周知，汇编代码是机器码的主机符，CPU确确实实是按照我们上面说的一步一步进行的
2. 所以如何编译这些汇编代码呢？整个操作系统的编译过程是`Makefile`和`build.c`配合完成的，最终结果为：
   > 1. 把 bootsect.s 编译成 bootsect 放在硬盘的 1 扇区。
   > 2. 把 setup.s 编译成 setup 放在硬盘的 2~5 扇区。
   > 3. 把剩下的全部代码（head.s 作为开头）编译成 system 放在硬盘的随后 240 个扇区。
3. 下图为整体过程：
![total_setup](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/total_setup.png)
<font size=4>上面我们说到，我们即将跳转到的内存中的 0x90200 处执行代码，就是从硬盘第二个扇区开始处加载到内存的。第二个扇区的最开始处，那也就是 setup.s 文件的第一行代码。于是我们就向着setup.s前进了。</font>