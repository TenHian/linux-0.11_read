# ret与栈
```
after_page_tables:
    push 0
    push 0
    push 0
    push L6
    push _main
    jmp setup_paging
...
setup_paging:
    ...
    ret
```  
<font size=4>push 指令就是压栈，五个 push 指令过去后，栈会变成这个样子。</font>  
![43](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/43.png)  
<font size=4>然后注意，setup_paging 最后一个指令是 **ret**，也就是我们上一回讲的设置分页的代码的最后一个指令，形象地说它叫返回指令，但 CPU 可没有那么聪明，它并不知道该返回到哪里执行，只是很机械地**把栈顶的元素值当做返回地址，跳转去那里执行**。</font>  
<font size=4>再具体说是，把 esp 寄存器（栈顶地址）所指向的内存处的值，赋值给 eip 寄存器，而 cs:eip 就是 CPU 要执行的下一条指令的地址。而此时栈顶刚好是 main.c 里写的 main 函数的内存地址，是我们刚刚特意压入栈的，所以 CPU 就理所应当跳过来了。</font>    
<font size=4>至于其他压入栈的 L6 是用作当 main 函数返回时的跳转地址，但由于在操作系统层面的设计上，main 是绝对不会返回的，所以也就没用了。而其他的三个压栈的 0，本意是作为 main 函数的参数，但实际上似乎也没有用到，所以也不必关心。</font>  
<font size=4>于是程序终于跳转到 main.c 这个由 c 语言写就的主函数 main 里了</font>  
```
void main(void) {
    ROOT_DEV = ORIG_ROOT_DEV;
    drive_info = DRIVE_INFO;
    memory_end = (1<<20) + (EXT_MEM_K<<10);
    memory_end &= 0xfffff000;
    if (memory_end > 16*1024*1024)
        memory_end = 16*1024*1024;
    if (memory_end > 12*1024*1024) 
        buffer_memory_end = 4*1024*1024;
    else if (memory_end > 6*1024*1024)
        buffer_memory_end = 2*1024*1024;
    else
        buffer_memory_end = 1*1024*1024;
    main_memory_start = buffer_memory_end;
    mem_init(main_memory_start,memory_end);
    trap_init();
    blk_dev_init();
    chr_dev_init();
    tty_init();
    time_init();
    sched_init();
    buffer_init(buffer_memory_end);
    hd_init();
    floppy_init();
    sti();
    move_to_user_mode();
    if (!fork()) {
        init();
    }
    for(;;) pause();
}
```  
<font size=4>以上就是main函数的全部了，整个操作系统也会最终停留在最后一行死循环中，永不返回，直到关机。</font>  
<br>
<br>
<br>
> 关于 ret 指令，其实 Intel CPU 是配合 call 设计的，有关 call 和 ret 指令，即调用和返回指令，可以参考 Intel 手册：  
> Intel 1 Chapter 6.4 CALLING PROCEDURES USING CALL AND RET