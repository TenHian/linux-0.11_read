# linux0.11源码阅读历程
自己当初阅读源码的历程，在这里整理成笔记，方便自己补充和查阅，也欢迎大家参考我的文章  

**项目架构图**
![24](https://raw.githubusercontent.com/TenHianPic/Picgo/main/Linux/24.png)  

**文章目录**
* **第一部分：进入内核前的工作**
    * [将操作系统移入内存--boot.s](boot.md)
    * [读取和存储一些必要的信息--setup.s](setup.md)
    * [全局描述符表（gdt）和gdtr 寄存器--setup.s](setup_1.md)
    * [进入保护模式--setup.s](setup_2.md)
    * [idt和idtr寄存器--head.s](head.md)
    * [开启分页机制--head.s](head_1.md)
    * [压栈与跳转到main--head .s](head_2.md)
    * [第一部分总结](summary1.md)