---
title: 论文阅读：Towards Transparent CPU Scheduling
date: 2020-08-18 17:31:35
tags: 论文
category: 操作系统
---

最近在看 [OSTEP](http://pages.cs.wisc.edu/~remzi/OSTEP/)，作者在进程调度部分推荐了该论文：

*Joseph T. Meehean*, [Towards Transparent CPU Scheduling](http://www.cs.wisc.edu/adsl/Publications/meehean-thesis11.pdf), UNIVERSITY OF WISCONSIN–MADISON, 2011  

---

本文提出用科学的方法对 CPU 调度器有更深入的了解； 我们使用这种方法来解释和理解 CPU 调度器有时不稳定的行为。这种方法首先将受控的工作负载引入商品操作系统，并观察 CPU 调度器的行为。通过这些观察，我们能够推断底层 CPU 调度策略，并创建预测调度行为的模型。

在将科学分析应用于 CPU 调度器方面，我们已经取得了两项进展。首先是 CPU Futures，这是一个嵌入到 CPU 调度器和用户空间控制器中的预测调度模型的组合，后者使用来自这些模型的反馈来指导应用程序。我们基于两种不同的调度范例(分时和比例-份额)，为两种不同的 Linux 调度器(CFS 和 O(1))开发了这些预测模型。通过三个不同的案例研究，我们演示了应用程序可以使用我们的预测模型将来自低重要性应用程序的干扰减少 70% 以上，将 Web 服务器的供应不足减少一个数量级，并执行与 CPU 调度器相矛盾的调度策略。

我们的第二个贡献是 Harmony。这是一个框架和一组实验，用于从普通操作系统中提取多处理器调度策略。我们使用这个工具提取和分析三个 Linux 调度器的策略：O(1)、CFS 和 BFS。这些调度器通常执行截然不同的策略。从高层次的角度来看，O(1) 调度器谨慎地选择要迁移的进程和强值处理器亲缘性（*原文：strongly values processor affinity*）。相反，CFS 则不断地寻找更好的平衡，最终选择了随机迁移进程。而 BFS 则非常看重公平性，经常忽略处理器关联性。

---

## 导论

> *把自己渴望的东西托付给漫不经心的希望，用至高无上的理由把自己不渴望的东西推到一边，这是人类的习惯。*   
> ————*修西得底斯*  

目前，应用程序、开发人员和系统研究人员都将最佳工作 CPU 调度器视为可靠的黑箱。依赖这些调度器来提供底层但至关重要的服务。在过去的几十年里，处理器快速增长的性能支持了这种依赖和映像。

单处理器速度不断增长的结束意味着这些假设应该重新考虑。如果对中等负载下的CPU调度器进行更仔细的检查，就会发现它们可以提供反复无常的服务，并实现与依赖于它们的应用程序冲突的资源争用策略。这种看似不稳定的行为对应用程序的可感知可靠性有可怕的后果；它会导致缓慢的响应时间，病态的行为，甚至应用程序失败。

在这项工作中，我们建议采取科学的方法来理解CPU调度程序；我们开始通过收集观察、生成假设和生成预测模型来揭开CPU调度器的神秘面纱。例如，对 CPU 调度的进一步研究表明，它有时不稳定的行为并不是不可预测的，而只是几个善意策略的不幸组合结果。这种方法产生的观察结果和预测模型使 CPU 调度对应用程序更加透明(并且可由应用程序控制)，并使研究者们能够改进或细化调度策略。

本文讨论了我们在科学分析 CPU 调度方面所做的两个改进。在第一个 CPU Futures 中，我们创建并演示嵌入到 CPU 调度器中的预测模型的价值。应用程序使用这些预测模型来主动引导两个当代的 Linux CPU 调度程序实现它们所期望的调度策略。在第二部分，我们实现了一个多处理器 CPU 调度观察的收集框架。然后，我们使用这个名为 Harmony 的框架从三个 Linux CPU 调度器(O(1)、CFS 和 BFS)提取 CPU 调度策略。

本章对驱动这些理论的思想进行了高层次的概述。第一部分讨论 CPU 调度日益增加的重要性。下一节将介绍黑箱 CPU 调度造成的困难。在本节之后，将使用科学的方法进行更广泛的讨论，以提高 CPU 调度器的透明度。最后，我们对本文的结构进行了概述。

### CPU 调度的重要性

在过去的 15 年里， CPU 调度算法的进步已经很大程度上与处理器快速增长的速度无关(有些例外[27,48,56,65])。在多媒体应用程序的实时调度领域中，许多有趣的工作只是被新处理器的速度超越了[24,25,26,28]。每一台现代的台式电脑都可以轻松地播放流媒体，而无需专门为此目的而设计的研究技术的帮助。事实上，许多电脑依然在使用 80 年代和 90 年代早期开发的分时调度技术来播放实时媒体。那么，为什么系统研究团体应该投入时间和金钱来研究任务调度呢?

答案是免费的硬件之旅已经结束，单核处理器速度已在 2003 年达到顶峰[63]。当前硬件的趋势是更多的处理器核数，而不是更快的单核处理器，这意味着CPU调度再次重要了起来。为了改进多核处理器下应用程序的性能，需要改进CPU调度算法。

多核处理器不会像更快的单核处理器那样自动地为应用程序提供性能改进。相反，必须重新设计应用程序以增加其并行性。同意，CPU 调度器也必须要重新设计，以最大限度地提高这种新的应用程序并行性的性能。CPU 调度策略(在很大程度上是一种机制)对于在自己的机器上运行的串行应用程序并不重要。然而，现在，一个应用程序可能在一台机器上与自身的多个并发实例竞争/合作。在这个场景中，CPU 调度就变得非常重要。如果应用程序的某些部分没有响应，而另一些部分运行良好，那么应用程序可能会出现无响应、不稳定甚至造成严重故障。

如果 CPU 未被充分利用，CPU 调度几乎无关紧要。事实上，在未充分利用的 CPU 上，调度器的存在只会影响调度延迟。人们最初的期望是，增加的每台机器的处理器数量可以减少潜在的CPU争用，从而减少对良好 CPU 调度的需求。然而，冗余硬件的增加是与服务器整合（*原文：server consolidation*）的增加同时发生的，这重新引入了 CPU 争用的可能性。

为了增加利润并提高(集群的)吞吐量，服务器整合可能会减少应用程序的硬件分配，直到应用程序以接近其分配的硬件容量运行为止。这种对资源分配的精确调整将部分调度问题转移到了应用程序的操作系统上。在接近系统满负荷的情况下运行意味着即使负载稍有增加，也可能使系统处于过载状态。而一旦系统过载， CPU 调度就变得至关重要。因为此时 CPU 调度器必须仔细决定如何分配其有限的资源。

### 不透明的 CPU 调度程序

普通 CPU 调度器非常不透明。现在正在运行的大部分系统中，来自 CPU 调度器的主要反馈只是其策略的直接结果：每个应用程序分配了多少 CPU。虽然总比没有好，但这种反馈传达的信息非常少。它没有提供关于为什么给应用程序一个特定分配的信息。这就是应用程序想要的吗？是否存在 CPU 争用？由于 CPU 争用，应用程序的运行速度会慢多少？如果应用程序有更高或更低的优先级，它会收到什么？

这种不透明性对用户、应用程序、应用程序开发人员和系统研究人员产生了负面影响。用户和应用程序无法确定他们的程序运行将怎样被 CPU 调度和争用所影响。对于运行缓慢的应用程序，用户甚至可能根本无法判断这是 CPU 本身的锅还是 CPU 调度器的锅。应用程序无法确定系统是否能够支持其当前的并行度级别，应用程序无法确定是否应该增加或减少并行度，更不要说确定增加或减少多少。这些问题常常使 CPU 调度器显得不可预测和不一致。

对 CPU 调度器的离线分析证明，它的不透明性既有所增加，亦有所减少。一般操作系统的单处理器调度策略通常有很好的文档，主要在教科书[20,35,88,89,108,111]中。然而，这些操作系统的多处理器调度策略常常缺少文档记录或者根本没有文档记录；发行主发行版 Linux 调度器时，没有任何关于其多处理器调度策略的文档。此外，常用的单处理器和多处理器调度策略的文档常常以实现为重点。这让读者大致了解了如何构建调度程序，但无法了解在给定特定工作负载时它的行为。

应用程序开发人员无法在其应用程序中有效地构建并行性，因为他们无法预测 CPU 调度器将如何管理这种并行性。相反，他们必须使用试验和错误来设计他们的应用程序。这种不确定性在实现结束时增加了一个额外的递归调试步骤，而不是允许应用程序设计人员基于很容易理解的 CPU 调度器参数构建应用程序。

最后，如果系统研究人员对当前技术没有一个坚实的了解，那么调度改进是困难的。有限的商品调度文件意味着每个研究人员必须从头开始发展这一理解。此外，如果没有有用的运行时反馈，用户和应用程序开发人员就无法提供关于他们使用普通 CPU 调度器时遇到的问题的详细说明。这些限制使得系统研究人员很难更改 CPU 调度器以更接近应用程序和用户的需求。

CPU 调度器的不透明性质让用户、开发人员和系统研究人员只能猜测系统失败的原因以及如何改进它们。这种“粗心的希望”是好的科学和工程的对立面。

### 增加 CPU 调度的透明度

要解决不透明调度器问题，不能简单地公开它们的内部工作，而是要形成对它们行为的深入理解，进而产生更有价值的信息。例如，“给用户空间展示 CPU 调度器所调度的运行队列”，这种就没有什么用，因为这种轻微的透明度（*原文： micro-transparency*）只能传递“接下来会运行哪个进程”这种没有什么用的信息。至于“各个进程能获得多少空间分配”和“队列里的最后一个进程要排多久的队”，进程则一无所知。要是进程修改了它自己的优先级或行为，进程就更不可能知道这些指标将如何变化。

我们认为，CPU 调度程序应该像研究自然系统一样：运用科学方法。第一步是观察给定各种工作负载(刺激)时调度器的行为。接下来，我们对观察到的行为做出假设。然后，使用这些假设作为基础，我们生成预测模型。最后，我们将预测模型的结果与所观察的系统进行比较，并完善我们的假设和模型。在此分析中创建的假设和模型不仅使我们能够预测CPU调度器的行为，还能使我们对 CPU 调度策略的行为有了基本的理解。这种理解和预测能力将把 CPU 调度器从难以理解的黑盒子转变为可理解的透明系统。

重要的是，我们不仅要了解 CPU 调度器实现的策略，还要了解该策略对实际工作负载的影响。策略通常是根据设计师关于如何满足高级目标的直觉创建的。为了理解调度策略，在某些情况下，我们必须从观察到的 CPU 调度程序的行为中反向工程高级目标。然后我们可以分析该策略满足目标的程度。分析这项政策可能产生的任何副作用也是至关重要的。这也是基于对 CPU 调度器行为的观察。一旦我们有了更深层次的理解，我们就可以开始提前预测给定特定工作负载的调度器的行为。

由这些观察和预测模型所产生的增加的透明性为系统研究人员和应用程序开发人员提供了提高性能和可靠性的能力。系统研究人员可以使用观察和假设来提出对调度政策的改进。应用程序开发人员可以使用预测模型来指导其应用程序的开发；开发人员可以将他们的应用程序转向 CPU 调度器中工作最好的部分，同时尝试减轻调度器的缺点。

在运行系统中嵌入调度模型可以为用户和应用程序提供非常需要的反馈。用户可以确切地知道由于 CPU 争用或选择的 CPU 调度器不佳而导致的性能下降。系统管理员可以将此信息用于性能调试或资源规划。应用程序也可以使用此反馈来密切监视其并发工作流的性能，并确保每个工作流都取得了足够的进展。然后，应用程序可以使用有关每个工作流的相对重要性的特定于应用程序的逻辑来解决由 CPU 争用引起的问题。

我们对 CPU 调度的科学分析做出了两个贡献：CPU Futures 和 Harmony。CPU Futures 是嵌入到 CPU 调度器中的一组预测模型和一个用户空间控制器的组合，用于使用来自这些模型的反馈来引导应用程序。我们已经为两种一般类型的 CPU 调度器(分时和比例共享)创建了这些预测调度模型，并为两种 Linux 调度器(CFS 和 O(1))实现了它们。将这些模型与一个简单的用户空间控制器结合起来，我们将展示它们对于分布式应用程序和低重要性的后台应用程序的价值。

Harmony 是一个为 CPU 调度器生成刺激(合成工作负载)并观察结果行为的实验框架。这个框架只需要少量的低级插装，并且不依赖于操作系统文档或源代码。我们还设计了一组实验来从商用 CPU 调度器中提取多处理器调度策略。通过这些实验，我们演示了 Harmony 的价值，并开始说明三个 Linux 调度器的调度策略：O(1)、CFS 和 BFS。

### 本文结构

在接下来的两章中，我们将提供 CPU 调度的背景资料。在第 2 章中，我们讨论了 CPU 调度器和多处理器调度架构的流行类型。第 3 章提供了对不透明 CPU 调度器问题的更深入的分析，包括激励实验。

再往后的两章介绍了 CPU Futures，这是一组 CPU 调度器的预测模型，它允许应用程序执行它们自己的调度策略。第 4 章概述了 CPU Futures，并详细讨论了调度模型。在第 5 章中，我们用三个案例来说明如何使用 CPU Futures 反馈，并说明嵌入式调度模型的有用性。

Harmony 将在那之后的两章中介绍。第 6 章概述了 Harmony ，以及使用 Harmony 从三个 Linux 调度器(O(1)、CFS 和 BFS)提取基本策略的结果。第 7 章则从这三个调度程序中引出更复杂的策略。

在最后两章，我们讨论了本论文的背景和贡献。第 8 章介绍与 Harmony 及 CPU Futures 有关的工作。在第 9 章中，我们讨论了我们的结论并总结了这项工作的贡献。

## CPU 调度

> Some have two feet  
> And some have four.  
> Some have six feet  
> And some have more.  
> — Dr. Seuss (One Fish, Two Fish, Red Fish, Blue Fish)  

CPU 调度器的目标是提供一种假象，即每个进程或线程都有自己的专用 CPU。虚拟化 CPU 所需的机制相当简单。创建一个策略来在竞争的进程和线程之间划分物理 CPU 是一个困难得多的问题。详细地理解这个问题对于理解提高 CPU 调度器透明度的重要性至关重要，无论是通过改进的接口还是经验观察。

CPU 调度没有计划；没有最佳的解决方案。相反，CPU调度是关于平衡目标和做出困难的权衡。确定 CPU 调度器的底层目标是了解其复杂行为的关键。了解调度器为实现其目标所做的权衡对于理解商用调度器的发展是至关重要的。新的调度器要么强调不同的目标，要么提供一种更简单的方法来实现与其前身相同的目标。理解商品调度程序是改进它们的第一步。