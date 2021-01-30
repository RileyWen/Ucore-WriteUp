# UCore-Lab 1

## Exersise 1: 理解通过make生成执行文件的过程。

### 1. 操作系统镜像文件ucore.img 是如何一步一步生成的？(需要比较详细地解释Makefile 中每一条相关命令和命令参数的含义，以及说明命令导致的结果)
以下所有Makefile Script中：

- `$(call create_target, xxxxx)`是添加对应的dependencies，并解除对于其目录的dependency。
- `$(call totarget, bin_files)`是在`bin_files`之前添加`bin/`的路径名称。
- `$(V)`是作者用来控制`Verbose Mode`的开关，当`V`为其默认值`@`的时候，所有Makefile中跟在`@`后面的命令执行将不输出。

#### ucore.img的拼接代码

```makefile
# create ucore.img                                       
UCOREIMG   := $(call totarget,ucore.img)                                       
                                                         
$(UCOREIMG): $(kernel) $(bootblock)                      
    $(V)dd if=/dev/zero of=$@ count=10000                
    $(V)dd if=$(bootblock) of=$@ conv=notrunc            
    $(V)dd if=$(kernel) of=$@ seek=1 conv=notrunc        
                                                         
$(call create_target,ucore.img)  
```

- `dd`的默认`block size`是512B，以上代码没有声明，所以其`block size`即为512B。
- `dd`的`conv=notrunc`参数指定了不清空原始文件。
- `dd if=/dev/zero of=$@ count=10000`将生成一个大小为5120000B的空白文件。
- `dd if=$(bootblock) of=$@ conv=notrunc`将引导块写入第一个block。
- `dd if=$(kernel) of=$@ seek=1 conv=notrunc`将kernel部分写入第一个block之后。

#### kernel部分的编译代码

```makefile
KINCLUDE	+= kern/debug/ \
			   kern/driver/ \
			   kern/trap/ \
			   kern/mm/
KSRCDIR		+= kern/init \
			   kern/libs \
			   kern/debug \
			   kern/driver \
			   kern/trap \
			   kern/mm
KCFLAGS		+= $(addprefix -I,$(KINCLUDE))

$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))
KOBJS	= $(call read_packet,kernel libs)

# create kernel target
kernel = $(call totarget,kernel)
$(kernel): tools/kernel.ld
$(kernel): $(KOBJS)
	@echo + ld $@
	$(V)$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)
	@$(OBJDUMP) -S $@ > $(call asmfile,kernel)
	@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' \ 
	> $(call symfile,kernel)

$(call create_target,kernel)
```

- `$(call add_files_cc,$(call listf_cc,$(KSRCDIR)),kernel,$(KCFLAGS))`将编译kernel依赖的`KSRCDIR`中的source files和`KINCLUDE`中的header files，编译参数为`KCFLAGS`。
- `KOBJS = $(call read_packet,kernel libs)`将`KOBJS`赋值为上一行中编译的所有文件的list。
- `kernel.ld`指定了kernel部分的VMA和各个section的分布。
- `$(LD) $(LDFLAGS) -T tools/kernel.ld -o $@ $(KOBJS)`将各个relocatable object file按`kernel.ld`指定的规则链接起来。
- `$(OBJDUMP) -S $@ > $(call asmfile,kernel)`根据符号表输出汇编和源代码对照的文本。
- `@$(OBJDUMP) -t $@ | $(SED) '1,/SYMBOL TABLE/d; s/ .* / /; /^$$/d' > $(call symfile,kernel)`输出了每个符号对应的地址。

#### bootblock的编译代码

```makefile
# create bootblock                                                              
bootfiles = $(call listf_cc,boot)                                               
$(foreach f,$(bootfiles),$(call cc_compile,$(f),$(CC),$(CFLAGS) -Os -nostdinc)) 
                                                                                
bootblock = $(call totarget,bootblock)                                          
                                                                                
$(bootblock): $(call toobj,$(bootfiles)) | $(call totarget,sign)                
    @echo + ld $@                                                               
    $(V)$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)
    @$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)        
    @$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)
    @$(call totarget,sign) $(call outfile,bootblock) $(bootblock)
                                                                                
$(call create_target,bootblock)                                                 
```

- `$(LD) $(LDFLAGS) -N -e start -Ttext 0x7C00 $^ -o $(call toobj,bootblock)`将各个.o文件链接成boot block。
  - 上述命令中，`ld`的几个参数解释：
    - `-N` 设置代码段和数据段均可读写。
    - `-e` 指定Enrty Point。
    - `-Ttext` 指定代码段被加载时的内存地址。
- `$(OBJDUMP) -S $(call objfile,bootblock) > $(call asmfile,bootblock)`同上，根据符号表输出汇编和源代码对照的文本。
- `$(OBJCOPY) -S -O binary $(call objfile,bootblock) $(call outfile,bootblock)`
  - `-S`剥离符号表和重定位信息。
  - `-O binary`导出boot block的memory dump，并消除重定位。
- `$(call totarget,sign) $(call outfile,bootblock) $(bootblock)`调用`sign.c`中的代码，将bootblock加上引导块的标记。

### 2. 一个被系统认为是符合规范的硬盘主引导扇区的特征是什么？

- 0~445 Byte 是 Bootstrap code area。
- 446~509 Byte 是一个存放Partition Entry、大小为4、每个Partition Entry大小为16的数组。
- 510和511 Byte存放`0x55AA`，这是Boot Signature。

## Exersise 2：使用qemu 执行并调试lab1中的软件

因为`gdb`默认是在`x86`模式下运行，为了调试方便，将lab1中提供的gdbinit文件改为如下

```
set architecture i8086        
                              
define hook-stop              
printf "[%x:%x]   ", $cs, $eip
x/4bx (( $cs << 4  ) + $eip)  
printf "[%x:%x]", $cs, $eip   
x/1i (( $cs << 4 ) + $eip)    
                              
end                           
                              
target remote :1235                   
```

然后在terminal中执行如下指令启动`qemu`：

```bash
qemu-system-i386 -S -gdb tcp::1235 -serial mon:stdio -nographic -drive format=raw,file=./bin/ucore.img
```

然后在另外一个窗口中启动`gdb`：

```bash
gdb -x tools/gdbinit -q
```

首先断点下在了

```assembly
[f000:fff0]   0xffff0:  0xea    0x5b    0xe0    0x00    0xf0    0x30    0x36
[f000:fff0]   0xffff0:  ljmp   $0x3630,$0xf000e05b                          
```

上面那条指令跳转到了BIOS 的加电自检程序 POST (Power-On Self-Test)，这个`$cs`的值是`0x3630`很奇怪。故去翻阅了一下`x86 Instruction Set Reference`

| **Opcode** | **Mnemonic** | Description                                                 |
| ---------- | ------------ | ------------------------------------------------------------ |
| E9 cw      | JMP rel16    | Jump near, relative, displacement relative to next instruction. |
| EA cd      | JMP ptr16:16 | Jump far, absolute, address given in operand.                |

可以知道这条指令其实是far jmp到[F000:E05B]。所以`0x3630`的值应该是gdb翻译错误了。

POST部分的代码不是关注重点，总之知道最后会跳转到`0x7c00`就好了，所以直接通过`b *0x7c00`在第一条引导指令下断点。可以看到这些代码与`bootasm.S`中的一致。

```assembly
[f000:fff0]   0xffff0:  0xea    0x5b    0xe0    0x00    0xf0    0x30    0x36
[f000:fff0]   0xffff0:  ljmp   $0x3630,$0xf000e05b                          
0x0000fff0 in ?? ()                                                         
(gdb)  b *0x7c00                                                            
Breakpoint 1 at 0x7c00                                                      
(gdb) c                                                                     
Continuing.                                                                 
[0:7c00]   0x7c00:      0xfa    0xfc    0x31    0xc0    0x8e    0xd8    0x8e
[0:7c00]=> 0x7c00:      cli      
Breakpoint 1, 0x00007c00 in ?? ()                                           
(gdb) nexti                                                                 
[0:7c01]   0x7c01:      0xfc    0x31    0xc0    0x8e    0xd8    0x8e    0xc0
[0:7c01]=> 0x7c01:      cld                                                 
0x00007c01 in ?? ()                                                         
(gdb)                                                                       
[0:7c02]   0x7c02:      0x31    0xc0    0x8e    0xd8    0x8e    0xc0    0x8e
[0:7c02]=> 0x7c02:      xor    %eax,%eax                                    
0x00007c02 in ?? ()                                                         
(gdb)                                                                       
[0:7c04]   0x7c04:      0x8e    0xd8    0x8e    0xc0    0x8e    0xd0    0xe4
[0:7c04]=> 0x7c04:      mov    %eax,%ds                                     
0x00007c04 in ?? ()                                                         
(gdb)                                                                       
[0:7c06]   0x7c06:      0x8e    0xc0    0x8e    0xd0    0xe4    0x64    0xa8
[0:7c06]=> 0x7c06:      mov    %eax,%es                                     
0x00007c06 in ?? ()                                                         
(gdb)                                                                       
[0:7c08]   0x7c08:      0x8e    0xd0    0xe4    0x64    0xa8    0x02    0x75
[0:7c08]=> 0x7c08:      mov    %eax,%ss                                     
0x00007c08 in ?? ()                                                         
(gdb)                                                                       
[0:7c0a]   0x7c0a:      0xe4    0x64    0xa8    0x02    0x75    0xfa    0xb0
[0:7c0a]=> 0x7c0a:      in     $0x64,%al                                    
0x00007c0a in ?? ()                                                         
```

另外下断点就直接`b <function name>`或者`b <*address>`就好了，此处不再赘述。略略略，不写了╮(￣▽￣)╭

## Exersise 3：分析bootloader进入保护模式的过程。

从real mode进入protected mode大致有3步:

- 开A20 Gate
- 设置GDT
- 设置CR0寄存器中的PE-Bit为1

### A20 Gate

`bootasm.S`中可以看到bootloader首先打开**A20 Gate**。

```assembly
.globl start
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    cld                                             # String operations increment

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    movw %ax, %ds                                   # -> Data Segment
    movw %ax, %es                                   # -> Extra Segment
    movw %ax, %ss                                   # -> Stack Segment

    # Enable A20:
    #  For backwards compatibility with the earliest PCs, physical
    #  address line 20 is tied low, so that addresses higher than
    #  1MB wrap around to zero by default. This code undoes this.
seta20.1:
    inb $0x64, %al                      # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.1

    movb $0xd1, %al                     # 0xd1 -> port 0x64
    outb %al, $0x64                     # 0xd1 means: write data to 8042's P2 port

seta20.2:
    inb $0x64, %al                      # Wait for not busy(8042 input buffer empty).
    testb $0x2, %al
    jnz seta20.2

    movb $0xdf, %al                     # 0xdf -> port 0x60
    outb %al, $0x60            # 0xdf = 11011111, means set P2's A20 bit(the 1 bit) to 1
```

这个A20 Gate属于历史遗留的工程性问题，所以不去深究，了解它关闭了当寻址大于1MB的内存的时候的“Wrap Around”，是进入保护模式的必须步骤就好了。

### GDT

然后是设置GDT。

先看一下Global Descriptor的格式。

![](http://www.independent-software.com/assets/osdev/gdt-descriptor.png)

其中占比最多的**Base**和**Limit**的含义如下：

- **Base address** – This is the address where the block of memory that the descriptor references starts. This is a 32-bit value, sadly spread out over three different areas in the descriptor – but nothing some bitshifting can’t solve. Being a 32-bit value, it’s big enough to indicate any starting address in the 4GB range.
- **Limit** – This is the size of the memory block that the descriptor references. There are only 20 bits available for this value, meaning that we can only address 1 MB of memory. However, we can enable so-called **page granularity ** which results in our value being multiplied by 4,096, which equals once again 4GB. Needless to say, we want this.

然后是**Flags**和**Access Byte**的结构：

![](http://www.independent-software.com/assets/osdev/gdt-descriptor-flags.png)

其中各个位解释如下：

| Bit       | Name                         | Description                                                  |
| :-------- | :--------------------------- | :----------------------------------------------------------- |
| **Pr**    | *Present*                    | Selectors can be marked as “not present” so they can’t be used. Normally, set it to 1. |
| **Privl** | *Privilege level*            | There are four privilege levels of which only levels 0 and 3 are relevant to us. Code running at level 0 (kernel code) has full privileges to all processor instructions, while code with level 3 has access to a limited set (user programs). This is relevant when the memory referenced by the descriptor contains executable code. |
| **Ex**    | *Executable*                 | If set to 1, the contents of the memory area are executable code. If 0, the memory contains data that cannot be executed. |
| **DC**    | *Direction (code segments)*  | A value of 1 indicates that the code can be executed from a lower privilege level. If 0, the code can only be executed from the privilege level indicated in the Privl flag. |
| **DC**    | *Conforming (data segments)* | A value of 1 indicates that the segment grows down, while a value of 0 indicates that it grows up. If a segment grows down, then the offset has to be greater than the base. You would normally set this to 0. |
| **RW**    | *Readable (code segments)*   | If set to 1, then the contents of the memory can be read. It is never allowed to write to a code segment. |
| **RW**    | *Writable (data segments)*   | If set to 1, then the contents of the memory can be written to. It is always allowed to read from a data segment. |
| **Ac**    | *Accessed*                   | The CPU will set this to 1 when the segment is accessed. Initially, set to 0. |
| **Gr**    | *Granularity*                | For a value of 0, the descriptor’s limit is specified in bytes. For a value of 1, it is specified in blocks (pages) of 4 KB. This is what you would normally want if you want to access the full 4GB of memory. |
| **Sz**    | *Size*                       | If set to 0, then the selector defines 16-bit protected mode (80286-style). A value of 1 defines 32-bit protected mode. This is what we want. |

通常需要3个Global Descriptor：

- A NULL-descriptor (an empty descriptor that is required to exist)
- A 4GB code scriptor
- A 4GB data descriptor

根据上面的注释，可以总结出三个Descrpitor的具体信息（摘自参考资料）：

> #### The NULL descriptor
>
> The NULL-descriptor is simply 8 empty bytes. No trick to it.
>
> #### The code descriptor
>
> The code descriptor should be configured like this:
>
> - **Base address** = 0x0
> - **Limit** = 0xffff (with page granularity turned on, this is actually 4GB)
> - Access byte
>   - Present = 1
>   - Privilege level = 0 (privilege level 0 is for kernel code)
>   - Executable = 1 (this is a code segment)
>   - Direction = 0
>   - Readable = 1 Combining all these bits gets us the value 1001 1010b, or 0x9a.
> - Flags
>   - Granularity = 1 (for 4KB pages)
>   - Size = 1 (32-bit style) Combining all these bits gets us the value 1100 1111b, or 0xcf.
>
> #### The data descriptor
>
> The data descriptor should be configured like this:
>
> - **Base address** = 0x0
> - **Limit** = 0xffff (with page granularity turned on, this is actually 4GB)
> - Access byte
>   - Present = 1
>   - Privilege level = 0 (privilege level 0 is for kernel code)
>   - Executable = 0 (this is a data segment)
>   - Conforming = 0
>   - Writable = 1 Combining all these bits get us the value 1001 0010b, or 0x92.
> - Flags
>   - Granularity = 1 (for 4KB pages)
>   - Size = 1 (32-bit style) Combining all these bits get us the value 1100 1111b, or 0xcf.

再来看一下`bootasm.S`和`asm.h`中GDT的定义部分：

- **NULL Descriptor**

  - ```assembly
    #define SEG_NULLASM                                             \
        .word 0, 0;                                                 \
        .byte 0, 0, 0, 0
    ```
  - 整个很简单，全部都是0就是了。

- **Code Descriptor** & **Data descriptor**

  - `asm.h`部分的代码

    ```assembly
    #define SEG_ASM(type,base,lim)                                  \
        .word (((lim) >> 12) & 0xffff), ((base) & 0xffff);          \
        .byte (((base) >> 16) & 0xff), (0x90 | (type)),             \
            (0xC0 | (((lim) >> 28) & 0xf)), (((base) >> 24) & 0xff)
    ```

  - `bootasm.S`部分的代码

    ```assembly
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)     # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)           # data seg for bootloader and kernel
    ```

  - 可见lab1中提供的代码与资料中有些不一致，主要是他将Code Descriptor 和 Data Descriptor 中共有的部分提取出来了：

    - 如二者的`Flags`部分中，`Gr`和`Sz`都是1，所以`Flags`部分的值为`0xc0`，并与`Limit[16:19]`or一下。
    - 如二者的`Access Byte`的`Privl`两位都是`0b01`及有一位都是1，所以`Access Byte`共有部分是`0x90`，故将这个值和另外指定的`type`再or一下。

在`bootasm.S`中设置GDT三个Enrty的完整代码如下：

```assembly
# Bootstrap GDT
.p2align 2                                   # force 4 byte alignment
gdt:
    SEG_NULLASM                              # null seg
    SEG_ASM(STA_X|STA_R, 0x0, 0xffffffff)    # code seg for bootloader and kernel
    SEG_ASM(STA_W, 0x0, 0xffffffff)          # data seg for bootloader and kernel

gdtdesc:
    .word 0x17                               # sizeof(gdt) - 1
    .long gdt                                # address gdt
```

其中`gdtdesc`设置了`gdtdesc`中要求载入GDT的要求的数据结构，然后调用`lgdt gdtdesc`载入GDT，然后通过设置`CR0`寄存器的最低位，即`PE`位，为1，开启保护模式：

```assembly
.set CR0_PE_ON, 0x1

movl %cr0, %eax     
orl $CR0_PE_ON, %eax
movl %eax, %cr0     
```

这时候利用`ljmp`指令重载`CS`为**Code Segment Descriptor**和`EIP`为接下来要执行的x86指令的地址：

```assembly
 ljmp $PROT_MODE_CSEG, $protcseg
```

此时进入32-Bit的Protected Mode，并重载剩下的段寄存器为**Data Segment Descriptor**：

```assembly
.code32                                             # Assemble for 32-bit mode
protcseg:
    # Set up the protected-mode data segment registers
    movw $PROT_MODE_DSEG, %ax                       # Our data segment selector
    movw %ax, %ds                                   # -> DS: Data Segment
    movw %ax, %es                                   # -> ES: Extra Segment
    movw %ax, %fs                                   # -> FS
    movw %ax, %gs                                   # -> GS
    movw %ax, %ss                                   # -> SS: Stack Segment
```

然后设置栈顶寄存器`esp`为`0x7c00`，因为`0x0`~`0x7c00`为stack的地址空间；设置栈基址寄存器`ebp`为`0x0`，并随后调用一条`call`指令，进入`bootmain.c`部分的代码：

```assembly
call bootmain
```

## Exercise 4：分析bootloader加载ELF格式的OS的过程。

### 1. bootloader如何读取硬盘扇区的？

对扇区的读写操作主要有两种方式：

- Programmed-Driven I/O
- Interrupt-Driven I/O

在Lab1的代码中采用了Programmed IO的方式，因为qemu是用IDE挂载的镜像，所以根据第一IDE的PIO端口列表：

| Offset from"I/O" base | Direction | Function                         | Description                                                  | Param. size LBA28/LBA48 |
| --------------------- | --------- | -------------------------------- | ------------------------------------------------------------ | ----------------------- |
| 0                     | R/W       | Data Register                    | Read/Write PIO **data** bytes                                | 16-bit / 16-bit         |
| 1                     | R         | Error Register                   | Used to retrieve any error generated by the last ATA command executed. | 8-bit / 16-bit          |
| 1                     | W         | Features Register                | Used to control command specific interface features.         | 8-bit / 16-bit          |
| 2                     | R/W       | Sector Count Register            | Number of sectors to read/write (0 is a special value).      | 8-bit / 16-bit          |
| 3                     | R/W       | Sector Number Register (LBAlo)   | This is CHS / LBA28 / LBA48 specific.                        | 8-bit / 16-bit          |
| 4                     | R/W       | Cylinder Low Register / (LBAmid) | Partial Disk Sector address.                                 | 8-bit / 16-bit          |
| 5                     | R/W       | Cylinder High Register / (LBAhi) | Partial Disk Sector address.                                 | 8-bit / 16-bit          |
| 6                     | R/W       | Drive / Head Register            | Used to select a drive and/or head. Supports extra address/flag bits. | 8-bit / 8-bit           |
| 7                     | R         | Status Register                  | Used to read the current status.                             | 8-bit / 8-bit           |
| 7                     | W         | Command Register                 | Used to send ATA commands to the device.                     | 8-bit / 8-bit           |

根据这个便可以写出读写**单个**磁盘逻辑块的代码：

```c
/* readsect - read a single sector at @secno into @dst */
static void
readsect(void *dst, uint32_t secno) {
    // wait for disk to be ready
    waitdisk();

    outb(0x1F2, 1);                         // count = 1
    outb(0x1F3, secno & 0xFF);
    outb(0x1F4, (secno >> 8) & 0xFF);
    outb(0x1F5, (secno >> 16) & 0xFF);
    outb(0x1F6, ((secno >> 24) & 0xF) | 0xE0);
    outb(0x1F7, 0x20);                      // cmd 0x20 - read sectors

    // wait for disk to be ready
    waitdisk();

    // read a sector
    insl(0x1F0, dst, SECTSIZE / 4);
}
```

Programmed IO需要写Busy Waiting的代码用来等待磁盘可用，根据`Status Register`的组成，

| Bit  | Abbreviation | Function                                                     |
| ---- | ------------ | ------------------------------------------------------------ |
| 0    | ERR          | Indicates an error occurred. Send a new command to clear it (or nuke it with a Software Reset). |
| 1    | IDX          | Index. Always set to zero.                                   |
| 2    | CORR         | Corrected data. Always set to zero.                          |
| 3    | DRQ          | Set when the drive has PIO data to transfer, or is ready to accept PIO data. |
| 4    | SRV          | Overlapped Mode Service Request.                             |
| 5    | DF           | Drive Fault Error (**does not set ERR**).                    |
| 6    | RDY          | Bit is clear when drive is spun down, or after an error. Set otherwise. |
| 7    | BSY          | Indicates the drive is preparing to send/receive data (wait for it to clear). In case of 'hang' (it never clears), do a software reset. |

磁盘就绪即`RDY`为1，`BSY`为0，所以有如下的代码：

```c
/* waitdisk - wait for disk ready */
static void
waitdisk(void) {
    while ((inb(0x1F7) & 0xC0) != 0x40)
        /* do nothing */;
}
```

然后在`readsect`外面在包装一层就有了从磁盘读写**任意字节**的函数`readseg`

```c
/* *
 * readseg - read @count bytes at @offset from kernel into virtual address @va,
 * might copy more than asked.
 * */
static void
readseg(uintptr_t va, uint32_t count, uint32_t offset) {
    uintptr_t end_va = va + count;

    // round down to sector boundary
    va -= offset % SECTSIZE;

    // translate from bytes to sectors; kernel starts at sector 1
    uint32_t secno = (offset / SECTSIZE) + 1;

    // If this is too slow, we could read lots of sectors at a time.
    // We'd write more to memory than asked, but it doesn't matter --
    // we load in increasing order.
    for (; va < end_va; va += SECTSIZE, secno ++) {
        readsect((void *)va, secno);
    }
}
```

这边可以注意到`readseg`中有一行代码`va -= offset % SECTSIZE;`，是因为磁盘的操作是以块为单位的。

### 2. bootloader是如何加载ELF格式的OS？

ELF Header的格式如下：

| Position (32 bit) | Position (64 bit) | Value                                                 |
| ----------------- | ----------------- | ----------------------------------------------------- |
| 0-3               | 0-3               | Magic number - 0x7F, then 'ELF' in ASCII              |
| 4                 | 4                 | 1 = 32 bit, 2 = 64 bit                                |
| 5                 | 5                 | 1 = little endian, 2 = big endian                     |
| 6                 | 6                 | ELF header version                                    |
| 7                 | 7                 | OS ABI - usually 0 for System V                       |
| 8-15              | 8-15              | Unused/padding                                        |
| 16-17             | 16-17             | 1 = relocatable, 2 = executable, 3 = shared, 4 = core |
| 18-19             | 18-19             | Instruction set - see table below                     |
| 20-23             | 20-23             | ELF Version                                           |
| 24-27             | 24-31             | Program entry position                                |
| 28-31             | 32-39             | Program header table position                         |
| 32-35             | 40-47             | Section header table position                         |
| 36-39             | 48-51             | Flags - architecture dependent; see note below        |
| 40-41             | 52-53             | Header size                                           |
| 42-43             | 54-55             | Size of an entry in the program header table          |
| 44-45             | 56-57             | Number of entries in the program header table         |
| 46-47             | 58-59             | Size of an entry in the section header table          |
| 48-49             | 60-61             | Number of entries in the section header table         |
| 50-51             | 62-63             | Index in section header table with the section names  |

与Lab1中`elf.h`定义的ELF Header大体一致。

```c
#define ELF_MAGIC    0x464C457FU            // "\x7FELF" in little endian

/* file header */
struct elfhdr {
    uint32_t e_magic;     // must equal ELF_MAGIC
    uint8_t e_elf[12];
    uint16_t e_type;      // 1=relocatable, 2=executable, 3=shared object, 4=core image
    uint16_t e_machine;   // 3=x86, 4=68K, etc.
    uint32_t e_version;   // file version, always 1
    uint32_t e_entry;     // entry point if executable
    uint32_t e_phoff;     // file position of program header or 0
    uint32_t e_shoff;     // file position of section header or 0
    uint32_t e_flags;     // architecture-specific flags, usually 0
    uint16_t e_ehsize;    // size of this elf header
    uint16_t e_phentsize; // size of an entry in program header
    uint16_t e_phnum;     // number of entries in program header or 0
    uint16_t e_shentsize; // size of an entry in section header
    uint16_t e_shnum;     // number of entries in section header or 0
    uint16_t e_shstrndx;  // section number that contains section name strings
};
```

加载OS的代码即`bootmain(...)`函数：

```c
/* bootmain - the entry of bootloader */
void
bootmain(void) {
    // read the 1st page off disk
    readseg((uintptr_t)ELFHDR, SECTSIZE * 8, 0);

    // is this a valid ELF?
    if (ELFHDR->e_magic != ELF_MAGIC) {
        goto bad;
    }

    struct proghdr *ph, *eph;

    // load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }

    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();

bad:
    outw(0x8A00, 0x8A00);
    outw(0x8A00, 0x8E00);

    /* do nothing */
    while (1);
}
```

可以看到将kernel载入内存中后，先检查了`ELFHDR->e_magic`是否为ELF的Magic  Number。

然后是根据`ELFHDR->e_phoff`读取了OS文件的Program Header，查阅资料可以知道Program Header的作用：

> An executable or shared object file’s program header table is an array of structures, each describing a segment or other information the system needs to prepare the program for execution.

根据这个可以知道Program Header提供了将文件加载到指定内存位置的信息。

```c
/* program section header */                                                
struct proghdr {                                                            
    uint32_t p_type;   // loadable code or data, dynamic linking info,etc.  
    uint32_t p_offset; // file offset of segment                            
    uint32_t p_va;     // virtual address to map segment                    
    uint32_t p_pa;     // physical address, not used                        
    uint32_t p_filesz; // size of segment in file                           
    uint32_t p_memsz;  // size of segment in memory (bigger if contains bss）
    uint32_t p_flags;  // read/write/execute bits                           
    uint32_t p_align;  // required alignment, invariably hardware page size 
};                                                                          
```

其中我们只要用到`p_offset`、`p_va`、`p_filesz`，所以只要将参照每个Program Header将文件中的每个段载入就好了：

```c
// load each program segment (ignores ph flags)
    ph = (struct proghdr *)((uintptr_t)ELFHDR + ELFHDR->e_phoff);
    eph = ph + ELFHDR->e_phnum;
    for (; ph < eph; ph ++) {
        readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);
    }
```

其中`readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);`个人觉得有误，其中`ph->p_memsz`应该为`ph->p_filesz`，虽然这个错误并不会导致内存错误。

上述的加载完成即程序已准备好运行，此时直接通过`ELFHDR->e_entry`获取Entry Point，通过一条`call`指令，即可进入OS：

```c
((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
```

## Exercise 5：实现函数调用堆栈跟踪函数

这个练习主要是要求了解Stack Frame的结构：

![](https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcTyhiI0TGmT8ONld9YE4WgoHVvr_g8IuJJcRhY5Y_9ioU3KAlkz5Q)

知道这个然后里面Lab1里面已经提供的一些内联汇编函数如`read_ebp()`和`read_eip()`，再根据代码中的提示，就很容易编写：

```c
void
print_stackframe(void) {
/* LAB1 YOUR CODE : STEP 1 */
/* (1) call read_ebp() to get the value of ebp. the type is (uint32_t);
 * (2) call read_eip() to get the value of eip. the type is (uint32_t);
 * (3) from 0 .. STACKFRAME_DEPTH
 *    (3.1) printf value of ebp, eip
 *    (3.2) (uint32_t)calling arguments [0..4] = the contents in address 
 *          (unit32_t)ebp +2 [0..4]
 *    (3.3) cprintf("\n");
 *    (3.4) call print_debuginfo(eip-1) to print the C calling function name 
 *          and line number, etc.
 *    (3.5) popup a calling stackframe
 *           NOTICE: the calling funciton's return addr eip  = ss:[ebp+4]
 *                   the calling funciton's ebp = ss:[ebp]
 */
    uint32_t ebp, eip;
    ebp = read_ebp();
    eip = read_eip();

    typedef struct {
        uint32_t arg1;
        uint32_t arg2;
        uint32_t arg3;
        uint32_t arg4;
    } __args;
    __args *args;

    int i;
    for (i = 0 ; i < STACKFRAME_DEPTH && ebp ; i++) {
        args = ((uint32_t *)ebp) + 2;
        cprintf("ebp:0x%08x eip:0x%08x ", ebp, eip);
        cprintf("arg:0x%08x 0x%08x 0x%08x 0x%08x\n",
        	args->arg1, args->arg2, args->arg3, args->arg4);
        print_debuginfo(eip-1);
        ebp = ( (uint32_t *)ebp )[0];
        eip = ( (uint32_t *)ebp )[1];
    }
}
```

输出如下：

```
ebp:0x00007b38 eip:0x00100bcd arg:0x00010094 0x0010e950 0x00007b68 0x001000a2
    kern/debug/kdebug.c:307: print_stackframe+34                             
ebp:0x00007b48 eip:0x001000a2 arg:0x00000000 0x00000000 0x00000000 0x0010008d
    kern/init/init.c:48: grade_backtrace2+32                                 
ebp:0x00007b68 eip:0x001000d1 arg:0x00000000 0x00007b90 0xffff0000 0x00007b94
    kern/init/init.c:53: grade_backtrace1+37                                 
ebp:0x00007b88 eip:0x001000f8 arg:0x00000000 0xffff0000 0x00007bb4 0x001000e5
    kern/init/init.c:58: grade_backtrace0+29                                 
ebp:0x00007ba8 eip:0x00100124 arg:0x00000000 0x00100000 0xffff0000 0x00100109
    kern/init/init.c:63: grade_backtrace+37                                  
ebp:0x00007bc8 eip:0x00100066 arg:0x00000000 0x00000000 0x00000000 0x00103788
    kern/init/init.c:28: kern_init+101                                       
ebp:0x00007be8 eip:0x00007d6e arg:0x00000000 0x00000000 0x00000000 0x00007c4f
    <unknow>: -- 0x00007d6d --                                               
ebp:0x00007bf8 eip:0x00007c4f arg:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007c4e --                                                 
```

其中最后两行是**<unknow>**，查找`/obj/bootblock.asm`中的`0x00007d6e`和`0x00007c4f`可以看到

```assembly
    ...
    call bootmain
    7c4a:	e8 be 00 00 00       	call   7d0d <bootmain>

00007c4f <spin>:

    # If bootmain returns (it shouldn't), loop.
spin:
    jmp spin
    7c4f:	eb fe                	jmp    7c4f <spin>
    7c51:	8d 76 00             	lea    0x0(%esi),%esi
    ...
```

```assembly
    ....
    // call the entry point from the ELF header
    // note: does not return
    ((void (*)(void))(ELFHDR->e_entry & 0xFFFFFF))();
    7d62:	a1 18 00 01 00       	mov    0x10018,%eax
    7d67:	25 ff ff ff 00       	and    $0xffffff,%eax
    7d6c:	ff d0                	call   *%eax
}

static inline void
outw(uint16_t port, uint16_t data) {
    asm volatile ("outw %0, %1" :: "a" (data), "d" (port));
    7d6e:	ba 00 8a ff ff       	mov    $0xffff8a00,%edx
    7d73:	89 d0                	mov    %edx,%eax
    ...
```

所以可以知道

```
ebp:0x00007be8 eip:0x00007d6e arg:0x00000000 0x00000000 0x00000000 0x00007c4f
    <unknow>: -- 0x00007d6d --   
```

对应于`bootmain`的栈帧。

```
ebp:0x00007bf8 eip:0x00007c4f arg:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007c4e --     
```

对应于`bootasm.S`中`call bootmain`的部分的代码，其中因为这部分代码执行是`esp`指向`0x7c00`，所以`ebp`的值就是`0x7c00-8=0x7bf8`，然后后面跟的`arg`的内容就是从

```assembly
.globl start
start:
.code16                                             # Assemble for 16-bit mode
    cli                                             # Disable interrupts
    7c00:	fa                   	cli    
    cld                                             # String operations increment
    7c01:	fc                   	cld    

    # Set up the important data segment registers (DS, ES, SS).
    xorw %ax, %ax                                   # Segment number zero
    7c02:	31 c0                	xor    %eax,%eax
    movw %ax, %ds                                   # -> Data Segment
    7c04:	8e d8                	mov    %eax,%ds
    movw %ax, %es                                   # -> Extra Segment
    7c06:	8e c0                	mov    %eax,%es
    movw %ax, %ss                                   # -> Stack Segment
    7c08:	8e d0                	mov    %eax,%ss
```

开始的汇编代码对应的二进制数值。

## Exercise 6：完善中断初始化和处理

### 1. 中断向量表中一个表项占多少字节？其中哪几位代表中断处理代码的入口？

<table border="2" cellpadding="4" cellspacing="0" style="margin-top:1em; margin-bottom:1em; background:#f9f9f9; border:1px #aaa solid; border-collapse:collapse; &#123;&#123;&#123;1}}}">
<caption><b>IDT entry, Interrupt Gates</b>
</caption>
<tr>
<th> Name
</th>
<th> Bit
</th>
<th> Full Name
</th>
<th> Description
</th></tr>
<tr>
<th> Offset
</th>
<td> 48..63 </td>
<td> Offset 16..31 </td>
<td> Higher part of the offset.
</td></tr>
<tr>
<th> P
</th>
<td> 47 </td>
<td> Present </td>
<td> Set to <b>0</b> for unused interrupts.
</td></tr>
<tr>
<th> DPL
</th>
<td> 45,46 </td>
<td> Descriptor Privilege Level </td>
<td> Gate call protection. Specifies which privilege Level the calling Descriptor minimum should have. So hardware and CPU interrupts can be protected from being called out of userspace.
</td></tr>
<tr>
<th> S
</th>
<td> 44 </td>
<td> Storage Segment </td>
<td> Set to <b>0</b> for interrupt and trap gates (see below).
</td></tr>
<tr>
<th> Type
</th>
<td> 40..43 </td>
<td> Gate Type 0..3 </td>
<td> Possible IDT gate types&#160;:
<table border="2" cellpadding="4" cellspacing="0" style="margin-top:1em; margin-bottom:1em; background:#f9f9f9; border:1px #aaa solid; border-collapse:collapse; &#123;&#123;&#123;1}}}">
<tr>
<td> 0b0101 </td>
<td> 0x5 </td>
<td> 5 </td>
<td> 80386 32 bit task gate
</td></tr>
<tr>
<td> 0b0110 </td>
<td> 0x6 </td>
<td> 6 </td>
<td> 80286 16-bit interrupt gate
</td></tr>
<tr>
<td> 0b0111 </td>
<td> 0x7 </td>
<td> 7 </td>
<td> 80286 16-bit trap gate
</td></tr>
<tr>
<td> 0b1110 </td>
<td> 0xE </td>
<td> 14 </td>
<td> 80386 32-bit interrupt gate
</td></tr>
<tr>
<td> 0b1111 </td>
<td> 0xF </td>
<td> 15 </td>
<td> 80386 32-bit trap gate
</td></tr></table>
</td></tr>
<tr>
<th> 0
</th>
<td> 32..39 </td>
<td> Unused 0..7 </td>
<td> Have to be <b>0</b>.
</td></tr>
<tr>
<th> Selector
</th>
<td> 16..31 </td>
<td> Selector 0..15 </td>
<td> Selector of the interrupt function (to make sense - the kernel's selector). The selector's descriptor's DPL field has to be <b>0</b> so the <b>iret</b> instruction won't throw a #GP exeption when executed.
</td></tr>
<tr>
<th> Offset
</th>
<td> 0..15 </td>
<td> Offset 0..15 </td>
<td> Lower part of the interrupt function's offset address (also known as pointer).
</td></tr></table>

根据上表首先可以了解到一个Entry的大小是`8 Byte`，且中断处理代码的地址由每个Entry的`[0:15]U[48:63]`位组成的`Offset`决定。

### 2. 请编程完善kern/trap/trap.c 中对中断向量表进行初始化的函数idt_init。在idt_init 函数中，依次对所有中断入口进行初始化。使用mmu.h 中的SETGATE 宏，填充idt 数组内容。注意除了系统调用中断(T_SYSCALL)以外，其它中断均使用中断门描述符，权限为内核态权限；而系统调用中断使用异常，权限为陷阱门描述符。每个中断的入口由
tools/vectors.c 生成，使用trap.c 中声明的vectors 数组即可。

首先在`kern/trap/vector.S`中，可以看到已经声明了`__vectors`

```assembly
 # vector table  
 .data           
 .globl __vectors
 __vectors:      
   .long vector0 
   .long vector1 
   .long vector2 
   .long vector3 
   .long vector4 
   .long vector5 
   .long vector6 
   .long vector7 
   .long vector8 
   .long vector9 
   .long vector10
   ...
```

并且每个`vector xx`都指向处理`Exception`或`trap`的代码。

并且在`kern/trap/trap.c`中调用`lidt`所需的`idt_pt`结构声明如下：

```c
static struct gatedesc idt[256] = {{0}};

static struct pseudodesc idt_pd = {
    sizeof(idt) - 1, (uintptr_t)idt
};
```

所以可以知道应当用`__vectors`中的元素去填充`idt[256]`中每个Entry的`Offset`，填充完调用一个`lidt`指令载入就好了。根据提示，填充`idt[]有`一个Macro：

```c
/* *
 * Set up a normal interrupt/trap gate descriptor
 *   - istrap: 1 for a trap (= exception) gate, 0 for an interrupt gate
 *   - sel: Code segment selector for interrupt/trap handler
 *   - off: Offset in code segment for interrupt/trap handler
 *   - dpl: Descriptor Privilege Level - the privilege level required
 *          for software to invoke this interrupt/trap gate explicitly
 *          using an int instruction.
 * */
#define SETGATE(gate, istrap, sel, off, dpl) {            \
    (gate).gd_off_15_0 = (uint32_t)(off) & 0xffff;        \
    (gate).gd_ss = (sel);                                \
    (gate).gd_args = 0;                                    \
    (gate).gd_rsv1 = 0;                                    \
    (gate).gd_type = (istrap) ? STS_TG32 : STS_IG32;    \
    (gate).gd_s = 0;                                    \
    (gate).gd_dpl = (dpl);                                \
    (gate).gd_p = 1;                                    \
    (gate).gd_off_31_16 = (uint32_t)(off) >> 16;        \
}
```

根据题述，`istrap`全填`0`，`sel`这个参数是要设置为对应的Global Descriptor，可以看到在`kern/mm/pmm.c`中的`pmm_init()`调用的`gdt_init()`中重新设置了`GDT`的地址为下文的`gdt`。

```c
static struct segdesc gdt[] = {
    SEG_NULL,
    [SEG_KTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_KERNEL),
    [SEG_KDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_KERNEL),
    [SEG_UTEXT] = SEG(STA_X | STA_R, 0x0, 0xFFFFFFFF, DPL_USER),
    [SEG_UDATA] = SEG(STA_W, 0x0, 0xFFFFFFFF, DPL_USER),
    [SEG_TSS]    = SEG_NULL,
};
```

在`kern/mm/memlayout.h`中我们可以看到相关的宏定义：

```c
/* global segment number */
#define SEG_KTEXT    1
#define SEG_KDATA    2
#define SEG_UTEXT    3
#define SEG_UDATA    4
#define SEG_TSS        5

/* global descrptor numbers */
#define GD_KTEXT    ((SEG_KTEXT) << 3)        // kernel text
#define GD_KDATA    ((SEG_KDATA) << 3)        // kernel data
#define GD_UTEXT    ((SEG_UTEXT) << 3)        // user text
#define GD_UDATA    ((SEG_UDATA) << 3)        // user data
#define GD_TSS        ((SEG_TSS) << 3)        // task segment selector

#define DPL_KERNEL    (0)
#define DPL_USER    (3)

```

可以其中已经定义了根据每个`segment number * sizeof(segdesc)`得到的每个

Entry的偏移的`GD_XXX`，所以直接用这个宏定义就好了。

```c
void
idt_init(void) {
    extern uintptr_t __vectors[];
    int i;

    for (i=0; i<256; i++)
        SETGATE(idt[i], 0, GD_KTEXT, __vectors[i], DPL_KERNEL);

    lidt(&idt_pd);
}
```

`idt_init()`完整填充完的结果如上。

### 3. 请编程完善trap.c 中的中断处理函数trap，在对时钟中断进行处理的部分填写trap 函数中处理时钟中断的部分，使操作系统每遇到100 次时钟中断后，调用print_ticks 子程序，向屏幕上打印一行文字”100 ticks”。

这题很简单，直接加`ticks`，每次时钟中断检测一下就好了。

```c
static void
trap_dispatch(struct trapframe *tf) {
    char c;

    switch (tf->tf_trapno) {
    case IRQ_OFFSET + IRQ_TIMER:
        ticks ++;
        if (ticks==TICK_NUM) {
            ticks-=TICK_NUM;
            print_ticks();
        }
        break;
    ...
```

`IRQ_TIMER`前面加一个`IRQ_OFFSET`的原因是：在x86中，`IRQ 0`~`IRQ 31`是下表的**Exception**。

| Name                                                         | Vector nr.        | Type       | Mnemonic | Error code? |
| ------------------------------------------------------------ | ----------------- | ---------- | -------- | ----------- |
| [Divide-by-zero Error](https://wiki.osdev.org/Exceptions#Divide-by-zero_Error) | 0 (0x0)           | Fault      | #DE      | No          |
| [Debug](https://wiki.osdev.org/Exceptions#Debug)             | 1 (0x1)           | Fault/Trap | #DB      | No          |
| [Non-maskable Interrupt](https://wiki.osdev.org/Non_Maskable_Interrupt) | 2 (0x2)           | Interrupt  | -        | No          |
| [Breakpoint](https://wiki.osdev.org/Exceptions#Breakpoint)   | 3 (0x3)           | Trap       | #BP      | No          |
| [Overflow](https://wiki.osdev.org/Exceptions#Overflow)       | 4 (0x4)           | Trap       | #OF      | No          |
| [Bound Range Exceeded](https://wiki.osdev.org/Exceptions#Bound_Range_Exceeded) | 5 (0x5)           | Fault      | #BR      | No          |
| [Invalid Opcode](https://wiki.osdev.org/Exceptions#Invalid_Opcode) | 6 (0x6)           | Fault      | #UD      | No          |
| [Device Not Available](https://wiki.osdev.org/Exceptions#Device_Not_Available) | 7 (0x7)           | Fault      | #NM      | No          |
| [Double Fault](https://wiki.osdev.org/Exceptions#Double_Fault) | 8 (0x8)           | Abort      | #DF      | Yes (Zero)  |
| ~~Coprocessor Segment Overrun~~                              | 9 (0x9)           | Fault      | -        | No          |
| [Invalid TSS](https://wiki.osdev.org/Exceptions#Invalid_TSS) | 10 (0xA)          | Fault      | #TS      | Yes         |
| [Segment Not Present](https://wiki.osdev.org/Exceptions#Segment_Not_Present) | 11 (0xB)          | Fault      | #NP      | Yes         |
| [Stack-Segment Fault](https://wiki.osdev.org/Exceptions#Stack-Segment_Fault) | 12 (0xC)          | Fault      | #SS      | Yes         |
| [General Protection Fault](https://wiki.osdev.org/Exceptions#General_Protection_Fault) | 13 (0xD)          | Fault      | #GP      | Yes         |
| [Page Fault](https://wiki.osdev.org/Exceptions#Page_Fault)   | 14 (0xE)          | Fault      | #PF      | Yes         |
| Reserved                                                     | 15 (0xF)          | -          | -        | No          |
| [x87 Floating-Point Exception](https://wiki.osdev.org/Exceptions#x87_Floating-Point_Exception) | 16 (0x10)         | Fault      | #MF      | No          |
| [Alignment Check](https://wiki.osdev.org/Exceptions#Alignment_Check) | 17 (0x11)         | Fault      | #AC      | Yes         |
| [Machine Check](https://wiki.osdev.org/Exceptions#Machine_Check) | 18 (0x12)         | Abort      | #MC      | No          |
| [SIMD Floating-Point Exception](https://wiki.osdev.org/Exceptions#SIMD_Floating-Point_Exception) | 19 (0x13)         | Fault      | #XM/#XF  | No          |
| [Virtualization Exception](https://wiki.osdev.org/Exceptions#Virtualization_Exception) | 20 (0x14)         | Fault      | #VE      | No          |
| Reserved                                                     | 21-29 (0x15-0x1D) | -          | -        | No          |
| [Security Exception](https://wiki.osdev.org/Exceptions#Security_Exception) | 30 (0x1E)         | -          | #SX      | Yes         |
| Reserved                                                     | 31 (0x1F)         | -          | -        | No          |
| [Triple Fault](https://wiki.osdev.org/Exceptions#Triple_Fault) | -                 | -          | -        | No          |
| ~~FPU Error Interrupt~~                                      | IRQ 13            | Interrupt  | #FERR    | No          |

`IRQ 32`~`IRQ 255`才是**User Defined Interrupts**，所以`IRQ_TIMER`前面要加一个`IRQ_OFFSET = 32`。

## 输出大合集：

```
(THU.CST) os is loading ...                                                  
                                                                             
Special kernel symbols:                                                      
  entry  0x00100000 (phys)                                                   
  etext  0x001033e6 (phys)                                                   
  edata  0x0010ea16 (phys)                                                   
  end    0x0010fd20 (phys)                                                   
Kernel executable memory footprint: 64KB                                     
ebp:0x00007b28 eip:0x00100a64 arg:0x00010094 0x00010094 0x00007b58 0x00100092
    kern/debug/kdebug.c:307: print_stackframe+22                             
ebp:0x00007b38 eip:0x00100092 arg:0x00000000 0x00000000 0x00000000 0x00007ba8
   kern/init/init.c:48: grade_backtrace2+33                                  
ebp:0x00007b58 eip:0x001000bc arg:0x00000000 0x00007b80 0xffff0000 0x00007b84
    kern/init/init.c:53: grade_backtrace1+38                                 
ebp:0x00007b78 eip:0x001000db arg:0x00000000 0xffff0000 0x00007ba4 0x00000029
    kern/init/init.c:58: grade_backtrace0+23                                 
ebp:0x00007b98 eip:0x00100101 arg:0x00000000 0x00100000 0xffff0000 0x0000001d
    kern/init/init.c:63: grade_backtrace+34                                  
ebp:0x00007bb8 eip:0x00100055 arg:0x0010341c 0x00103400 0x0000130a 0x00000000
    kern/init/init.c:28: kern_init+84                                        
ebp:0x00007be8 eip:0x00007d72 arg:0x00000000 0x00000000 0x00000000 0x00007c4f
    <unknow>: -- 0x00007d71 --                                               
ebp:0x00007bf8 eip:0x00007c4f arg:0xc031fcfa 0xc08ed88e 0x64e4d08e 0xfa7502a8
    <unknow>: -- 0x00007c4e --
++ setup timer interrupts
100 ticks                
100 ticks                
100 ticks                
```

