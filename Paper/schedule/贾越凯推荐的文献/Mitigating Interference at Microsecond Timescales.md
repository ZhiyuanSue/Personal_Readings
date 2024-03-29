为了实现任务之间的性能隔离，常规智慧认为需要对CPU资源（如核心、缓存和内存带宽）进行分区。现代CPU广泛提供缓存分区功能，并推荐将延迟敏感的应用程序固定在专用核心上，这证明了这种观点的正确性。

本文指出资源分配既非必要也非充分。许多应用程序经历突发请求模式或分阶段行为，从而大大改变它们所需的资源类型和数量。然而，基于分区的系统无法快速反应以跟上这些变化，导致延迟极高和失去提高CPU利用率的机会。

Caladan是一种新的CPU调度器，通过快速核心分配而不是资源分区，通过一系列控制信号和策略，可以实现显着更好的服务质量（尾延迟、吞吐量等）。Caladan由一个集中式调度器核心和一个内核模块组成，绕过标准的Linux内核调度器，支持微秒级任务监控和放置。当将memcached与最佳努力的垃圾回收工作负载放置在一起时，Caladan比最先进的资源分区系统Parties表现更好，将尾延迟从580毫秒降低到52微秒，同时保持高CPU利用率。

1 Introduction

数据密集型的网络服务需要在成千上万台服务器之间分配请求，减少尾延迟对于这些服务至关重要。然而，减少尾延迟必须与最大化数据中心效率的需求进行平衡。在共享资源争用高的情况下，任务之间会发生干扰，导致延迟显著增加。

为了管理干扰，开发了几种硬件机制来分配资源，例如Intel的Cache Allocation Technology（CAT）使用基于路的缓存分区来为特定核心保留LLC的部分。许多系统使用这些分区机制来提高性能隔离，但它们要么静态分配足够的资源以应对峰值负载，要么在数百毫秒到几秒钟内进行动态调整。每次调整都是增量的，因此在资源使用发生变化后收敛到正确的配置可能需要几十秒钟。

实际工作负载的资源使用会在更短的时间内发生变化，例如网络流量和垃圾回收等任务会在短时间内消耗大量资源，现有系统无法检测和应对这种突发情况。

我们的目标是在资源使用和干扰频繁变化的实际条件下，保持高CPU利用率和严格的性能隔离（吞吐量和尾延迟）。实现微秒级的反应时间有两个挑战：一是在微秒级时间尺度上准确检测各种干扰的控制信号很困难，二是现有系统在收集控制信号或快速调整资源分配方面存在过多的软件开销。

Caladan是一个干扰感知的CPU调度器，由**一个集中的调度器核心和一个名为KSCHED的Linux内核模块**组成。调度器核心区分高优先级的延迟关键任务和低优先级的尽力而为任务。为了避免硬件分区带来的反应时间限制，Caladan仅依靠核心分配来管理干扰。

Caladan使用精选的控制信号和相应的操作，以微秒级时间尺度快速准确地检测和响应干扰。干扰有两个相互关联的影响：首先，干扰会减慢核心的执行速度（更多的缓存未命中，更高的内存延迟等），影响请求的服务时间；其次，随着核心减速，计算能力下降，当它低于提供的负载时，排队延迟会急剧增加。

PAGE 3

Caladan调度器的目标是收集内存带宽使用和请求处理时间的细粒度测量数据，用于检测内存带宽和超线程干扰。然后，它限制与BE任务相冲突的核心，消除对服务时间的大部分影响。对于LLC干扰，Caladan无法直接消除服务时间开销，但它仍然可以通过允许LC任务从BE任务中窃取额外的核心来防止计算能力下降。

KSCHED内核模块通过分摊发送中断的成本、将调度工作从调度器核心转移到任务核心上，并提供非阻塞API来处理多个正在进行的操作，加速调度操作，收集干扰度指标。这些技术消除了调度瓶颈，使Caladan能够快速响应并在重度干扰下扩展到多个核心和任务。

Caladan是第一个能在频繁变化的干扰和负载下保持严格性能隔离和高CPU利用率的系统。为了实现这些好处，Caladan对应用程序提出了两个新要求：采用自定义运行时系统进行调度和需要LC任务公开其内部并发性。Caladan能够比现有的资源分区系统快500,000倍收敛到正确的资源配置，这导致尾延迟降低了11,000倍。此外，Caladan非常通用，能够扩展到多个任务，并在放置各种工作负载（如memcached、内存数据库、闪存存储服务、x264视频编码器、垃圾收集器等）时保持相同的好处。Caladan可在https://github.com/shenango/caladan上获得。

2 Motivation

当干扰没有被迅速缓解时，性能会下降。许多工作负载表现出分阶段行为，会在亚秒级时间尺度上大幅改变它们使用的资源类型和数量。请求率也可能在微秒级时间尺度上迅速变化，这些负载突发可能会导致资源使用的突发。这些突发会导致干扰的突然增加，从而降低请求服务时间并导致请求队列增长。

本文探讨了时变干扰的挑战，以LC任务memcached和BE工作负载为例，由于垃圾回收的相位行为，导致了问题的出现。使用Boehm GC和更复杂的增量GC，如Go语言运行时，也会出现类似的问题。

本实验固定负载，静态分配核心，使memcached的99.9%尾延迟保持在50微秒以下。当GC运行时，分配不足以保护memcached的尾延迟，GC暂停BE任务100-200毫秒，扫描整个堆，导致memcached请求的缓存未命中率增加，内存访问延迟增加，导致memcached服务请求的速率下降，队列增加，队列延迟每10微秒增加5微秒，最终达到比正常高1000倍的尾延迟。

固定的核心分配不足以有效减轻干扰，需要快速重新分配核心。如果干扰可以瞬间减少请求服务速率一半，为了防止延迟增加X，CPU调度器必须在2X内检测和响应干扰。现有系统无法如此快速响应，导致数据中心运营商要么容忍严重的尾延迟峰值，要么将这些任务隔离在不同的服务器上。

3 Background

PAGE 4

本文讨论了共享CPU时可能出现的三种干扰形式：**超线程干扰、内存带宽干扰和LLC干扰**。超线程干扰通常在运行任务时存在，但具体严重程度取决于共享资源的争用情况。内存带宽和LLC干扰会影响共享同一物理CPU的所有核心。随着内存带宽使用量的增加，访问延迟会逐渐增加，直到内存带宽饱和；此时访问延迟呈指数级增长。LLC干扰取决于每个应用程序使用的缓存量：当需求超过容量时，LLC缺失率会增加。

本节讨论现有系统无法处理干扰突变的原因（§3.1），并探讨商用CPU硬件扩展对其施加的限制（§3.2）。

3.1 Existing Approaches to Interference

Heracles和Parties等现代系统通过动态分配资源来处理干扰，但它们的决策和资源分配调整速度过慢，无法应对突发干扰。这是因为它们使用应用程序级别的尾延迟测量来检测干扰，需要数百毫秒才能获得稳定的结果，并且它们缺乏直接识别干扰源的能力，**因此调整过程需要进行大量试错，需要几秒钟才能收敛到新的资源分配**。在调整期间，延迟通常会继续受到影响。

Heracles和Parties在适应干扰变化方面所需的时间是我们示例中GC周期的50倍以上。因此，操作员必须根据可调参数进行权衡：要么将尾延迟容忍度设置得更高，以容忍GC干扰而不进行资源重新分配，要么将其设置得更低，导致持续限制GC工作负载。由于GC工作负载在大部分执行过程中（非垃圾收集时）干扰很小，需要更快的反应时间来保持核心忙碌而不影响尾延迟。

现有系统在收敛速度和可扩展性方面存在问题。Heracles只能处理一个LC任务，而Parties虽然可以支持多个LC和BE任务，但随着任务增加，其收敛时间也会增加。此外，数据中心服务器需要同时处理多个LC和BE任务，但现有系统存在限制。

这些系统依赖的硬件机制也存在限制。例如，超线程缺乏资源分配控制，因此Heracles和Parties完全关闭了它们。同时使用一个核上的两个超线程可以比使用单个超线程提高30%的吞吐量，因此这显著降低了系统吞吐量。此外，可控制的可用硬件分区机制限制了反应速度和可扩展性。下面我们将讨论这个问题。

3.2 Limitations of Hardware Extensions

英特尔为其服务器CPU添加了几个扩展，旨在分区和监视LLC和内存带宽。这些扩展针对资源需求变化缓慢的情况进行了优化，但在GC工作负载的研究中，这种假设并不总是成立。为了更好地了解这些限制，我们详细讨论了每个组件。

CAT是一种常用的技术，可以将LLC分配给不同的任务以提高性能确定性。但是，CAT的硬件实现存在两个限制：配置更改需要时间才能生效，而且分区数量有限，会降低关联性。KPart通过将互补任务分组到同一分区中来避免这个问题，但需要频繁的在线分析，导致尾延迟高。

PAGE 5

MBA是一种扩展，用于限制DRAM访问的带宽消耗。它适用于静态分配核心的系统，因为它是唯一可以用来限制带宽消耗的方法。然而，MBA与我们实现高CPU利用率的目标相冲突：被MBA严重限制的核心将花费大部分时间处于停顿状态。相反，我们发现分配较少的核心更有效，可以实现相同的任务吞吐量，但每个核心的利用率更高。

配置分区机制需要将资源使用归属于特定任务。英特尔引入了缓存监控技术和内存带宽监控技术来帮助实现这一目标，但这些机制无法快速检测系统条件的变化。例如，使用缓存监控技术监测流媒体任务需要112毫秒才能稳定测量其缓存占用。类似地，实验发现内存带宽监控技术需要毫秒级才能准确估计内存带宽使用情况。

4 Challenges and Approach

我们的目标是在最大化CPU利用率的同时保持性能隔离。实现这个目标很困难，因为管理干扰变化需要微秒级的反应时间。硬件分区资源的速度对于这些时间尺度来说太慢了，所以Caladan的方法是通过控制核心分配给任务来管理干扰。以前的系统在管理干扰时调整核心作为其策略的一部分，但Caladan是第一个完全依靠核心分配来管理多种形式干扰的系统。为了快速减轻干扰，我们必须克服两个关键挑战。

为了快速和有针对性地反应，Caladan需要能够在微秒级别内识别干扰和其来源的控制信号。常用的性能指标如CPI或尾延迟等过于嘈杂，无法在短时间内使用。队列延迟等指标可以在微秒级别内测量，但无法确定干扰的来源。

现有系统依赖于Linux内核来收集控制信号并调整资源分配。然而，Linux会增加这些操作的开销，并且在干扰和核心数量增加时，这些开销会增加。

我们通过精选控制信号并专门分配一个核心来监测和减轻干扰，解决了敏感性的挑战。我们通过一个名为KSCHED的Linux内核模块解决了可扩展性的挑战。

4.1 Caladan ’s Approach

Caladan使用调度程序来检测干扰并调整核心分配，以管理多种形式的干扰。对于超线程，它专注于减少长时间运行请求的干扰。对于内存带宽，它测量全局内存带宽使用情况以检测DRAM饱和，并测量每个核心的LLC缺失率以将使用归因于特定任务。通过关注干扰驱动的控制信号，Caladan可以在服务质量降低之前检测到问题。

Caladan采取了多种措施来减少干扰，包括控制逻辑核心的使用、限制任务使用的核心数量以减少内存带宽干扰、给受害任务分配额外的核心以弥补干扰造成的计算能力损失。这些措施可以减少服务时间的增加和队列延迟。然而，减少LLC干扰更加困难，因为LLC干扰的大小主要取决于任务使用的LLC容量。

PAGE 6

Caladan推出了一个名为KSCHED的Linux内核模块，它能够在微秒级的时间内跨多个核心执行调度功能，即使在存在干扰的情况下也能实现。KSCHED通过三种主要技术实现这些目标：（1）它在Caladan管理的所有核心上运行，并将调度工作从调度器核心转移到运行任务的核心上；（2）它利用硬件支持的多播处理器间中断（IPI）来分摊同时在多个核心上启动操作的成本；（3）它提供了一个完全异步的调度器接口，以便调度器可以在远程核心上启动操作并在等待其完成时执行其他工作。

5 Design

5.1 Overview

Caladan和Shenango共享一些架构和实现构建块，每个应用程序都与运行时系统链接，并且有一个专用的调度器核心。这两个系统都设计为在正常的Linux环境中相互操作，可能管理可用核心的子集。

Caladan和Shenango在调度机制上有不同，Caladan使用多个控制信号来管理干扰和负载变化，而Shenango只使用排队延迟作为控制信号。Caladan的调度核心只负责CPU调度，而Shenango的调度核心将网络处理与CPU调度结合在一起。Caladan使用KSCHED来更有效地执行调度功能，包括抢占任务、为任务分配核心、检测任务何时自愿放弃以及从远程核心读取性能计数器。Shenango使用标准的Linux系统调用来分配核心，限制了其可扩展性。

Caladan和Shenango的运行时具有许多相似之处。Caladan的应用程序在普通的Linux进程中运行，称为任务。运行时提供“绿色”线程和内核旁路I/O，使用工作窃取来平衡负载，并在没有工作可偷时释放核心。在用户空间处理线程和I/O使得管理干扰更容易。通过在需要处理的任务内执行所有处理，我们可以更好地管理它生成的资源争用。其次，我们可以轻松地仪器化运行时系统以导出正确的每个任务控制信号。

用户可以为每个任务分配一定数量的保证核心数，这些核心数在需要时始终可用。用户还可以为任务分配额外的可突发核心数，以利用任何空闲容量。此外，每个任务被指定为LC或BE。BE任务的优先级较低：只有在LC任务不需要时，它们才被分配可突发核心数，它们始终分配零个保证核心数，并根据需要进行限制以管理干扰。

为了避免干扰影响LC任务的性能，建议配置一些未分配任务的核心来提供足够的弹性。如果无法管理干扰，Caladan可以检测到并向集群调度程序报告，以便将任务迁移到其他机器。此外，以前的研究已经探索了在集群调度程序层添加类似的干扰协调和识别互补工作负载的方法。

5.2 The Caladan Scheduler

本文介绍了调度器的关键组件、控制信号和它们之间的交互。调度器包括**内存带宽控制器**和**超线程控制器**，它们限制了任务分配的核心数量和禁止兄弟核心对。顶层核心分配器考虑了这些限制和每个任务的资源配置，尝试最小化排队延迟，以管理负载变化和未缓解的干扰。控制器和核心分配器每10微秒运行一次。调度器可以快速重新分配核心，因此可以平均分配分数核心给任务。

PAGE 7

调度器从三个来源收集控制信号：运行时间提供有关请求处理时间和排队延迟的信息；DRAM控制器提供有关全局内存带宽使用情况的信息；KSCHED提供有关每个核心LLC缺失率的信息，并在任务自愿放弃时通知调度器核心。每个算法都依赖于一个可调参数。

5.2.1 The Top-level Core Allocator

顶层核心分配器的目标是向正在经历排队延迟的任务授予更多核心，无论这些延迟是由于持续的干扰还是由于负载变化引起的。它会定期检查每个任务的排队延迟，并尝试向延迟超过可配置的每个任务阈值（THRESH_QD）的任务添加核心。排队可以发生在每个运行时核心的绿色线程运行队列、网络入口队列、存储完成队列和计时器堆中。每个排队元素都包含其到达时间的时间戳，并且所有队列都放置在共享内存中。QueueingDelay()通过将每个队列中最老元素的延迟相加来计算每个核心的延迟。然后它报告任务核心中观察到的最大延迟。

当任务的延迟超过THRESH_QD时，分配器会循环遍历所有核心，检查哪些核心被超线程控制器允许，并检查每个核心上运行的任务。空闲核心可以分配给任何任务，但繁忙核心只有在核心配置允许的情况下才能被抢占。此外，BE任务永远不能抢占LC任务。

该算法通过三个因素来评分，首先优先选择两个空闲的兄弟核心，因为它们没有超线程干扰；其次，优先选择不同任务之间的超线程配对，因为当任务具有不同的性能瓶颈时，超线程最有效；最后，优化时间局部性，记录每个任务上次使用每个核心的时间，并给最近的时间戳最高分数。时间戳在超线程兄弟之间共享，反映它们共享的缓存资源。

核心分配器从KSCHED接收通知，每当一个运行时自愿放弃一个核心时（在算法1中未显示）。在这种情况下，它会更新task_on_core数组，并立即尝试将核心分配给另一个任务，减少核心空闲的时间。

5.2.2 The Memory Bandwidth Controller

本文介绍了一种内存带宽控制器，旨在利用大部分可用内存带宽，同时避免饱和。该控制器定期轮询DRAM控制器的全局内存带宽使用计数器，计算自上次轮询间隔以来的访问速率，并在其超过饱和阈值时触发。它通过依赖KSCHED从每个调度核的性能监控单元（PMU）高效采样LLC misses来将内存带宽使用归因于特定任务。等待10微秒的采样间隔足以准确估计LLC misses。带宽控制器每次运行时从最糟糕的任务中撤销一个核心，直到内存带宽不再饱和。当一个任务被限制时，另一个消耗更少内存带宽的任务仍然可以使用被限制的核心。

PAGE 8

KSCHED控制器使用PMU计数器来监测LLC缓存的缺失率。为了提高准确性，KSCHED使用本地时间戳计数器（TSC）来消除计时偏差，并且在测量间隔期间放弃被抢占或让出CPU的任务的样本。

5.2.3 The Hyperthread Controller

Caladan的超线程控制器检测到超线程干扰后，禁止使用兄弟超线程直到当前请求完成。运行时将时间戳放入共享内存中，以指示每个超线程何时开始处理绿色线程。超线程控制器使用GetRequestStartTime()检索这些时间戳，并检查当前线程是否运行超过每个任务处理时间阈值（THRESH_HT）。

当超过阈值时，控制器通过KSCHED禁止使用兄弟超线程。兄弟的运行时收到KSCHED的请求，抢占核心并将当前绿色线程放回其运行队列。顶层核心分配器可以检测到这一点，将不同的（未被禁止的）核心添加回来。然后，KSCHED使用mwait指令将兄弟放置在浅C1空闲状态中；mwait将本地超线程停放并重新分配共享物理核心资源给兄弟，从而提高其性能。

Caladan的超线程控制器具有全局知识，可以避免在高负载下降低吞吐量，同时优先加速处理时间最长的绿色线程，以保持尾延迟尽可能低。超线程控制器还可以解除核心禁令，以满足顶层核心分配器的需求。

Caladan的管理超线程干扰的方法受到Elfen调度的启发，两者都使用mwait来使超线程空闲。但是，Caladan的方法有两个关键区别：首先，Caladan的调度器可以制定和执行决策，利用全局知识，而Elfen则依赖于可信的BE任务来测量干扰和自愿放弃。其次，Elfen只能支持将一个LC任务和一个BE任务固定到每个超线程对中，而我们允许任何配对（甚至是自我配对），并且可以处理LC任务之间的干扰。这使得所有逻辑核心都可以被任何任务使用，从而实现了更高的吞吐量。

5.2.4 An Example: Reacting to Garbage Collection

Caladan的调度器在GC周期开始时会响应，当全局内存带宽使用超过阈值时，内存带宽控制器会从GC任务中撤回核心，直到总内存带宽使用量低于阈值。此时，LC任务可能会受到干扰，增加其排队延迟。调度器会授予LC任务额外的核心，直到其排队延迟再次低于阈值。成功缓解GC干扰后，LC任务将放弃额外的核心。

5.3 KSCHED : Fast and Scalable Scheduling

KSCHED的目标是高效地将CPU调度的控制权暴露给用户空间调度器核心。KSCHED必须克服Linux系统调用接口的限制，包括计算密集型的工作、阻塞和重新调度以及无法同时执行多个操作和查询其他核心的性能计数器。

PAGE 9

KSCHED采用了与Linux现有机制截然不同的方法，通过每个核心的共享内存区域支持调度器核心与运行在其他核心上的内核代码之间的直接通信。调度器核心将命令写入这些区域，然后使用ioctl()向远程核心发送IPIs来唤醒它们。KSCHED然后在远程核心上执行命令（在内核空间中），并写回结果。

KSCHED支持三个命令：唤醒任务（可能抢占当前任务）、空闲核心和读取性能计数器。在抢占任务或空闲核心之前，KSCHED向运行时发送信号，以便它在几微秒内干净地让出当前的绿色线程的寄存器状态并将其放回运行队列。然后，为了在核心上唤醒新任务，KSCHED锁定任务的亲和性，以防Linux将其迁移到另一个核心，并调用Linux调度程序。相反，如果要空闲一个核心，KSCHED调用mwait。最后，KSCHED可以在任何核心上采样任何性能计数器，并在响应中包括TSC。

KSCHED调度器通过IPI和轮询处理命令，避免了Linux标准空闲处理程序的使用。它使用监视指令和mwait指令来监视共享区域的缓存一致性消息，并在调度器核心写入共享区域时立即恢复执行。

KSCHED在发送IPI时利用中断控制器的组播功能，一次性发送多个IPI，降低成本。所有操作都是异步执行，避免阻塞。KSCHED在目标核心上执行昂贵的操作，如发送信号和任务亲和性分配，以降低开销。这些特性使得KSCHED能够以低开销执行调度操作，支持高并发任务。

6 Implementation

Caladan是基于Shenango的开源版本开发的，但我们实现了全新的调度器和KSCHED内核模块，分别为3524行和533行代码。Shenango是我们系统的良好起点，因为它具有功能丰富的运行时环境，支持绿色线程和TCP/IP网络。此外，Shenango的运行时环境已经设计用于处理信号以清晰地抢占核心。

本文介绍了对Shenango运行时的两个重要修改：一是将libibverbs库直接链接到每个运行时中，以提供快速的内核绕行网络访问，消除了Shenango的数据包转发瓶颈；二是使用Intel的SPDK库增加了对NVMe存储的支持，绕过了内核。这些修改需要添加2,943行新代码，主要是为了与libibverbs和SPDK集成。

KSCHED支持空闲状态，每个核心共享内存区域使用一个缓存行（64字节），以便mwait只能监视这个大小的区域。我们将这些缓存行打包成一个连续的数组，以便调度器核心可以利用硬件预取来加快轮询速度。KSCHED允许调度器核心控制mwait进入的空闲状态，但我们还没有探索功耗管理。我们还修改了Linux内核源代码以加速多播IPI；虽然Linux内核提供了一个名为smp_call_function_many()的API来支持此功能，但它会增加额外的软件开销，特别是在内存带宽干扰较大的情况下。

7 Evaluation

我们通过回答以下问题来评估Caladan：

Caladan与之前的系统相比如何？

Caladan能否同时完成不同任务并保持低尾延迟和高CPU利用率。

PAGE 10

Caladan的设计组件使其能够表现出色。

实验设置：我们在一台服务器上评估我们的系统，该服务器配备了两个12物理核心（24超线程）的Xeon Broadwell CPU和64 GB的RAM，运行着Ubuntu 18.04操作系统，内核版本为5.2.0（经过修改以加快多播IPI的速度）。我们不考虑NUMA，并将所有中断、内存分配和线程直接指向第一个插槽。该服务器配备了一个40 Gb/s的ConnectX-5 Mellanox NIC和一个280 GB的Intel Optane NVMe设备，可以以550,000 IOPS的速度进行随机读取。为了生成负载，我们使用一组四核机器，配备了10 Gb/s的ConnectX-3 Mellanox NIC，通过Mellanox SX1024非阻塞交换机连接到我们的服务器。我们按照推荐做法调整机器以实现低延迟，禁用TurboBoost、CPU空闲状态、CPU频率缩放和透明大页[37]。我们还禁用了Meltdown [2]和MDS [24]的修复措施，因为这些漏洞已经在最新的Intel CPU中修复。在评估Linux的性能时，我们使用SCHED_IDLE运行低优先级的BE任务，并使用内核版本5.4.0以利用对SCHED_IDLE的最新改进。我们使用一个名为loadgen的开环负载生成器，在TCP连接上生成具有泊松到达率的请求[61]。除非另有说明，否则我们将所有Caladan实验配置为具有22个保证核心的LC任务，为调度程序留出一个物理核心。

本文评估了三个LC任务：memcached、silo和storage。memcached是一个流行的内存键值存储，我们基于Facebook的USR请求分布生成读写混合负载。silo是一个先进的内存研究数据库，我们使用TPC-C请求模式进行测试。storage是一个新的NVMe块存储服务器，我们添加了压缩和加密功能，并预加载了来自维基百科的XML格式数据，以评估服务时间的变化。

我们使用PARSEC基准套件中的工作负载来进行BE任务的评估。具体来说，我们评估了x264（一种H.264/AVC视频编码器）、swaptions（一个投资组合定价工具）和streamcluster（一种在线聚类算法）。我们修改了swaptions，使用Boehm垃圾收集器来分配其内存对象，以便研究垃圾收集引起的干扰；我们将这个版本称为swaptions-GC。这三个工作负载都表现出阶段性行为，它们在规律的时间间隔内改变资源使用情况（有些变化幅度比其他工作负载大）。最后，我们评估了一个合成的对抗者，它不断地读写内存数组，有两种配置：stream-L2会占用L2缓存，而stream-DRAM会占用LLC并消耗所有可用的内存带宽。

该系统使用修改后的Shenango运行时，支持标准抽象，如TCP套接字和pthread接口，使得移植和开发应用程序相对简单。

Caladan有三个可调参数，可以在延迟和CPU效率之间进行权衡。在评估中，我们将这三个参数都调整为低延迟。首先，我们将队列延迟阈值设置为10微秒。其次，我们将内存带宽阈值设置为25 GB/s。最后，我们为每个LC任务设置了处理时间阈值，对于silo设置为25微秒，对于storage设置为40微秒，对于memcached设置为无限。

Parties是最相关的先前工作，用于减轻干扰。它在Heracles的基础上添加了对多个LC任务的支持。由于Parties的源代码不公开且无法获取，我们按照其论文中的细节重新实现了Parties。

我们通过自己实现Parties，能够在Caladan和Parties中使用相同的运行时系统，从而使它们都能够受益于内核绕过I/O，这样我们只需要评估调度策略的差异。我们没有实现Parties中与我们实验无关的一些组件，例如磁盘、网络或内存容量的争用。管理这些资源很重要，但与我们关注的CPU干扰无关。此外，我们没有包括CPU频率调节控制器，因为减少能耗不在我们的研究范围内。我们实现了Parties的所有关键机制，包括核心分配、CAT和一个外部测量客户端，它在500毫秒的时间段内对尾延迟进行采样。我们还花了很多精力来调整Parties的延迟阈值，以获得最佳性能。

Parties通常禁用超线程，因为无法管理这种干扰，降低了CPU吞吐量。但是，我们启用了超线程，并采用了自我配对策略，这对于memcached负载来说是一种互补的配对，对延迟影响很小，但允许我们进行同样数量核心的直接比较。添加内核旁路网络和超线程配对使我们的Parties*版本显著优于原始版本的性能，因此我们将其称为Parties*。

7.1 Comparison to Other Systems

PAGE 11

本文比较了不管理干扰的系统和管理干扰的系统，以展示管理干扰的必要性。在一个相对简单的场景中，将一个LC任务（memcached）与一个产生不断内存带宽干扰的BE任务（stream-DRAM）放置在一起进行评估。

Linux和Shenango在共存时，尾延迟显著增加，分别高达235倍和6倍。Shenango的吞吐量也下降了75％，因为其调度器核心由于流DRAM引起的高缓存未命中率和内存访问延迟而过载。相比之下，Caladan和Parties*能够明确地管理干扰，因此能够保持类似的尾延迟，并且实现比Linux和Shenango更高的吞吐量。他们使用内核旁路网络堆栈直接发送和接收数据包，防止Linux网络堆栈或调度器核心成为瓶颈。尽管将Shenango适应我们的运行时内核旁路网络堆栈可以消除吞吐量瓶颈，但它不会改善受干扰的LC任务的尾延迟。

本文研究了相位干扰问题，重点解决了垃圾回收实验中的干扰问题。在实验中，将memcached与swaptions-GC放在一起，每秒发出800,000个请求，测量其尾延迟。结果显示，前20秒的行为代表了整个实验期间的行为。

Caladan在每个GC周期开始时立即限制BE任务的核心数，以防止延迟峰值，并在GC周期结束时将核心数归还给BE任务，保持高BE吞吐量。然而，Parties*在资源需求在小于其500毫秒调整间隔的时间尺度上发生变化时无法收敛。Parties*经常在GC周期中响应性地分配额外的核心，但这些调整发生得太慢，无法防止延迟峰值。结果导致Parties*在GC周期中的延迟比Caladan高出11,000倍。此外，Parties*还会损害BE吞吐量，平均比Caladan低5%，因为它对swaptions-GC的惩罚过多且时间过长。这些结果表明，在处理具有阶段行为的任务时，更快的反应时间是必不可少的。

7.2 Diverse Colocations

本文评估了15个不同资源使用、服务时间分布和吞吐量的LC和BE任务之间的15个共存情况，以了解Caladan在不同情况下是否能保持其优势。其中9个配对包括具有分阶段行为的BE任务。我们考虑了使用可突发核心对每个LC任务的尾延迟和BE任务可以实现的吞吐量的影响。

PAGE 12

实验结果显示，Caladan在减轻干扰方面非常有效，存储和memcached的尾延迟几乎与没有共存时相同。Silo在低负载时会出现尾延迟的小幅增加，因为它对LLC干扰敏感，导致服务时间增加但排队延迟不变。在高负载时，Silo会产生自身干扰，因此它在有或没有共存时的尾延迟相似。总体而言，Caladan可以在具有挑战性的共存条件下轻松维持微秒级的尾延迟。

Caladan系统能够同时实现微秒级的LC尾延迟和高BE任务吞吐量。BE任务的吞吐量取决于与LC任务的资源争用程度，不同的BE任务与LC任务的交互方式也会影响吞吐量。Caladan是第一个在广泛条件下实现这两个目标的系统。

Caladan展示了同时管理多个任务的能力，通过将3个LC任务与swaptions-GC和streamcluster放置在一起进行实验。实验结果显示，当负载或干扰发生变化时，Caladan几乎立即收敛。当GC不运行时，streamcluster和swaptions-GC的组合不会饱和内存带宽。然而，当GC开始时，这两个任务一起饱和了DRAM带宽，并受到Caladan的限制。Caladan的快速反应能力使得三个LC任务能够在不断变化的负载和干扰中保持低尾延迟。

7.3 Microbenchmarks

PAGE 13

KSCHED进行了微基准测试，比较了使用标准Linux系统调用和KSCHED的调度操作速度。测试中不断将任务分配到不同的核心上，并测试了不同数量任务的迁移。测试结果表明，KSCHED的调度操作更快，具有更好的可扩展性。

KSCHED使用多播IPIs可以大大提高调度工作和调度延迟的效率，而Linux的系统调用接口则因为无法支持批处理而导致调度工作和调度延迟增加。此外，KSCHED通过将昂贵的操作（如向远程核发送信号）卸载到其他地方来保持低调度工作。

本文介绍了一种名为Caladan的系统，它使用了一种新的内核调度机制KSCHED，可以在保证服务质量的前提下提高系统的资源利用率。实验结果表明，Caladan可以显著降低任务的尾延迟，并且在高负载下仍能保持较好的性能。此外，KSCHED还可以提高性能计数器的采样效率。

研究发现，为了确保在各种任务和负载下的隔离性，内存带宽和超线程控制器都是必需的。举一个具体例子，当与流DRAM BE任务共存时，图8c评估了每个控制器模块对存储LC任务的贡献。在非常低的负载下，带宽控制器足以提供低尾延迟。这是因为当Caladan从BE任务中撤销核心时，它会使LC任务的超线程对核心空闲，从而使超线程控制器变得不必要。然而，在较高的LC负载下，为了使存储任务实现几乎与无共存时相同的尾延迟，两个控制器都是必需的。

本文研究了超线程控制器的作用，评估了允许任意两个任务在一个物理核心上共同运行的好处。通过比较Caladan和两个修改版的Caladan，发现Caladan通过允许LC任务自身共同运行，实现了比Elfen更高的吞吐量。同时，Caladan还能够在低LC负载下实现比Elfen更高的BE吞吐量。禁用超线程控制器会导致更高的BE吞吐量，但代价是更高的尾延迟。因此，需要明确管理超线程干扰，以实现高吞吐量和低尾延迟。

8 Discussion

PAGE 14

Caladan需要应用程序使用其运行时系统，以便快速映射线程和数据包处理工作到可用核心的不断变化的集合中。它提供了一个现实的并发编程模型，包括线程、互斥锁、条件变量和同步I/O。Caladan还包括一个系统库的部分兼容性层，可以支持PARSEC，但不完全兼容Linux。未使用Caladan运行时的应用程序可以在同一台机器上共存，但它们必须在未由Caladan管理的核心上运行，并且如果它们引起干扰，它们不能被限制。

Caladan需要任务暴露内部并发性以利用多核处理器，建议每个连接或请求生成一个线程。如果并发性不足，任务将无法受益于额外的核心，阻碍Caladan管理负载或干扰的转移。例如，memcached将多个TCP连接复用到一个线程中，但我们修改它以生成一个单独的线程来处理每个TCP连接。

Caladan可以支持不暴露内部并发性的BE任务，并且如果它们引起太多干扰，仍然可以对它们进行限制。例如，如果一个BE任务是单线程的（即没有并发性），并且消耗了太多的内存带宽，Caladan将在给予它一个和零个核心之间进行振荡，有效地对其内存带宽使用进行时间复用。然而，BE任务可以选择通过暴露其内部并发性来实现更高的性能：负载将更加均衡，并且它们将能够利用可突发的核心。

Caladan目前存在两个限制：无法管理NUMA节点间的干扰，调度策略不能最小化超线程兄弟之间的瞬态执行攻击威胁。未来计划探索NUMA感知的干扰缓解策略和开发类似Linux内核的能力。

未来工作：未来的一个有前景的机会是将硬件分区重新纳入Caladan的设计中。例如，如果一个BE任务使用高内存带宽并且缺乏时间局部性，它在LLC中占用的许多缓存行将被浪费。在这种情况下，Caladan仍然有效地防止延迟增加，但必须为受害任务分配额外的核心。如果未来的硬件分区机制能够设计成适应频繁变化的LLC使用，或者通过现有机制能够识别和管理静态LLC使用，CPU的效率可以进一步提高。

9 Related Work

许多先前的系统通过静态分配资源来管理LC和BE任务之间的干扰，但这种方法会牺牲CPU利用率。Heracles、Parties和PerfIso可以动态调整分区，但不能在保持微秒级延迟和高利用率的同时管理干扰变化。Caladan可以做到这一点。

Caladan不仅仅是一个网络隔离的解决方案，还包括存储隔离。目前不关注功率管理和TurboBoost，但可以将其与Caladan集成以提高CPU效率。优化的目标是所有核心都被充分利用。

为了在面对波动负载时实现低延迟，一些系统引入了用户级核心分配器，例如IX、PerfIso、Shenango和Arachne。Caladan通过核心分配来管理干扰，进一步管理负载变化。TAS和Snap也可以根据数据包处理负载的变化来调整核心。

Shinjuku提出了细粒度抢占来减少尾延迟，使用Dune在用户空间提供快速、直接访问IPI。KSCHED包括内核优化，允许在向单个核心发送IPI时获得类似的性能，但在发送多个IPI时，由于其多播IPI优化，它比Shinjuku报告的速度更快。

该文提到了针对数据平面系统的优化工作，包括通过工作窃取技术减少尾延迟、利用绿色线程提高可编程性、以及消除网络处理和排队瓶颈等。Caladan结合了这些想法，消除了软件开销和负载不平衡的干扰，从而管理干扰。

10 Conclusion

PAGE 15

本文介绍了Caladan，一种考虑干扰的CPU调度器，可以显著提高性能隔离并保持高CPU利用率。Caladan的有效性来自于其速度：通过将控制信号和行动匹配到干扰影响性能的相同时间尺度，Caladan可以在干扰影响服务质量之前减轻干扰。Caladan依赖于精心选择的一组控制信号来协调管理多种形式的干扰，并结合广泛的优化来快速收集控制信号并使核心分配更快。这些贡献使Caladan能够在相位行为的多个任务共存时提供微秒级的尾延迟和高CPU利用率。

11 Acknowledgments

本文感谢导师和匿名审稿人的反馈，以及CloudLab和Mellanox提供的设备。研究得到了DARPA FastNICs计划、Facebook研究奖和Google教师奖的资助。研究的重点是开发高效的网络卡，名为Caladan。

A Parameter Tuning and Sensitivity

本附录介绍了如何设置Caladan的三个用户可调参数，并展示了这些参数的特定选择对Caladan性能的敏感程度。为了说明THRESH_QD和THRESH_HT的行为，我们将存储工作负载与stream-L2放置在一起。对于THRESH_BW，我们将存储工作负载与stream-DRAM放置在一起。在每种情况下，我们改变一个参数，同时将其他参数固定为我们评估中使用的值。

THRESH_QD是任务排队延迟限制，超过这个限制后，顶层核心分配器才会尝试授予另一个核心。通过将THRESH_QD设置为大于Caladan默认值10微秒，可以在一定程度上牺牲LC尾延迟，以换取更高的BE吞吐量。我们选择优化尾延迟，发现低于10微秒的值会降低BE吞吐量，而不会进一步改善LC尾延迟。

THRESH_HT是一个限制请求被任务生成的干扰延迟的最坏情况的阈值。如果设置得太低（即大多数请求处理需要专用物理核心），则BE吞吐量将受到影响，高负载时LC延迟会因计算能力不足而降低。对于偏斜的服务时间分布，如我们的存储工作负载，选择一个高于中位数的值是一个好的启发。图10b说明了将THRESH_HT设置在中位数以下会显著降低BE吞吐量，而略高于中位数的值会增加BE吞吐量并提供良好的尾延迟。对于服务时间小于5微秒的工作负载（例如memcached），我们建议将THRESH_HT设置为无限，因为mwait需要几微秒来停放超线程。

THRESH_BW是全局允许的最大内存带宽使用量，超过此值Caladan会开始限制任务。应该在每台机器上设置THRESH_BW，以避免接近内存带宽饱和时内存访问延迟呈指数增长。我们的机器设置为25 GB/s，保持任何访问模式的内存延迟较低。这种设置在可预测的延迟方面换取了一小部分BE吞吐量。

B Artifact

PAGE 16

Caladan的源代码、移植应用程序和实验脚本可以在https://github.com/shenango/caladan-all找到。
