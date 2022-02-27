# 加载内核后记
## 读取和存储一些必要的信息
```x86
start:
    mov ax,#0x9000  ; this is done in bootsect already, but...
    mov ds,ax
    mov ah,#0x03    ; read cursor pos
    xor bh,bh
    int 0x10        ; save it in known place, con_init fetches
    mov [0],dx      ; it from 0x90000
```  
<font size=4>我们来解释一下0x10这个中断，如果想要了解更多与中断有关的信息，请阅读</font>
<font size=4>int 0x10是触发 BIOS 提供的**显示服务中断处理程序**，而 ah 寄存器被赋值为 0x03 表示显示服务里具体的读取光标位置功能<br>这个 int 0x10 中断程序执行完毕并返回时，dx 寄存器里的值表示光标的位置，具体说来其高八位 dh 存储了行号，低八位 dl 存储了列号。</font>
> 说明：计算机在加电自检后会自动初始化到文字模式，在这种模式下，一屏幕可以显示 25 行，每行 80 个字符，也就是 80 列。  

<font size=4>`mov [0],dx` 就是把这个光标位置存储在 [0] 这个内存地址处。注意，前面我们说过，这个内存地址仅仅是偏移地址，还需要加上 ds 这个寄存器里存储的段基址，最终的内存地址是在 0x90000 处，这里存放着光标的位置，以便之后在初始化控制台的时候用到。</font>  
<font size=4>再接下来的几行代码，都是和刚刚一样的逻辑，调用一个 BIOS 中断获取点什么信息，然后存储在内存中某个位置</font>
```
获取内存信息。
; Get memory size (extended mem, kB)
    mov ah,#0x88
    int 0x15
    mov [2],ax
获取显卡显示模式。
; Get video-card data:
    mov ah,#0x0f
    int 0x10
    mov [4],bx      ; bh = display page
    mov [6],ax      ; al = video mode, ah = window width
检查显示方式并取参数
; check for EGA/VGA and some config parameters
    mov ah,#0x12
    mov bl,#0x10
    int 0x10
    mov [8],ax
    mov [10],bx
    mov [12],cx
获取第一块硬盘的信息。
; Get hd0 data
    mov ax,#0x0000
    mov ds,ax
    lds si,[4*0x41]
    mov ax,#INITSEG
    mov es,ax
    mov di,#0x0080
    mov cx,#0x10
    rep
    movsb
获取第二块硬盘的信息。
; Get hd1 data
    mov ax,#0x0000
    mov ds,ax
    lds si,[4*0x46]
    mov ax,#INITSEG
    mov es,ax
    mov di,#0x0090
    mov cx,#0x10
    rep
    movsb
```  
<font size=4>经此之后，内存变化为：</font>  
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

<font size=4>将这些信息存储好后，我们继续向下看：</font>  
```
cli         ; no interrupts allowed ;
```  
<font size=4>我们用`cli`关闭中断，因为后面我们要把原本是 BIOS 写好的中断向量表给覆盖掉，写上我们自己的中断向量表，所以这个时候是不允许中断进来的。这个命令在为以后做准备，我们以后会用到，到时候会提起这个地方。
</font>  
## 移动操作系统内核的主要代码
```
; first we move the system to it's rightful place
    mov ax,#0x0000
    cld         ; 'direction'=0, movs moves forward
do_move:
    mov es,ax       ; destination segment
    add ax,#0x1000
    cmp ax,#0x9000
    jz  end_move
    mov ds,ax       ; source segment
    sub di,di
    sub si,si
    mov cx,#0x8000
    rep movsw
    jmp do_move
; then we load the segment descriptors
end_move:
    ...
```  
<font size=4>上面代码中，我们把内存地址 0x10000 处开始往后一直到 0x90000 的内容，统统复制到内存的最开始的 0 位置。<br>在这里我们重新梳理一下内存信息：</font>  
![25](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/25.png)
1. 栈顶地址仍然是 0x9FF00 没有改变
2. 0x90000 开始往上的位置，原来是 bootsect 和 setup 程序的代码，现 bootsect 的一部分代码在已经被操作系统为了记录内存、硬盘、显卡等一些临时存放的数据给覆盖了一部分
3. 内存最开始的 0 到 0x80000 这 512K 被 system 模块给占用了，之前讲过，这个 system 模块就是除了 bootsect 和 setup 之外的全部程序链接在一起的结果，可以理解为操作系统的全部。  

<font size=4>于是现在的内存就变化为了下图：</font>  
![26](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/26.png)  
<font size=4>进行了这些准备后，下一章我们要干一件大事：**模式的转换**,从现在的 **16 位的实模式**转变为之后**32 位的保护模式**</font>