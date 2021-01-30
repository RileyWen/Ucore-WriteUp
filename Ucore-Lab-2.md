# Ucore-Lab 2

## Analysis of Ucore Lab 2 Source Code

Ucore的文档里面已经详细覆盖了所需要的基础知识，这边就再大致串一下和Ucore Lab1中不一样的地方和大致的启动流程。

### Flow Chart of Booting & Kernel Init & VM Init

```flow
start=>start: BIOS Init
step1=>operation: Preparation_Code ! boot/bootasm.S
step2=>operation: bootmain ! boot/bootmain.c
step3=>operation: kern_entry ! kern/init/entry.S
step4=>operation: kern_init ! kern/init/init.c
step5=>operation: pmm_init ! kern/mm/pmm.c
step6=>operation: page_init ! kern/mm/pmm.c

start->step1->step2->step3->step4->step5->step6
```

### Difference between `bootasm.S` in Lab1 and `bootasm.S` in Lab2

相对于Lab1中的`bootasm.S`，Lab2中增加了探测内存的代码

```assembly
probe_memory:          
    movl $0, 0x8000    
    xorl %ebx, %ebx    
    movw $0x8004, %di  
start_probe:           
    movl $0xE820, %eax 
    movl $20, %ecx     
    movl $SMAP, %edx   
    int $0x15          
    jnc cont           
    movw $12345, 0x8000
    jmp finish_probe   
cont:                  
    addw $20, %di      
    incl 0x8000        
    cmpl $0, %ebx      
    jnz start_probe    
finish_probe:
	...
```

这部分代码将探测到的内存写入`0x8000`，定义如下的数据结构中

```c
struct e820map {                           
    int nr_map;                            
    struct {                               
        uint64_t addr;                     
        uint64_t size;                     
        uint32_t type;                     
    } __attribute__((packed)) map[E820MAX];
};                                         
```

根据`make qemu-nox`的输出，我们可以知道探测到的内存信息为

```
e820map:                                           
  memory: 0009fc00, [00000000, 0009fbff], type = 1.
  memory: 00000400, [0009fc00, 0009ffff], type = 2.
  memory: 00010000, [000f0000, 000fffff], type = 2.
  memory: 07ee0000, [00100000, 07fdffff], type = 1.
  memory: 00020000, [07fe0000, 07ffffff], type = 2.
  memory: 00040000, [fffc0000, ffffffff], type = 2.
```

### Difference between `entry.S` in Lab1 and `entry.S` in Lab2

和Lab1中`bootmain.c`在bootloader结束的时候直接跳转到`init.c`中的`kern_init`不一样，Lab2中先跳转到了一个新增文件`entry.S` 中的`kern_entry`:

```assembly
#include <mmu.h>
#include <memlayout.h>

#define REALLOC(x) (x - KERNBASE)

.text
.globl kern_entry
kern_entry:
    # load pa of boot pgdir
    movl $REALLOC(__boot_pgdir), %eax
    movl %eax, %cr3

    # enable paging
    movl %cr0, %eax
    orl $(CR0_PE | CR0_PG | CR0_AM | CR0_WP | CR0_NE | CR0_TS | CR0_EM | CR0_MP), %eax
    andl $~(CR0_TS | CR0_EM), %eax
    movl %eax, %cr0

    # update eip
    # now, eip = 0x1.....
    leal next, %eax
    # set eip = KERNBASE + 0x1.....
    jmp *%eax
next:

    # unmap va 0 ~ 4M, it's temporary mapping
    xorl %eax, %eax
    movl %eax, __boot_pgdir

    # set ebp, esp
    movl $0x0, %ebp
    # the kernel stack region is from bootstack -- bootstacktop,
    # the kernel stack size is KSTACKSIZE (8KB)defined in memlayout.h
    movl $bootstacktop, %esp
    # now kernel stack is ready , call the first C function
    call kern_init

# should never get here
spin:
    jmp spin

.data
.align PGSIZE
    .globl bootstack
bootstack:
    .space KSTACKSIZE
    .globl bootstacktop
bootstacktop:

# kernel builtin pgdir
# an initial page directory (Page Directory Table, PDT)
# These page directory table and page table can be reused!
.section .data.pgdir
.align PGSIZE
__boot_pgdir:
.globl __boot_pgdir
    # map va 0 ~ 4M to pa 0 ~ 4M (temporary)
    .long REALLOC(__boot_pt1) + (PTE_P | PTE_U | PTE_W)
    .space (KERNBASE >> PGSHIFT >> 10 << 2) - (. - __boot_pgdir) # pad to PDE of KERNBASE
    # map va KERNBASE + (0 ~ 4M) to pa 0 ~ 4M
    .long REALLOC(__boot_pt1) + (PTE_P | PTE_U | PTE_W)
    .space PGSIZE - (. - __boot_pgdir) # pad to PGSIZE

.set i, 0
__boot_pt1:
.rept 1024
    .long i * PGSIZE + (PTE_P | PTE_W)
    .set i, i + 1
.endr
```

下图是页表有关信息的结构。

![1557674578108](Ucore-Lab-2.assets\1557674578108.png)

结合上图，可以看到这一段代码在进入`kern_init`之前，先设置了临时页表，将原来`Virtual Address = Linear Address = Physical Address`的映射关系更改为如下的临时映射：

```
# 线性地址在0~4MB之内三者的映射关系
Virtual Address = Linear Addrress = Physical Address 

# 线性地址在0xC0000000~0xC0000000+4MB=0xc0400000之内三者的映射关系
Virtual Address = Linear Addrress = Physical Address + 0xC0000000 
```

然后将当前`eip`的值通过

```assembly
# update eip
# now, eip = 0x1.....
leal next, %eax
# set eip = KERNBASE + 0x1.....
jmp *%eax
```

切换到高位的虚拟地址，这样就消除了对虚拟地址空间的低位部分的引用。这样就可以消除

```
# 线性地址在0~4MB之内三者的映射关系
Virtual Address = Linear Addrress = Physical Address 
```

这个映射了，这通过直接将`__boot_pgdir[0]`置零就可以了。

然后跳转进入真正的`kern_init`。

在老版本的Ucore中，这个临时映射是通过修改段选择子达到的：

```assembly
#include <mmu.h>
#include <memlayout.h>

#define REALLOC(x) (x - KERNBASE)

.text
.globl kern_entry
kern_entry:
    # reload temperate gdt (second time) to remap all physical memory
    # virtual_addr 0~4G=linear_addr&physical_addr -KERNBASE~4G-KERNBASE 
    lgdt REALLOC(__gdtdesc)
    movl $KERNEL_DS, %eax
    movw %ax, %ds
    movw %ax, %es
    movw %ax, %ss

    ljmp $KERNEL_CS, $relocated

relocated:

    # set ebp, esp
    movl $0x0, %ebp
    # the kernel stack region is from bootstack -- bootstacktop,
    # the kernel stack size is KSTACKSIZE (8KB)defined in memlayout.h
    movl $bootstacktop, %esp
    # now kernel stack is ready , call the first C function
    call kern_init

# should never get here
spin:
    jmp spin

.data
.align PGSIZE
    .globl bootstack
bootstack:
    .space KSTACKSIZE
    .globl bootstacktop
bootstacktop:

.align 4
__gdt:
    SEG_NULL
    SEG_ASM(STA_X | STA_R, - KERNBASE, 0xFFFFFFFF)      # code segment
    SEG_ASM(STA_W, - KERNBASE, 0xFFFFFFFF)              # data segment
__gdtdesc:
    .word 0x17                                          # sizeof(__gdt) - 1
    .long REALLOC(__gdt)
```

虽然老版本的方法代码精简一点，但是还是觉得新版本的代码更优雅一点，因为PDE和PTE是可以重用的。

### `pmm_init` & `page_init` in `pmm.c` & `default_pmm.c`

对于页表的初始化主要是在`pmm_init`调用的`page_init`中完成的。

`page_init`的完整代码如下：

```c
/* pmm_init - initialize the physical memory management */
static void
page_init(void) {
    struct e820map *memmap = (struct e820map *)(0x8000 + KERNBASE);
    uint64_t maxpa = 0;

    cprintf("e820map:\n");
    int i;
    for (i = 0; i < memmap->nr_map; i ++) {
        uint64_t begin = memmap->map[i].addr, end = begin + memmap->map[i].size;
        cprintf("  memory: %08llx, [%08llx, %08llx], type = %d.\n",
                memmap->map[i].size, begin, end - 1, memmap->map[i].type);
        if (memmap->map[i].type == E820_ARM) {
            if (maxpa < end && begin < KMEMSIZE) {
                maxpa = end;
            }
        }
    }
    if (maxpa > KMEMSIZE) {
        maxpa = KMEMSIZE;
    }

    extern char end[];

    npage = maxpa / PGSIZE;
    pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);

    for (i = 0; i < npage; i ++) {
        //cprintf("[pages+%d] addr:0x%08x\n", i, pages+i);
        SetPageReserved(pages + i);
    }

    uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * npage);

    for (i = 0; i < memmap->nr_map; i ++) {
        uint64_t begin = memmap->map[i].addr, end = begin + memmap->map[i].size;
        if (memmap->map[i].type == E820_ARM) {
            if (begin < freemem) {
                begin = freemem;
            }
            if (end > KMEMSIZE) {
                end = KMEMSIZE;
            }
            if (begin < end) {
                begin = ROUNDUP(begin, PGSIZE);
                end = ROUNDDOWN(end, PGSIZE);
                if (begin < end) {
                    init_memmap(pa2page(begin), (end - begin) / PGSIZE);
                }
            }
        }
    }
}
```

#### 物理内存空间初始化

ucore以`4KB`作为页框的大小对内存进行管理，页框信息的数据结构定义如下：

```c
struct Page {
    int ref;               // page frame's reference counter
    uint32_t flags;        // array of flags that describe the status of the page frame
    unsigned int property; // the num of free block, used in first fit pm manager
    list_entry_t page_link;// free list lin
};
```

首先可以看到下面的代码先将ucore的BSS段后的第一个2的n次方的地址作为页框信息数组的首地址，然后将所有页框设置为`Reserved`状态。

```c
npage = maxpa / PGSIZE;
pages = (struct Page *)ROUNDUP((void *)end, PGSIZE);

for (i = 0; i < npage; i ++) {
    //cprintf("[pages+%d] addr:0x%08x\n", i, pages+i);
    SetPageReserved(pages + i);
}
```

然后接下来对空闲内存进行初始化：

```c
uintptr_t freemem = PADDR((uintptr_t)pages + sizeof(struct Page) * npage);

for (i = 0; i < memmap->nr_map; i ++) {
    uint64_t begin = memmap->map[i].addr, end = begin + memmap->map[i].size;
    if (memmap->map[i].type == E820_ARM) {
        if (begin < freemem) {
            begin = freemem;
        }
        if (end > KMEMSIZE) {
            end = KMEMSIZE;
        }
        if (begin < end) {
            begin = ROUNDUP(begin, PGSIZE);
            end = ROUNDDOWN(end, PGSIZE);
            if (begin < end) {
                init_memmap(pa2page(begin), (end - begin) / PGSIZE);
            }
        }
    }
}
```

其中

- `end`是ucore的BSS段结束的地址。
- `freemem`是空闲的物理内存空间的起始地址，前面加了`PADDR`是因为内核链接的地址是`0xc0100000`，但由于`bootasm.S`中的`readseg(ph->p_va & 0xFFFFFF, ph->p_memsz, ph->p_offset);`装载内核的时候，`ph->p_va`&了一个`0xFFFFFF`，所以实际上内核被装载到了`0x00100000`。所以真正空闲的物理内存空间的起始地址应当是在kernel的代码中计算完的虚拟地址再减掉一个`KERNBASE=0xC0000000`。

### `le2page`的神秘实现

这个根据`list_entry_t`数据结构反推外部数据结构的代码过于骚气。。真的过于骚气。。感受一下

```c
// convert list entry to page                
#define le2page(le, member)                 \
    to_struct((le), struct Page, member)     

/* *                                                                
 * to_struct - get the struct from a ptr                            
 * @ptr:    a struct pointer of member                              
 * @type:   the type of the struct this is embedded in              
 * @member: the name of the member within the struct                
 * */                                                               
#define to_struct(ptr, type, member)                               \
    ((type *)((char *)(ptr) - offsetof(type, member)))              

/* Return the offset of 'member' relative to the beginning of a struct type */ 
#define offsetof(type, member)                                      \          
    ((size_t)(&((type *)0)->member))                                           
```

rbq，rbq

## Exercise 1 - 实现 first-fit 连续物理内存分配算法（需要编程）

### Description

在实现first fit 内存分配算法的回收函数时，要考虑地址连续的空闲块之间的合并操作。提示:在建立空闲页块链表时，需要按照空闲页块起始地址来排序，形成一个有序的链表。可能会修改`default_pmm.c`中的`default_init`，`default_init_memmap`，`default_alloc_pages`， `default_free_pages`等相关函数。请仔细查看和理解`default_pmm.c`中的注释。

请在实验报告中简要说明你的设计实现过程。

### Solution

下面所有代码中的注释说明了每一步的意义。

#### `default_init_memmap`

```c
static void
default_init_memmap(struct Page *base, size_t n) {
    assert(n > 0);
    struct Page *p = base;
    for (; p != base + n; p ++) {
        assert(PageReserved(p));

        p->flags = p->property = 0;
        set_page_ref(p, 0);
    }
    base->property = n;
    SetPageProperty(base);
    nr_free += n;

    // Add to "TAIL" of free_list
    list_add_before(&free_list, &(base->page_link));
}
```

#### `default_alloc_pages`

```c
static struct Page *
default_alloc_pages(size_t n) {
    assert(n > 0);
    if (n > nr_free) {
        return NULL;
    }

    struct Page *page = NULL, *p_temp;

    list_entry_t *le = &free_list;
    while ((le = list_next(le)) != &free_list) {
        // This 'le2page' macro seems like black magic :)
        p_temp = le2page(le, page_link);

        // If a proper "free-page list" that
        // has >=n free page is found, return it
        if (p_temp->property >= n) {
            page = p_temp;
            break;
        }
    }

    if (page != NULL) {
        // if the length of this free-page list > n,
        // we need to truncate it to 'page->property - n'
        if (page->property > n) {
            p_temp = page + n;
            p_temp->property = page->property - n;

            SetPageProperty(p_temp);
            list_add(&(page->page_link), &(p_temp->page_link));
        }
        // if the length of this free-page list is exactly n,
        // we only need to remove it from free-page list and
        // clear PG_PROPERTY bit
        list_del(&(page->page_link));
        ClearPageProperty(page);

        // subtract n from # of available pages
        nr_free -= n;
    }
    return page;
}
```

#### `default_free_pages`

```c
static void
default_free_pages(struct Page *base, size_t n) {
    assert(n > 0);

    struct Page *p_temp = base;
    for (; p_temp != base + n; p_temp ++) {
        assert(!PageReserved(p_temp) && !PageProperty(p_temp));

        // Clear PG_PROPERTY bit to indicate it is usable
        p_temp->flags = 0;

        // Clear ref count
        set_page_ref(p_temp, 0);
    }
    base->property = n;
    SetPageProperty(base);

    list_entry_t *le = list_next(&free_list);
    while (le != &free_list) {
        p_temp = le2page(le, page_link);
        le = list_next(le);

        // If preceding free-page list can merge with me
        if (base + base->property == p_temp) {
            base->property += p_temp->property;
            ClearPageProperty(p_temp);
            list_del(&(p_temp->page_link));
        }
        // If following free-page list can merge with me
        else if (p_temp + p_temp->property == base) {
            p_temp->property += base->property;
            ClearPageProperty(base);
            base = p_temp;
            list_del(&(p_temp->page_link));
        }
    }
    nr_free += n;

    // Insert this block to proper position
    // We want to keep the ascending order of free list
    le = list_next(&free_list);
    while (le != &free_list) {
        p_temp = le2page(le, page_link);

        // If proper position is found
        if (base + base->property <= p_temp) {
             assert(base + base->property != p_temp);

             break;
        }

        // This line cannot be moved before `break;`
        le = list_next(le);
    }
    list_add_before(le, &(base->page_link));
}
```

#### 思考题

##### 你的first fit算法是否有进一步的改进空间

如果有，也只有改进搜索的时间复杂度了。

## Exercise 2 - 实现寻找虚拟地址对应的页表项（需要编程）

### Description

通过设置页表和对应的页表项，可建立虚拟内存地址和物理内存地址的对应关系。其中的`get_pte`函数是设置页表项环节中的一个重要步骤。此函数找到一个虚地址对应的二级页表项的内核虚地址，如果此二级页表项不存在，则分配一个包含此项的二级页表。本练习需要补全`get_pte`函数 in `kern/mm/pmm.c`，实现其功能。请仔细查看和理解`get_pte`函数中的注释。`get_pte`函数的调用关系图如下所示：

[![img](https://github.com/chyyuu/ucore_os_docs/raw/master/lab2_figs/image001.png)](https://github.com/chyyuu/ucore_os_docs/blob/master/lab2_figs/image001.png) 图1 `get_pte`函数的调用关系图

请在实验报告中简要说明你的设计实现过程。

### Solution

#### `get_pte`

```c
//get_pte - get pte and return the kernel virtual address of this pte for la
//        - if the PT contians this pte didn't exist, alloc a page for PT
// parameter:
//  pgdir:  the kernel virtual base address of PDT
//  la:     the linear address need to map
//  create: a logical value to decide if alloc a page for PT
// return vaule: the kernel virtual address of this pte
pte_t *
get_pte(pde_t *pgdir, uintptr_t la, bool create) {
#if 0
    pde_t *pdep = NULL;   // (1) find page directory entry
    if (0) {              // (2) check if entry is not present
                      // (3) check if creating is needed, then alloc page for page table
                  // CAUTION: this page is used for page table, not for common data page
                          // (4) set page reference
        uintptr_t pa = 0; // (5) get linear address of page
                          // (6) clear page content using memset
                          // (7) set page directory entry's permission
    }
    return NULL;          // (8) return page table entry
#endif
    pde_t *pdep = &pgdir[PDX(la)];
    if ( !(*pdep & PTE_P) ) {
        struct Page *pt_temp;

        if (!create || (pt_temp = alloc_page()) == NULL)
            return NULL;

        set_page_ref(pt_temp, 1);

        uintptr_t pt_pa = page2pa(pt_temp);
        memset(KADDR(pt_pa), 0, PGSIZE);

        *pdep = pt_pa | PTE_U | PTE_P | PTE_W;
    }
    return &((pte_t *)KADDR(PDE_ADDR(*pdep)))[PTX(la)];
}
```

这题照着注释写就好了。

大概思路就是如果`*pdep`如果`Present bit == 0`，则说明这个PTE所属的Page Table也不存在，而正好`sizeof(Page Table) = 4KB`，即一个Page的大小，所以需要先用`alloc_page()`分配一个Page再返回对应PTE的地址。如果`Present bit == 1`，就直接返回PTE的地址就好了。

这题还用到了实验指导书中提及的**自映射机制**，详见实验指导书。

#### 思考题

##### 请描述页目录项（Page Directory Entry）和页表项（Page Table Entry）中每个组成部分的含义以及对ucore而言的潜在用处。

![1557674578108](Ucore-Lab-2.assets\1557674578108.png)

还是这张图再贴一遍，Ucore中使用到的几个位就是`Bit 0`、`Bit 1`、`Bit 2`。对于PTE和PDE的详细注解如下，这在*Intel® 64 and IA-32 Architectures Software Developer Manuals*中可以找到。

![1557837958893](Ucore-Lab-2.assets\1557837958893.png)

![1557838126280](Ucore-Lab-2.assets\1557838126280.png)

潜在用处？我看不出来。。。

##### 如果ucore执行过程中访问内存，出现了页访问异常，请问硬件要做哪些事情？

首先CPU会产生一个中断，把引起Page Fault的Linear Address装到CR2寄存器中，将`CS`、`EIP`、`EFLAGS`、`Error Code`等压入内核栈，并根据TSS中的信息切换`SS`和`ESP`，根据IDT跳转到对应的ISR，在Ucore中，最终是跳转到`kern/trap/trap.c`中的`trap_dispatch`这个函数。

然后在`trap_dispatch`这个函数中，switch到`T_PGFLT = 14`这个值，在这里面处理Page Fault相关的处理操作。

## Exercise 3 - 释放某虚地址所在的页并取消对应二级页表项的映射（需要编程）

### Description

当释放一个包含某虚地址的物理内存页时，需要让对应此物理内存页的管理数据结构Page做相关的清除处理，使得此物理内存页成为空闲；另外还需把表示虚地址与物理地址对应关系的二级页表项清除。请仔细查看和理解`page_remove_pte`函数中的注释。为此，需要补全在 `kern/mm/pmm.c`中的`page_remove_pte`函数。`page_remove_pte`函数的调用关系图如下所示：

[![img](https://github.com/chyyuu/ucore_os_docs/raw/master/lab2_figs/image002.png)](https://github.com/chyyuu/ucore_os_docs/blob/master/lab2_figs/image002.png)

图2 page_remove_pte函数的调用关系图

请在实验报告中简要说明你的设计实现过程。

### Solution

#### `page_remove_pte`

```c
//page_remove_pte - free an Page sturct which is related linear address la
//                - and clean(invalidate) pte which is related linear address la
//note: PT is changed, so the TLB need to be invalidate
static inline void
page_remove_pte(pde_t *pgdir, uintptr_t la, pte_t *ptep) {
#if 0
    if (0) {                      //(1) check if this page table entry is present
        struct Page *page = NULL; //(2) find corresponding page to pte
                                  //(3) decrease page reference
                                  //(4) and free this page when page reference reachs 0
                                  //(5) clear second page table entry
                                  //(6) flush tlb
    }
#endif
    if ( *ptep & PTE_P ) {
        struct Page *p = pte2page(*ptep);

        if (page_ref_dec(p) == 0)
            free_page(p);

        *ptep = 0;

        tlb_invalidate(pgdir, la);
    }
}
```

这题照着注释写就好了。

#### 思考题

##### 数据结构Page的全局变量（其实是一个数组）的每一项与页表中的页目录项和页表项有无对应关系？如果有，其对应关系是啥？

有。页目录项或页表项的高20位表示它对应的是哪个Page。

##### 如果希望虚拟地址与物理地址相等，则需要如何修改lab2，完成此事？ 鼓励通过编程来具体完成这个问题。

前文的[Difference between `entry.S` in Lab1 and `entry.S` in Lab2](#Difference between `entry.S` in Lab1 and `entry.S` in Lab2)中可以看到

```assembly
.section .data.pgdir
.align PGSIZE
__boot_pgdir:
.globl __boot_pgdir
    # map va 0 ~ 4M to pa 0 ~ 4M (temporary)
    .long REALLOC(__boot_pt1) + (PTE_P | PTE_U | PTE_W)
    .space (KERNBASE >> PGSHIFT >> 10 << 2) - (. - __boot_pgdir) # pad to PDE of KERNBASE
    # map va KERNBASE + (0 ~ 4M) to pa 0 ~ 4M
    .long REALLOC(__boot_pt1) + (PTE_P | PTE_U | PTE_W)
    .space PGSIZE - (. - __boot_pgdir) # pad to PGSIZE
```

在`pmm_init`中也可以看到`boot_map_segment(boot_pgdir, KERNBASE, KMEMSIZE, 0, PTE_W);`

说明虚拟地址和物理地址的偏移是通过`KERNBASE`来定义的。即如果我们希望虚拟地址与物理地址相等，只需要修改`kern/mm/memlayout.h`中的`#define KERNBASE 0xC0000000` 为`#define KERNBASE 0x00000000` 即可。