# Lab1: Booting A PC
### Part1 PC BootStrap
- real mode and protected mode
	- 段内地址 + 偏移地址
		```
		内存不分段，全部连续排列。
		但由于一个寄存器16位无法表示全部的1m（20位）内存
		因此取另一个寄存器的四位作段地址（2^4)
		因此物理地址 = 左移四位的段地址 + 偏移地址
		```
	- 段基址： 段内偏移
		``` 
		段基址放在段寄存器里，分为cs代码段 ds数据段 ss堆栈段 es扩展段 
		```
- ***Exercise2***: gdb si
	- 0xffff0:	ljmp $0xf000, $0xe05b
		```
		跳转指令，跳转到后者
		0xf000 << 4+0xe5b = 0xfe5b
		```
	- 0xfe05b:	cmpl $0x0, $cs:0x6ac8
		```
		比较用指令
		```
	- 0xfe062:	jne 0xfd2e1
		```
		jne指令，zf标志位为0时跳转（zf标志位为上一个cmpl指令的比较结果，
		$cs:0x6ac8的值不为0x0时跳转
		```
	- 0xfe066:	xor %dx, %dx
		```
		将dx寄存器清零
		```
整个bios的操作为控制、初始化、检测各种底层设备，如时钟、gdtr寄存器。  
以及设置中断向量表（依托中断实现进程的并发  
作为启动后第一段程序，最重要的功能为***把操作系统从磁盘中导入内存，再把控制权交给操作系统***  
bios确认操作系统所在位置（磁盘）后，会访问其第一扇区（启动区boot sector）  
启动区会有***boot loader***，负责将整个操作系统导入内存并进行配置。  
bios -> boot loader -> kernel
### Part2 The Boot Loader
