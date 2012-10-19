---
layout: post
title: "Windows设定线程运行在指定CPU(CORE)?"
---

众所周知的，Windows下可以分别使用下列函数来设置进程或者线程的关联性，也即控制线程运行在哪些特定CPU上。

SetProcessAffinityMask(GetCurrentProcess(), 0x00000001);
SetThreadAffinityMask(GetCurrentThread(), 0x00000001);

请参考MSDN[SetThreadAffinityMask function](http://msdn.microsoft.com/en-us/library/windows/desktop/ms686247(v=vs.85).aspx)来查看其详细的使用说明。

但我疑惑的是，既然MSDN上说：大多数情形下，让系统选择处理器来运行线程或进程更好，原文:
>Setting an affinity mask for a process or thread can result in threads receiving less processor time, as the system is restricted from running the threads on certain processors. In most cases, it is better to let the system select an available processor.

那么，合适的使用时机是什么呢？

在《Windows 核心编程》(第5版)中提到，若要检查每个CPU是否存在Pentium浮点Bug的话，这时这些函数是个合适的应用场景。

在进行高精度时间计算时，也需要用到这个东西，具体请参看[Game Timing and Multicore Processors](http://msdn.microsoft.com/en-us/library/ee417693(v=vs.85).aspx)

或许你还有其他的使用时机，请回复我，谢谢了!