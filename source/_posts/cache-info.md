---
title: 如何在Linux系统中查询cache信息
date: 2020-04-28 13:43:23
tags: cache
category: 系统结构
---

<div align = "center">
<img src = "MemoryHierarchy.png" width = 50% height = 50% alt = "内存层次模型">
<p>内存分层模型</p>
<p><i>Computer Architecture A Quantitative Approach (6th Edition)</i></p>
</div>

## 使用`getconf`命令来查询cache信息
`getconf`命令可以查询计算机硬件的很多信息，这其中就包括了cache，因此我们可以使用如下的命令来获取cache相关信息：  
```bash
$ getconf -a | grep CACHE
```
在我的电脑里获取到了如下信息： 
*CPU: AMD Ryzen 7 2700*   
```
LEVEL1_ICACHE_SIZE                 65536
LEVEL1_ICACHE_ASSOC                4
LEVEL1_ICACHE_LINESIZE             64
LEVEL1_DCACHE_SIZE                 32768
LEVEL1_DCACHE_ASSOC                8
LEVEL1_DCACHE_LINESIZE             64
LEVEL2_CACHE_SIZE                  524288
LEVEL2_CACHE_ASSOC                 8
LEVEL2_CACHE_LINESIZE              64
LEVEL3_CACHE_SIZE                  16777216
LEVEL3_CACHE_ASSOC                 16
LEVEL3_CACHE_LINESIZE              64
LEVEL4_CACHE_SIZE                  0
LEVEL4_CACHE_ASSOC                 0
LEVEL4_CACHE_LINESIZE              0
```
1. 这些数据的单位均为字节（Byte）。  
2. 一级缓存分“指令缓存”和“数据缓存”两种，分别用`ICACHE`和`DCACHE`来表示。  
3. `SIZE`指的是该级缓存的总大小。
4. `LINESIZE`指的是该级缓存cache行的大小。即在内存层次模型（Memory Hierarchy）中，该级cache向低一级cache访问时，一次性抓取的数据量。
5. `ASSOC`指的是该级缓存组相联的组数。

---

## 通过系统自带的库来获取cache信息  
头文件`unistd.h`封装了大量针对系统调用的API，可以藉此获取相应的信息：  
```c
#include <stdio.h>
#include <unistd.h>
 
int main (void)
{
  long l1_cache_line_size = sysconf(_SC_LEVEL1_DCACHE_LINESIZE);
  long l2_cache_line_size = sysconf(_SC_LEVEL2_CACHE_LINESIZE); 
  long l3_cache_line_size = sysconf(_SC_LEVEL3_CACHE_LINESIZE);
 
  printf("L1 Cache Line Size is %ld bytes.\n", l1_cache_line_size); 
  printf("L2 Cache Line Size is %ld bytes.\n", l2_cache_line_size); 
  printf("L3 Cache Line Size is %ld bytes.\n", l3_cache_line_size); 
 
  return (0);
}
```
`gcc`编译后运行可得如下信息：  
```
$ vim cache-info.c
$ gcc cache-info.c
$ ./a.out
L1 Cache Line Size is 64 bytes.
L2 Cache Line Size is 64 bytes.
L3 Cache Line Size is 64 bytes.
```

---

## 通过不同系统相应的文件来获取cache信息
参考[https://stackoverflow.com/questions/794632/programmatically-get-the-cache-line-size]  
```c
#ifndef GET_CACHE_LINE_SIZE_H_INCLUDED
#define GET_CACHE_LINE_SIZE_H_INCLUDED

#include <stddef.h>
size_t cache_line_size();

#if defined(__APPLE__)

#include <sys/sysctl.h>
size_t cache_line_size() {
    size_t line_size = 0;
    size_t sizeof_line_size = sizeof(line_size);
    sysctlbyname("hw.cachelinesize", &line_size, &sizeof_line_size, 0, 0);
    return line_size;
}

#elif defined(_WIN32)

#include <stdlib.h>
#include <windows.h>
size_t cache_line_size() {
    size_t line_size = 0;
    DWORD buffer_size = 0;
    DWORD i = 0;
    SYSTEM_LOGICAL_PROCESSOR_INFORMATION * buffer = 0;

    GetLogicalProcessorInformation(0, &buffer_size);
    buffer = (SYSTEM_LOGICAL_PROCESSOR_INFORMATION *)malloc(buffer_size);
    GetLogicalProcessorInformation(&buffer[0], &buffer_size);

    for (i = 0; i != buffer_size / sizeof(SYSTEM_LOGICAL_PROCESSOR_INFORMATION); ++i) {
        if (buffer[i].Relationship == RelationCache && buffer[i].Cache.Level == 1) {
            line_size = buffer[i].Cache.LineSize;
            break;
        }
    }

    free(buffer);
    return line_size;
}

#elif defined(linux)

#include <stdio.h>
size_t cache_line_size() {
    FILE * p = 0;
    p = fopen("/sys/devices/system/cpu/cpu0/cache/index0/coherency_line_size", "r");
    unsigned int i = 0;
    if (p) {
        fscanf(p, "%d", &i);
        fclose(p);
    }
    return i;
}

#else
	#error Unrecognized platform
	#endif
#endif
```
