---
title: Windows 10 的热点与 Gradle 冲突
date: 2021-03-03 18：52：00
tags: Gradle
category: 日常踩坑
---

> 你的主机中的软件中止了一个已建立的连接。

构建gradle出现错误提示：你的主机中的软件中止了一个已建立的连接。反复折腾也不得要领，最后看到别人提示要关闭 Windows 10 的热点。我一试，果然好使。

这就很奇怪。而且如果每次运行 Gradle 都要关闭网络共享的话，未免也太不方便了。

在 Gradle 的 GitHub repo 里发现了相关的 issue。其中有人提到：

> I investigated a little more and it appears that after connecting a device to the mobile hotspot, even a ping 127.0.0.1 on the console does not work for me. This is very strange behaviour on Microsoft's part, and very likely the reason for this bug, as @kriegaex pointed out a commit with the addition of a check if the loopback interface address is reachable. Accessing services started on 127.0.0.1 is possible though (e.g. curl http://127.0.0.1 works).
I wasn't able to find any resources on this behaviour, or even how the hotspot feature exactly works.

所以这个本质上是 Windows 10 自己的毛病，然后还一直没修。太可恶了。

其中还有人提到，如果将 Gradle 版本降低到 6.4.0 及以下，问题或能改善。