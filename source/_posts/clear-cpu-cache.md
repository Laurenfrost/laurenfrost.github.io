---
title: 思考：如何清掉CPU的cache
date: 2020-07-20 20:35:14
tags: cache
category: 系统结构
---

bangumi的三一大佬最近在研究计算机系统结构，他今天突然在群里提出了一个问题：linux有命令能刷新CPU cache吗？  
课本里都讲了，CPU的cache是对用户应该是透明的。换句话说，CPU的cache从来是自己独立运行，没有办法直接控制的。所以不存在一种命令直接刷新cache。  
但真的一点办法都没有吗？  

## 思路一：大量访存  
cache嘛，众所周知，帮助CPU访存的东西。CPU需要什么，cache就帮你从内存里抓过来。无论cache是用哪一种算法实现的，它总会有一个更新cache内容的机制，把用过了的数据丢回内存，从而腾出空间存放新的数据。既然如此，那么我们就故意申请一个大于cache大小的空间，然后把它们挨个访问一遍。这样不就能实现“刷新”cache了吗。  

通过CPU（而非DMA）反复读取大量数据：[2]  
```c++
int main() {
    const int size = 20*1024*1024; // Allocate 20M. Set much larger than L2
    char *c = (char *)malloc(size);
    for (int i = 0; i < 0xffff; i++)
        for (int j = 0; j < size; j++)
            c[j] = i*j;
}
```

但这又存在新的问题：  
1. 绝大多数的现代CPU有两个L1 cache：data cache和instruction cache。这种大量访存的方式只能清除L1的data cache，无法清除L1的instruction cache。  
2. 因为不知道CPU内部的具体实现方式，所以无法保证CPU会把cache里的所有旧数据全部替换掉。如果上述程序所访问的数据只在cache的一个section里打转，那么就根本算不上“清除”了cache。  
3. 最致命的一点就是：这种方式与随便找一堆代码执行一下又有什么分别呢？  

*我还有一些别的想法：现在多核CPU基本上L1和L2是每个核独占，而L3则是同一个numa结点内的所有核共享，所以这种方式是绝对无法做到“清除”L3缓存的。*

## 思路二：干等着  
```bash
#!/usr/bin/ruby
puts "main:"
200000.times { puts "  nop" }
puts "  xor rax, rax"
puts "  ret"
```
Running a few times under different names (code produced not the script) should do the work  [2]

## 思路三：特殊的CPU指令  
经过查阅资料，发现了一个有趣的事情，现代CPU提供了一种能直接作用于cache line的指令：`CLFLUSH`（即flush cache line）。  
***所以严格来说，能直接接触CPU cache的方法是存在的***  

> There are x86 assembly instructions to force the CPU to flush certain cache lines (such as CLFLUSH), but they are pretty obscure. CLFLUSH in particular only flushes a chosen address from L1 caches.  

> The CLFLUSH instruction does not flush only the L1 cache. From the Intel x86-64 reference manual: "The CLFLUSH (flush cache line) instruction writes and invalidates the cache line associated with a specified linear address. The invalidation is for all levels of the processor’s cache hierarchy, and it is broadcast throughout the cache coherency domain."  [1]

另外似乎还存在指令`wbinvd`和`invd`，使指定cache line的数据变成invalid。

> Fortunately, there is more than one way to explicitly flush the caches.

> The instruction "wbinvd" writes back modified cache content and marks the caches empty. It executes a bus cycle to make external caches flush their data. Unfortunately, it is a privileged instruction. But if it is possible to run the test program under something like DOS, this is the way to go. This has the advantage of keeping the cache footprint of the "OS" very small.

> Additionally, there is the "invd" instruction, which invalidates caches without flushing them back to main memory. This violates the coherency of main memory and cache, so you have to take care of that by yourself. Not really recommended.

> For benchmarking purposes, the simplest solution is probably copying a large memory block to a region marked with WC (write combining) instead of WB. The memory mapped region of the graphics card is a good candidate, or you can mark a region as WC by yourself via the MTRR registers.

> You can find some resources about benchmarking short routines at [Test programs for measuring clock cycles and performance monitoring](https://www.agner.org/optimize/).[1]

关于这些指令的讨论可以参考[4]。  
更完整的论述可以参考[3]。

## 参考内容  
stack overflow:  
1. [How can I do a CPU cache flush in x86 Windows?](https://stackoverflow.com/questions/1756825/how-can-i-do-a-cpu-cache-flush-in-x86-windows)  
2. [How to clear CPU L1 and L2 cache \[duplicate\]
](https://stackoverflow.com/questions/3446138/how-to-clear-cpu-l1-and-l2-cache)  
3. [Is there a way to flush the entire CPU cache related to a program?](https://stackoverflow.com/questions/48527189/is-there-a-way-to-flush-the-entire-cpu-cache-related-to-a-program)  
4. [WBINVD instruction usage](https://stackoverflow.com/questions/6745665/wbinvd-instruction-usage/6745706#6745706)  
