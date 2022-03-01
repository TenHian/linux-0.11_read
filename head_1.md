# 分段与分页
## 分页机制
<font size=4>上一章我们说到，head.s 代码在重新设置了 gdt 与 idt 后</font>  
![35](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/35.png)  
<font size=4>开启了分页机制，并跳转到main执行</font>
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
<font size=4>这里我们补充一下**分页机制**如何开启，即**setup_paging**</font>  
```
setup_paging:
    mov ecx,1024*5
    xor eax,eax
    xor edi,edi
    pushf
    cld
    rep stosd
    mov eax,_pg_dir
    mov [eax],pg0+7
    mov [eax+4],pg1+7
    mov [eax+8],pg2+7
    mov [eax+12],pg3+7
    mov edi,pg3+4092
    mov eax,00fff007h
    std
L3: stosd
    sub eax,00001000h
    jge L3
    popf
    xor eax,eax
    mov cr3,eax
    mov eax,cr0
    or  eax,80000000h
    mov cr0,eax
    ret
```  
<font size=4>在讲**分页**前，我们先回顾一下前面说的**分段**，就是通过**全局描述符表（gdt）**计算物理地址</font>  
![29](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/29.png)  
<font size=4>这是在没有开启分页机制的时候，只需要经过这一步转换即可得到最终的物理地址了，但是在开启了分页机制后，又会多一步转换。</font>  
![36](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/36.png)  
<font size=4>在开启分页机制后，逻辑地址仍然要先通过分段机制进行转换，只不过转换后不再是最终的物理地址，而是**线性地址**，然后再通过一次分页机制转换，得到最终的**物理地址**。</font>  
<font size=4>这里我们举个例子，假设我们根据ds中存放的地址，从gdt的数据段取出的32位段基址与偏移地址相加，组成的线性地址为15M，转换为2进制为</font>
```
0000000011_0100000000_000000000000
```  
![37](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/37.png)  
<font size=4>也就是说我们将线性地址划分</font>
> 高 10 位：中间 10 位：后 12 位

<font size=4>高 10 位负责在页目录表中找到一个页目录项，这个页目录项的值加上中间 10 位拼接后的地址去页表中去寻找一个页表项，这个页表项的值，再加上后 12 位偏移地址，就是最终的物理地址。<br>而这一切的操作，都由计算机的一个硬件叫 **MMU**，中文名字叫**内存管理单元**，有时也叫 PMMU，分页内存管理单元。由这个部件来负责将虚拟地址转换为物理地址。<br>所以整个过程我们不用操心，作为操作系统这个软件层，只需要提供好页目录表和页表即可，这种页表方案叫做**二级页表**，第一级叫**页目录表 PDE**，第二级叫**页表 PTE**。他们的结构如下。</font>  
![38](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/38.png)  
<font size=4>之后再开启分页机制的开关。其实就是更改 cr0 寄存器中的一位即可（31 位），还记得我们开启保护模式么，也是改这个寄存器中的一位的值。</font>  
![39](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/39.png)  
<font size=4>然后，**MMU**就可以帮我们进行分页的转换了。此后指令中的内存地址（就是程序员提供的逻辑地址），就统统要先经过分段机制的转换，再通过分页机制的转换，才能最终变成物理地址。</font>  
<font size=4>所以这段代码，就是帮我们把页表和页目录表在内存中写好，之后开启 **cr0 寄存器**的分页开关，仅此而已，我们再把代码贴上来。</font>  
```
setup_paging:
    mov ecx,1024*5
    xor eax,eax
    xor edi,edi
    pushf
    cld
    rep stosd
    mov eax,_pg_dir
    mov [eax],pg0+7
    mov [eax+4],pg1+7
    mov [eax+8],pg2+7
    mov [eax+12],pg3+7
    mov edi,pg3+4092
    mov eax,00fff007h
    std
L3: stosd
    sub eax,00001000h
    jge L3
    popf
    xor eax,eax
    mov cr3,eax
    mov eax,cr0
    or  eax,80000000h
    mov cr0,eax
    ret
```  
<font size=4>那么这段代码实现了什么呢？<br>当时 linux-0.11 认为，总共可以使用的内存不会超过 16M，也即最大地址空间为 0xFFFFFF。<br>而按照当前的页目录表和页表这种机制，1 个页目录表最多包含 1024 个页目录项（也就是 1024 个页表），1 个页表最多包含 1024 个页表项（也就是 1024 个页），1 页为 4KB（因为有 12 位偏移地址），因此，16M 的地址空间可以用 1 个页目录表 + 4 个页表搞定。</font>  
> 4（页表数）* 1024（页表项数） * 4KB（一页大小）= 16MB

<font size=4>上面这段代码就是，**将页目录表放在内存地址的最开头**，还记得上一章开头让你留意的 _pg_dir 这个标签吧？</font>  
```
_pg_dir:
_startup_32:
    mov eax,0x10
    mov ds,ax
    ...
```  
<font size=4>**之后紧挨着这个页目录表，放置 4 个页表**，代码里也有这四个页表的标签项。</font>  
```
.org 0x1000 pg0:
.org 0x2000 pg1:
.org 0x3000 pg2:
.org 0x4000 pg3:
.org 0x5000
```  
<font size=4>最终将页目录表和页表填写好数值，来覆盖整个 16MB 的内存。随后，开启分页机制。此时内存中的页表相关的布局如下。</font>  
![40](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/40.png)   
<font size=4>这些页目录表和页表放到了整个内存布局中最开头的位置，就是覆盖了开头的 system 代码了，不过被覆盖的 system 代码已经执行过了，所以无所谓。<br>同时，如 idt 和 gdt 一样，我们也需要通过一个寄存器告诉 CPU 我们把这些页表放在了哪里，就是这段代码。</font>  
```
xor eax,eax
mov cr3,eax
```  
<font size=4>我们将**cr3寄存器**的值赋为0，**0 地址处就是页目录表，再通过页目录表可以找到所有的页表**，也就相当于 CPU 知道了分页机制的全貌了。<br>现在的内存结构</font>  
![41](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/41.png)  
<font size=4>那么具体页表设置好后，映射的内存是怎样的情况呢？那就要看页表的具体数据了</font>  
```
setup_paging:
    ...
    mov eax,_pg_dir
    mov [eax],pg0+7
    mov [eax+4],pg1+7
    mov [eax+8],pg2+7
    mov [eax+12],pg3+7
    mov edi,pg3+4092
    mov eax,00fff007h
    std
L3: stosd
    sub eax, 1000h
    jpe L3
    ...
```  
<font size=4>对照刚刚的页目录表与页表结构看</font>  
![38](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/38.png)  
<font size=4>前五行表示，页目录表的前 4 个页目录项，分别指向 4 个页表。比如页目录项中的第一项 [eax] 被赋值为 pg0+7，也就是 0x00001007，根据页目录项的格式，表示页表地址为 0x1000，页属性为 0x07 表示改页存在、用户可读写。<br>后面几行表示，填充 4 个页表的每一项，一共 4*1024=4096 项，依次映射到内存的前 16MB 空间。<br>其实就是刚刚的图</font>  
![37](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/37.png)  
<font size=4>经过这套分页机制，线性地址将恰好和最终转换的物理地址一样。<br>现在只有四个页目录项，也就是将前 16M 的线性地址空间，与 16M 的物理地址空间一一对应起来了。</font>  
![42](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/42.png)  
## 小结
<font size=4>我们在此梳理一下<br>Intel 体系结构的**内存管理**可以分成两大部分，也就是标题中的两板斧，**分段**和**分页**。</font>  
> **分段机制**在之前几回已经讨论过多次了，其目的是为了为每个程序或任务提供单独的代码段（cs）、数据段（ds）、栈段（ss），使其不会相互干扰。
> **分页机制**是本章讲的内容，开机后分页机制默认是关闭状态，需要我们手动开启，并且设置好页目录表（PDE）和页表（PTE）。其目的在于可以按需使用物理内存，同时也可以在多任务时起到隔离的作用，这个在后面将多任务时将会有所体会。

<font size=4>以及一些地址</font>  
> **逻辑地址**：我们程序员写代码时给出的地址叫逻辑地址，其中包含段选择子和偏移地址两部分。  
> **线性地址**：通过分段机制，将逻辑地址转换后的地址，叫做线性地址。而这个线性地址是有个范围的，这个范围就叫做线性地址空间，32 位模式下，线性地址空间就是 4G。  
> **物理地址**：就是真正在内存中的地址，它也是有范围的，叫做物理地址空间。那这个范围的大小，就取决于你的内存有多大了。  
> **虚拟地址**：如果没有开启分页机制，那么线性地址就和物理地址是一一对应的，可以理解为相等。如果开启了分页机制，那么线性地址将被视为虚拟地址，这个虚拟地址将会通过分页机制的转换，最终转换成物理地址。

<font size=4>下一章我们将讲一下head.s最后的代码，将main压栈，至于细节，我们下一章慢慢说来</font>  
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