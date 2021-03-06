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
#### Exercise2: gdb si
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
- boot loader 由两个文件组成
	- boot/boot.S
	- boot/main.c
- BootLoader必须完成的两件事
	- 将cpu从real mode转向32-bit protected mode
	- 通过x86 i/o指令，访问IDE磁盘设备寄存器，从磁盘中读取内核。
#### Exercise3: 设置断点以追踪boot/boot.S and boot/main.c // boot.asm为反编译得出文件
##### Analyze boot.S
```
.globl start
start:
  .code16                     # Assemble for 16-bit mode
  cli                         # Disable interrupts
  cld                         # String operations increment
```
```cli```(disable interrupts). cli is the 1st instruction of boot.S. And CPU works in real mode now.  
```cld``` specifies the direction of the pointer after the String operations.  
```
  # Set up the important data segment registers (DS, ES, SS).
  xorw    %ax,%ax             # Segment number zero
  movw    %ax,%ds             # -> Data Segment
  movw    %ax,%es             # -> Extra Segment
  movw    %ax,%ss             # -> Stack Segment
```
After bios, OS can not ensure the value of theses three segment register.  
These instructions is for resetting of ds,es and ss.  
```
  # Enable A20:
  #   For backwards compatibility with the earliest PCs, physical
  #   address line 20 is tied low, so that addresses higher than
  #   1MB wrap around to zero by default.  This code undoes this.
seta20.1:
  inb     $0x64,%al               # Wait for not busy
  testb   $0x2,%al
  jnz     seta20.1

  movb    $0xd1,%al               # 0xd1 -> port 0x64
  outb    %al,$0x64

seta20.2:
  inb     $0x64,%al               # Wait for not busy 
  testb   $0x2,%al
  jnz     seta20.2

  movb    $0xdf,%al               # 0xdf -> port 0x60
  outb    %al,$0x60
```
Thses instructions are for switching CPU from real mode to protected mode.  
```inb``` and ```outb``` operated on the external devices. (i/o instructions)  
0x64: keyboaed controller read status (MCA) which has bit0 ~ bit7.  
1st part constantly check the value of bit1. While bit1 == 0, the code can continue running.  
part2: put ```0xd1``` in ```0x64```  
And it will check if the data is read. if 1, ```oxdf``` will be put in port 0x60.  
```0xdf``` means that CPU can switch to protected mode.  
```
  # Switch from real to protected mode, using a bootstrap GDT
  # and segment translation that makes virtual addresses 
  # identical to their physical addresses, so that the 
  # effective memory map does not change during the switch.
  lgdt    gdtdesc
  movl    %cr0, %eax
  orl     $CR0_PE_ON, %eax
  movl    %eax, %cr0
```
```lgdt	gdtdesc``` put the value of gdtdesc in GDTR(全局映射描述符表寄存器).  
```orl```意为按位或（常用来把寄存器某些位置为1）  
```$CR0_PE``` = 0x00000001  
Next three structions set the bit1 of CR0 to 1.  
```
The Bit0 of the CR0 register is 
the protected mode boot bit and 
1 represents the protected mode boot.
```
```
  # Jump to next instruction, but in 32-bit code segment.
  # Switches processor into 32-bit mode.
  ljmp    $PROT_MODE_CSEG, $protcseg
```
SWITCH TO THE 32-BIT PROTECTED MODE!!!
```
protcseg:
  # Set up the protected-mode data segment registers
  movw    $PROT_MODE_DSEG, %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS
  movw    %ax, %ss                # -> SS: Stack Segment
```
Set the data segment registers.  
After loading the GDTR Register, we must reload all of the segment registers.  
And cs register need 'Long jump instruction'  //长跳转指令  
Then the value of GDTR can take effect.
```
  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call bootmain
```
Set the value of esp register, and prepare for jumping in the bootmain function.
##### Analyze main.c
```bootmain```
```
	// read 1st page off disk
	readseg((uint32_t) ELFHDR, SECTSIZE*8, 0);
```
Call the ```void readseg(uint32_t, uint32_t, uint32_t);```  
it puts the first 8\*512kb of kernel in ELFHDR(0x10000).  
it equates with we put the head of ```elf``` in the memory.  
```
	// is this a valid ELF?
	if (ELFHDR->e_magic != ELF_MAGIC)
		goto bad;
```
Like the annotation said.  
If we get a valid elf, elf->magic == ELF_MAGIC.  
***CANNOT UNDERSTAND THE ```goto bad```***
```
	ph = (struct Proghdr *) ((uint8_t *) ELFHDR + ELFHDR->e_phoff);
```
The head of ELF has Program Header Table, and phoff is its offset.  
```
	eph = ph + ELFHDR->e_phnum;
```
```e_phnum``` means the table's number of Program Header Table.  
And then eph points the tail of the table.  
```
	for (; ph < eph; ph++)
		// p_pa is the load address of this segment (as well
		// as the physical address)
		readseg(ph->p_pa, ph->p_memsz, ph->p_offset);
```
Load every segment into the memory.  
```ph->p_pa``` means the physics address  
```ph->p_memsz``` means the actual size while the segment is loaded in the memory.  
```ph->p_offset``` offset of the segment's head in the whole ELF.  
```ph->p_filesz``` the size of segment in the ELF. Generally speaking, ```ph->p_memsz```>```ph->p_filesz```
```
	((void (*)(void)) (ELFHDR->e_entry))();
```
```e_entry``` points the entry of kernel file.  
And then the CONTROL will be switched from boot loader to the kernel of system.  
##### Analysis finished, and let's start tracking them.
***To be continued...***
- Loading the Kernel
#### Exercise4: pointers.c
```
gcc pointers.c -o pointers
./pointers
```
AND THEN WE CAN GET
```
1: a = 0x7ffee855d620, b = 0x559ab64c42a0, c = 0x7ffee855dad9
2: a[0] = 200, a[1] = 101, a[2] = 102, a[3] = 103
3: a[0] = 200, a[1] = 300, a[2] = 301, a[3] = 302
4: a[0] = 200, a[1] = 400, a[2] = 301, a[3] = 302
5: a[0] = 200, a[1] = 128144, a[2] = 256, a[3] = 302
6: a = 0x7ffee855d620, b = 0x7ffee855d624, c = 0x7ffee855d621
```
Let's check them one by one...


```
    int a[4];
    int *b = malloc(16);
    int *c;
    int i;

    printf("1: a = %p, b = %p, c = %p\n", a, b, c);
-------
1: a = 0x7ffee855d620, b = 0x559ab64c42a0, c = 0x7ffee855dad9
```
a = The first address of Array A  
b = the address of pointer b
c = the value of pointer c (?



```
    c = a;
    for (i = 0; i < 4; i++)
	a[i] = 100 + i;
    c[0] = 200;
    printf("2: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);
-------
2: a[0] = 200, a[1] = 101, a[2] = 102, a[3] = 103
```
c = a means c = &a\[0].


```
    c[1] = 300;
    *(c + 2) = 301;
    3[c] = 302;
    printf("3: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);
-------
3: a[0] = 200, a[1] = 300, a[2] = 301, a[3] = 302
```
Represents three different ways to access the values in an array.


```.
    c = c + 1;
    *c = 400;
    printf("4: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);
-------
4: a[0] = 200, a[1] = 400, a[2] = 301, a[3] = 302
```
```c=c+1;*c=400;``` means c\[1]=400.


```
    c = (int *) ((char *) c + 1);
    *c = 500;
    printf("5: a[0] = %d, a[1] = %d, a[2] = %d, a[3] = %d\n",
	   a[0], a[1], a[2], a[3]);
-------
5: a[0] = 200, a[1] = 128144, a[2] = 256, a[3] = 302
```
char \*pointer can store 1 byte and int \*pointer can store 4 bytes.  
  
Before that, c = 0x7ffee855d624  
  
now c = 0x7ffee855d625 and if we change \*c, it will influence a\[1] and a\[2].  
  
(if we dont let ```c = (int *) ((char *) c + 1)```,it will only influence a\[1])  
  
a\[1]= 400 = 0x00000190H  
a\[2]= 301 = 0x0000012DH  
500 = 0x000001F4
```
a[1]
0x7ffee855d624 0x7ffee855d625 0x7ffee855d626 0x7ffee855d627
90             01             00             00

a[2]
0x7ffee855d628 0x7ffee855d629 0x7ffee855d62A 0x7ffee855d62B
2D             01             00             00

CHANGE INTO
a[1]
0x7ffee855d624 0x7ffee855d625 0x7ffee855d626 0x7ffee855d627
90             F4             01             00

a[2]
0x7ffee855d628 0x7ffee855d629 0x7ffee855d62A 0x7ffee855d62B
00             01             00             00
```
0x0001F490 = 128144  
0x00000100 = 256


```
    b = (int *) a + 1;
    c = (int *) ((char *) a + 1);
    printf("6: a = %p, b = %p, c = %p\n", a, b, c);
}
------
6: a = 0x7ffee855d620, b = 0x7ffee855d624, c = 0x7ffee855d621
```
The difference of int pointer and char pointer.
#### BACK TO LOAD THE KERNEL
For purposes of 6.828, you can consider an ***ELF*** executable to be a header with ***loading information***, followed by several ***program sections***, each of which is a contiguous chunk of code or data intended to be loaded into memory at a specified address. The boot loader does not modify the code or data; it loads it into memory and starts executing it.   
  
An ELF binary starts with a fixed-length ELF header, followed by a variable-length program header listing each of the program sections to be loaded. The C definitions for these ELF headers are in inc/elf.h. We are interested in three sections:
- .text: executable instructions
- .rodata: read-only data, such as ASCII string constants produced by the C complier.
- .data: data section holds the program's initialized data, such as global variables declared with initializers like ```int x = 5;```.
  
```
umiz@umiz-ubuntu:~/6.828/lab$ objdump -h obj/kern/kernel

obj/kern/kernel:     file format elf32-i386

Sections:
Idx Name          Size      VMA       LMA       File off  Algn
  0 .text         000019e1  f0100000  00100000  00001000  2**4
                  CONTENTS, ALLOC, LOAD, READONLY, CODE
  1 .rodata       000006bc  f0101a00  00101a00  00002a00  2**5
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  2 .stab         00003739  f01020bc  001020bc  000030bc  2**2
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  3 .stabstr      00001529  f01057f5  001057f5  000067f5  2**0
                  CONTENTS, ALLOC, LOAD, READONLY, DATA
  4 .data         00009300  f0107000  00107000  00008000  2**12
                  CONTENTS, ALLOC, LOAD, DATA
......
```
- VMA: address the section expects to excute.
- LMA: memory address the section should be loaded in.
- Typecally, VMA and LMA should be same.
  
  
The boot loader uses the ELF program headers to decide how to load the sections. The program headers specify which parts of the ELF object to load into memory and the destination address each should occupy. Inspect the program headers:
```
objdump -x obj/kern/kernel
```
And we can get:
```
miz@umiz-ubuntu:~/6.828/lab$ objdump -x obj/kern/kernel

obj/kern/kernel:     file format elf32-i386
obj/kern/kernel
architecture: i386, flags 0x00000112:
EXEC_P, HAS_SYMS, D_PAGED
start address 0x0010000c

Program Header:
    LOAD off    0x00001000 vaddr 0xf0100000 paddr 0x00100000 align 2**12
         filesz 0x00006d1e memsz 0x00006d1e flags r-x
    LOAD off    0x00008000 vaddr 0xf0107000 paddr 0x00107000 align 2**12
         filesz 0x0000b6c1 memsz 0x0000b6c1 flags rw-
   STACK off    0x00000000 vaddr 0x00000000 paddr 0x00000000 align 2**4
         filesz 0x00000000 memsz 0x00000000 flags rwx
```
The program headers are then listed under "Program Headers" in the output. The areas of the ELF object that need to be loaded into memory are those that are marked as "LOAD". Other information for each program header is given, such as the virtual address("vaddr"), the physical address("paddr"), and the size of the loaded area("memsz" and "filesz").
##### Exercise 5: trace the first instruction that would "break" if you were to get the boot loader's link addres wrong.
GIVE UP
https://www.cnblogs.com/fatsheep9146/p/5216681.html
##### Exercise 6
### Part3: The Kernel
