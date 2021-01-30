# Ucore Lab 4

## Source Code Analysis

这个实验不是非常复杂，好像源代码没啥很难懂的地方。

相对于前面的3个Lab，这个Lab在`kern_init`中多调用了一个`proc_init`。`proc_init`代码如下：

```c
 // proc_init - set up the first kernel thread idleproc "idle" by itself and
 //           - create the second kernel thread init_main
 void
 proc_init(void) {
     int i;

     list_init(&proc_list);
     for (i = 0; i < HASH_LIST_SIZE; i ++) {
         list_init(hash_list + i);
     }

     if ((idleproc = alloc_proc()) == NULL) {
         panic("cannot alloc idleproc.\n");
     }

     idleproc->pid = 0;
     idleproc->state = PROC_RUNNABLE;
     idleproc->kstack = (uintptr_t)bootstack;
     idleproc->need_resched = 1;
     set_proc_name(idleproc, "idle");
     nr_process ++;

     current = idleproc;

     int pid = kernel_thread(init_main, "Hello world!!", 0);
     if (pid <= 0) {
         panic("create init_main failed.\n");
     }

     initproc = find_proc(pid);
     set_proc_name(initproc, "init");

     assert(idleproc != NULL && idleproc->pid == 0);
     assert(initproc != NULL && initproc->pid == 1);
 }
```

可以看到`proc_init`先是初始化了初始化了两个表，一个是所有进程的双向链表`proc_list`，一个是进程PID的哈希表，注意`hash_list`是数组。

然后将当前执行的context初始化为`idle_proc`，并利用`kernel_thread`创建了一个内核线程，追踪一下