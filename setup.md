# 加载setup.s后记
## 加载内核
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