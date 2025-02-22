chapter4练习
============================================

- 本节难度： **有一定困难，尽早开始** 


本章任务
-------------------------------------------

- ``make test BASE=1``
- 理解 vm.c 中的几个函数的大致功能，通过 bin_loader 理解当前用户程序的虚存布局。
- 结合课堂内容，完成本章问答作业。
- 完成本章编程作业。
- 最终，完成实验报告并 push 你的 ch4 分支到远程仓库。

编程作业
---------------------------------------------

重新实现 sys_gettimeofday
++++++++++++++++++++++++++++++++++++++++++++

引入虚存机制后，原来内核的 sys_gettimeofday 函数实现就无效了。请你重写这个函数，恢复其正常功能。

代码中已经为你预留了函数，你需要填写 ``YOUR CODE`` 部分的代码。

完成后你应该能够正确执行 ch3b_sleep* 对应的测例。通过 ``make test CHAPTER=4_3 BASE=1`` 来测试你的实现。

tips:

- 抄框架其他传指针的 syscall 实现。

mmap 匿名映射
++++++++++++++++++++++++++++++++++++++++++++

你有没有想过，当你在 C 语言中写下的 ``new int[100];`` 执行时可能会发生哪些事情？你可能已经发现，目前我们给用户程序的内存都是固定的并没有增长的能力，这些程序是不能执行 ``new`` 这类导致内存使用增加的操作。libc 中通过 `sbrk <https://linux.die.net/man/2/sbrk>`_ 系统调用增加进程可使用的堆空间，这也是本来的题目设计，但是一位热心的往年助教J学长表示：这一点也不酷！他推荐了另一个申请内存的系统调用。

`mmap <https://man7.org/linux/man-pages/man2/mmap.2.html>`_ 本身主要使用来在内存中映射文件的，这里我们简化它的功能，仅仅使用匿名映射。

mmap 系统调用新定义：

- syscall ID：222
- 接口： ``int mmap(void* start, unsigned long long len, int port， int flag, int fd)``
- 功能：申请长度为 len 字节的匿名物理内存（不要求实际物理内存位置，可以随便找一块），并映射到 addr 开始的虚存，内存页属性为 port。
- 参数：
    - start：需要映射的虚存起始地址。
    - len：映射字节长度，可以为 0 （如果是则直接返回），不可过大(上限 1GiB )。
    - port：第 0 位表示是否可读，第 1 位表示是否可写，第 2 位表示是否可执行。其他位无效（必须为 0 ）。
    - flag：目前始终为 0，忽略该参数。
    - fd：目前始终为 0, 忽略该参数。
- 返回值:
    - 成功返回 0，错误返回 -1。
- 说明：
    - 为了简单，addr 要求按页对齐(否则报错)，len 可直接按页上取整。
    - 为了简单，不考虑分配失败时的页回收。
    - flag, fd 参数留待后续实验拓展。
- 错误：
    - [addr, addr + len) 存在已经被映射的页。
    - 物理内存不足。
    - port & ~0x7 == 0，port 其他位必须为 0
    - port & 0x7 != 0，不可读不可写不可执行的内存无意义

munmap 系统调用新定义：

- syscall ID：215
- 接口： ``int munmap(void* start, unsigned long long len)``
- 功能：取消一块虚存的映射。
- 参数：同 mmap
- 说明：
    - 为了简单，参数错误时不考虑内存的恢复和回收。
- 错误：
    - [start, start + len) 中存在未被映射的虚存。


正确实现后，你的 os 应该能够正确运行 ch4_* 对应的一些测试用例，``make test BASE=0`` 来执行测试。

tips:

- 匿名映射的页可以使用 kalloc() 得到。
- 注意 kalloc 不支持连续物理内存分配，所以你必须把多个页的 mmap 逐页进行映射。
- 一定要注意 mmap 是的页表项，注意 riscv 页表项的格式与 port 的区别。
- 你增加 PTE_U 了吗？


问答作业
-------------------------------------------------

1. 请列举 SV39 页表页表项的组成，结合课堂内容，描述其中的标志位有何作用／潜在作用？

2. 缺页

    这次的实验没有涉及到缺页有点遗憾，主要是缺页难以测试，而且更多的是一种优化，不符合这次实验的核心理念，所以这里补两道小题。

    缺页指的是进程访问页面时页面不在页表中或在页表中无效的现象，此时 MMU 将会返回一个中断，告知 os 进程内存访问出了问题。os 选择填补页表并重新执行异常指令或者杀死进程。

    - 请问哪些异常可能是缺页导致的？
    - 发生缺页时，描述相关的重要寄存器的值（lab2中描述过的可以简单点）。

    缺页有两个常见的原因，其一是 Lazy 策略，也就是直到内存页面被访问才实际进行页表操作。比如，一个程序被执行时，进程的代码段理论上需要从磁盘加载到内存。但是 os 并不会马上这样做，而是会保存 .text 段在磁盘的位置信息，在这些代码第一次被执行时才完成从磁盘的加载操作。

    - 这样做有哪些好处？

    此外 COW(Copy On Write) 也是常见的容易导致缺页的 Lazy 策略，这个之后再说。其实，我们的 mmap 也可以采取 Lazy 策略，比如：一个用户进程先后申请了 10G 的内存空间，然后用了其中 1M 就直接退出了。按照现在的做法，我们显然亏大了，进行了很多没有意义的页表操作。

    - 请问处理 10G 连续的内存页面，需要操作的页表实际大致占用多少内存(给出数量级即可)？
    - 请简单思考如何才能在现有框架基础上实现 Lazy 策略，缺页时又如何处理？描述合理即可，不需要考虑实现。

    缺页的另一个常见原因是 swap 策略，也就是内存页面可能被换到磁盘上了，导致对应页面失效。

    - 此时页面失效如何表现在页表项(PTE)上？

3. 双页表与单页表

   为了防范侧信道攻击，我们的 os 使用了双页表。但是传统的设计一直是单页表的，也就是说，用户线程和对应的内核线程共用同一张页表，只不过内核对应的地址只允许在内核态访问。请结合课堂知识回答如下问题：(备注：这里的单/双的说法仅为自创的通俗说法，并无这个名词概念，详情见 `KPTI <https://en.wikipedia.org/wiki/Kernel_page-table_isolation>`_ )

   - 如何更换页表？
   - 单页表情况下，如何控制用户态无法访问内核页面？（tips:看看上一题最后一问）
   - 单页表有何优势？（回答合理即可）
   - 双页表实现下，何时需要更换页表？假设你写一个单页表操作系统，你会选择何时更换页表（回答合理即可）？

报告要求
--------------------------------------------------------
- 注意目录要求，报告命名 ``lab2.md``（或 pdf），位于 ``reports`` 目录下。命名错误视作没有提交。不需要删除 ``lab1.md``。后续实验同理。
- 简单总结本次实验你新添加的代码。
- 完成 ch4 问答作业。
- [可选，不占分]你对本次实验设计及难度的看法。
   