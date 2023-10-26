关于这部分，首先得说明go的协程模型MGP

# MGP模型（顺序不知道）

P（processor），M（thread），G（goroutine）

按照网上的说法是这样的

整体是一个M:N的用户态线程映射到内核态线程的工作



我的理解如下，一个go的进程，里面包含了若干个线程（M）

而最多情况下，每个物理核心上面（最多每个物理核心都有，最少不知道），都挂了一个G的队列，还有对应的runtime（相当于P），也就是说对应的P上面挂了一串G

当某个线程M被OS调度到对应的P上面去时，就去对应的P上面的G队列里面轮询并执行对应的G的任务。

除了每个P上面挂的队列里面的任务，还有一个全局的global的队列，新创建的会塞到到本地的G队列，而如果满了，就扔到全局

# 一些调度细节

如果某个M发现P的队列空了，就先去全局看看有没有，如果全局也没有，就去其他那里偷一半过来。

如果某个G因为系统调用阻塞了，也会阻塞M，系统就会先去找没抢到P的线程M，如果没有就创建一个新的M，继续从P队列上面挂着的但是没阻塞的进行调度。

如果G因为channel或者I/O和network阻塞了，那么M不会被阻塞，而是M寻找其他的可运行的G



还有一个特殊的G0

每个M都有一个G0，而全局的G0是M0的G0



自旋问题，某个M发现自己的P的队列空了，全局也空了，其他在跑的都只有一个可以用的协程（G），那他只能继续跑他自己的G0然后自旋

但是有个问题是为什么要浪费CPU，因为希望它能够当某个go协程进来之后，能够立马有个M可以跑他，但也不能全都自旋，因此设定了一个上限，超过上限就让那个M休眠。



以上是网上对这个Go的调度器的架构上的大概说明

# 在每个队列中实际使用的调度算法

对于具体的调度算法，应该是按照队列轮询的，但是对于过长的任务也可以抢占

## go的抢占式调度

### 上古版本

首先，据说go以前的栈空间有限，因此在函数调用的时候进行扩栈

因此扩栈的时候可以检查是否有抢占请求

但是不是所有运行代码都需要扩栈，所以死循环就gg了

### 当前版本

go还搞了一个sysmon线程，这个线程不会放在P里面，所以不会因为各种奇奇怪怪的原因不被调度到，而是始终都能守护

现在的机制是使用这个sysmon线程来搞事情，如果检测到某个G跑的时间太长或者很久没有GC，就发信号给M，而M本身作为一个内核线程，是能注册一个信号的，因此可以通过这种方式来进行调度。

但是go本身的调度器应该还是说，没有什么所谓的优先级等等问题。



以上是我网上能找到的资料（并且按照自己的理解写了一遍）

总而言之，深刻实践了所谓的M:N的用户态线程映射到内核态线程的工作。。。



# Go调度器代码阅读研究

首先去github下个源码

git@github.com:golang/go.git



然后网上盗了个图



```
                            +-------------------- sysmon ---------------//------+ 
                            |                                                   |
                            |                                                   |
               +---+      +---+-------+                   +--------+          +---+---+
go func() ---> | G | ---> | P | local | <=== balance ===> | global | <--//--- | P | M |
               +---+      +---+-------+                   +--------+          +---+---+
                            |                                 |                 | 
                            |      +---+                      |                 |
                            +----> | M | <--- findrunnable ---+--- steal <--//--+
                                   +---+ 
                                     |
                                   mstart
                                     |
              +--- execute <----- schedule 
              |                      |   
              |                      |
              +--> G.fn --> goexit --+ 
```

然后这边一堆数据结构放在src/runtime/runtime2.go中

包括如下几个数据结构

**type g struct**

里面包含了

协程的栈顶栈底

当前g的m

gobuf等一些相关指针寄存器，相当于上下文信息

是否可抢占的标记

goroutine函数的入口地址

等等



除此之外，还给G定义了一堆status



**type p struct**

这里面定义的内容

我觉得重要的包括几个，一个是指向m的结构，一个是g的队列

还有就是指向下一个g

空的g的链表等等

更多相当于在某个逻辑处理器上的资源

**type m struct**

M是相当于具体的工作线程

他首先有个g0

然后有指向当前p和g的数据结构

m的状态，有Spinning和blocked

**type schedt struct**

调度器

```
        midle        muintptr // idle m's waiting for work
    nmidle       int32    // number of idle m's waiting for work
    nmidlelocked int32    // number of locked m's waiting for work
    mnext        int64    // number of m's that have been created and next M ID
    maxmcount    int32    // maximum number of m's allowed (or die)
    nmsys        int32    // number of system m's not counted for deadlock
    nmfreed      int64    // cumulative number of freed m's
```

这些都是关于m的数据结构

其他还有对应的空闲的g和p的管理的数据结构

而对应的，找了一个全局的sched的结构体放在这里

```
var (
    m0           m
    g0           g
    mcache0      *mcache
    raceprocctx0 uintptr
    raceFiniLock mutex
)
```

用全局的变量m0的g0表示主线程的主协程





然后具体的代码调度，应该放在proc.go当中，

注释里面分享了一个设计思路https://golang.org/s/go11sched，至于我为什么打不开，唔，大概是我没有魔法吧



对于新的G的创建，

newproc函数

```
func newproc(fn *funcval) {
    gp := getg()
    pc := getcallerpc()
    //systemstack是切换到m0对应的g0，然后创建新的goroutine对象
    systemstack(func() {
        newg := newproc1(fn, gp, pc)

        pp := getg().m.p.ptr()
        //放到某个p下面的队列中去
        runqput(pp, newg, true)

        if mainStarted {
            wakep()
        }
    })
}
```

对于systemstack是在asm_amd64.s中的汇编（应该别的架构也有吧）

然后主要是一个newproc1，这是主要创建的函数

比较长

```
// Create a new g in state _Grunnable, starting at fn. callerpc is the
// address of the go statement that created this. The caller is responsible
// for adding the new g to the scheduler.
func newproc1(fn *funcval, callergp *g, callerpc uintptr) *g {
    if fn == nil {
        fatal("go of nil func value")
    }
        //首先就是按照这说的去禁用抢占
    mp := acquirem() // disable preemption because we hold M and P in local vars.
    //然后需要去找一个p
        pp := mp.p.ptr()
        //看看有没有空的p结构，如果有，则直接复用
    newg := gfget(pp)
        //没有就创建一个新的
    if newg == nil {
        newg = malg(stackMin)
                //设置g的状态，按照注释说的那样，让他变成dead，GC就不访问
        casgstatus(newg, _Gidle, _Gdead)
        allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
    }
    if newg.stack.hi == 0 {
        throw("newproc1: newg missing stack")
    }

    if readgstatus(newg) != _Gdead {
        throw("newproc1: new g is not Gdead")
    }

    totalSize := uintptr(4*goarch.PtrSize + sys.MinFrameSize) // extra space in case of reads slightly beyond frame
    totalSize = alignUp(totalSize, sys.StackAlign)
    sp := newg.stack.hi - totalSize
    if usesLR {
        // caller's LR
        *(*uintptr)(unsafe.Pointer(sp)) = 0
        prepGoExitFrame(sp)
    }
    if GOARCH == "arm64" {
        // caller's FP
        *(*uintptr)(unsafe.Pointer(sp - goarch.PtrSize)) = 0
    }
        //设置一大堆属性
    memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
    newg.sched.sp = sp
    newg.stktopsp = sp
        //按照这个说法是，go exit需要被设置为运行完成之后使用的一个函数，因此需要在gostartcallfn中重新指个地方，将这个作为sp，而把真正的pc再塞进去，这样可以返回的时候就返回到goexit
    newg.sched.pc = abi.FuncPCABI0(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
    newg.sched.g = guintptr(unsafe.Pointer(newg))
        //这里面继续处理g启动的时候的上下文相关指针的设置
    gostartcallfn(&newg.sched, fn)
    newg.parentGoid = callergp.goid
    newg.gopc = callerpc
    newg.ancestors = saveAncestors(callergp)
    newg.startpc = fn.fn
    if isSystemGoroutine(newg, false) {
        sched.ngsys.Add(1)
    } else {
        // Only user goroutines inherit pprof labels.
        if mp.curg != nil {
            newg.labels = mp.curg.labels
        }
        if goroutineProfile.active {
            // A concurrent goroutine profile is running. It should include
            // exactly the set of goroutines that were alive when the goroutine
            // profiler first stopped the world. That does not include newg, so
            // mark it as not needing a profile before transitioning it from
            // _Gdead.
            newg.goroutineProfiled.Store(goroutineProfileSatisfied)
        }
    }
    // Track initial transition?
    newg.trackingSeq = uint8(fastrand())
    if newg.trackingSeq%gTrackingPeriod == 0 {
        newg.tracking = true
    }
        //设置为可运行的。
    casgstatus(newg, _Gdead, _Grunnable)
    gcController.addScannableStack(pp, int64(newg.stack.hi-newg.stack.lo))

    if pp.goidcache == pp.goidcacheend {
        // Sched.goidgen is the last allocated id,
        // this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
        // At startup sched.goidgen=0, so main goroutine receives goid=1.
        pp.goidcache = sched.goidgen.Add(_GoidCacheBatch)
        pp.goidcache -= _GoidCacheBatch - 1
        pp.goidcacheend = pp.goidcache + _GoidCacheBatch
    }
    newg.goid = pp.goidcache
    pp.goidcache++
    if raceenabled {
        newg.racectx = racegostart(callerpc)
        newg.raceignore = 0
        if newg.labels != nil {
            // See note in proflabel.go on labelSync's role in synchronizing
            // with the reads in the signal handler.
            racereleasemergeg(newg, unsafe.Pointer(&labelSync))
        }
    }
    if traceEnabled() {
        traceGoCreate(newg, newg.startpc)
    }
        //这里需要释放m和g，因为现在绑定的m应该还是m0
    releasem(mp)

    return newg
}
```

换句话说，还是用m0去绑定了新生成的一个g，然后让他去进行创建的这些工作。



然后，这里面最重要的工作我认为是，设置好从runtime进入以及结束后从runtime退出的机制

上面已经说了先设置了goexit的sp到pc去，然后来看gostartcallfn

gostartcallfn

```
// adjust Gobuf as if it executed a call to fn
// and then stopped before the first instruction in fn.
func gostartcallfn(gobuf *gobuf, fv *funcval) {
    var fn unsafe.Pointer
    if fv != nil {
        fn = unsafe.Pointer(fv.fn)
    } else {
        fn = unsafe.Pointer(abi.FuncPCABIInternal(nilfunc))
    }
    gostartcall(gobuf, fn, unsafe.Pointer(fv))
}
```

以及他调用的在sys_x86.go中的

```
// adjust Gobuf as if it executed a call to fn with context ctxt
// and then stopped before the first instruction in fn.
func gostartcall(buf *gobuf, fn, ctxt unsafe.Pointer) {
    sp := buf.sp
    sp -= goarch.PtrSize
    *(*uintptr)(unsafe.Pointer(sp)) = buf.pc
    buf.sp = sp
    buf.pc = uintptr(fn)
    buf.ctxt = ctxt
}
```

就是使用类似于模拟栈的方式，让他类似于被调用的样子，但是，并没有真正的发生调用关系



对于G的切换

在asm_amd64.s中

```
// func gogo(buf *gobuf)
// restore state from Gobuf; longjmp
TEXT runtime·gogo(SB), NOSPLIT, $0-8
    MOVQ    buf+0(FP), BX        // gobuf
    MOVQ    gobuf_g(BX), DX
    MOVQ    0(DX), CX        // make sure g != nil
    JMP    gogo<>(SB)

TEXT gogo<>(SB), NOSPLIT, $0
    get_tls(CX)
    MOVQ    DX, g(CX)
    MOVQ    DX, R14        // set the g register
        //切换栈
    MOVQ    gobuf_sp(BX), SP    // restore SP
    MOVQ    gobuf_ret(BX), AX
    MOVQ    gobuf_ctxt(BX), DX
    MOVQ    gobuf_bp(BX), BP
    MOVQ    $0, gobuf_sp(BX)    // clear to help garbage collector
    MOVQ    $0, gobuf_ret(BX)
    MOVQ    $0, gobuf_ctxt(BX)
    MOVQ    $0, gobuf_bp(BX)
    MOVQ    gobuf_pc(BX), BX
    JMP    BX
```

这里面的切换看上去非常的轻量级，就不像正常切换线程那样，需要保存一大堆寄存器。

就只有这几个寄存器是需要考虑的（是这样的嘛？？？）



对于G的结束，main的结束直接就结束了（是调用的系统的exit，相当于线程的退出，但是我不清楚arceos下要做这个事情如何同步多个vcpu

对于协程

```
// The top-most function running on a goroutine
// returns to goexit+PCQuantum.
TEXT runtime·goexit(SB),NOSPLIT|TOPFRAME|NOFRAME,$0-0
    BYTE    $0x90    // NOP
    CALL    runtime·goexit1(SB)    // does not return
    // traceback from goexit1 must hit code range of goexit
    BYTE    $0x90    // NOP

```

然后调用了goexit1

```
// Finishes execution of the current goroutine.
func goexit1() {
    if raceenabled {
        racegoend()
    }
    if traceEnabled() {
        traceGoEnd()
    }
    mcall(goexit0)
}
```

再调用goexit0，mcall也和前面创建的时候systemstack一样，切换到g0的栈

```

// goexit continuation on g0.
func goexit0(gp *g) {
    mp := getg().m
    pp := mp.p.ptr()
        //切换状态
    casgstatus(gp, _Grunning, _Gdead)
    gcController.addScannableStack(pp, -int64(gp.stack.hi-gp.stack.lo))
    if isSystemGoroutine(gp, false) {
        sched.ngsys.Add(-1)
    }
        //清理所有的数据结构
    gp.m = nil
    locked := gp.lockedm != 0
    gp.lockedm = 0
    mp.lockedg = 0
    gp.preemptStop = false
    gp.paniconfault = false
    gp._defer = nil // should be true already but just in case.
    gp._panic = nil // non-nil for Goexit during panic. points at stack-allocated data.
    gp.writebuf = nil
    gp.waitreason = waitReasonZero
    gp.param = nil
    gp.labels = nil
    gp.timer = nil

    if gcBlackenEnabled != 0 && gp.gcAssistBytes > 0 {
        // Flush assist credit to the global pool. This gives
        // better information to pacing if the application is
        // rapidly creating an exiting goroutines.
        assistWorkPerByte := gcController.assistWorkPerByte.Load()
        scanCredit := int64(assistWorkPerByte * float64(gp.gcAssistBytes))
        gcController.bgScanCredit.Add(scanCredit)
        gp.gcAssistBytes = 0
    }

    dropg()

    if GOARCH == "wasm" { // no threads yet on wasm
        gfput(pp, gp)
        schedule() // never returns
    }

    if mp.lockedInt != 0 {
        print("invalid m->lockedInt = ", mp.lockedInt, "\n")
        throw("internal lockOSThread error")
    }
        //还记得开始创建的时候，那个从空队列中找一个g的事情嘛，这就是循环利用了
    gfput(pp, gp)
    if locked {
        // The goroutine may have locked this thread because
        // it put it in an unusual kernel state. Kill it
        // rather than returning it to the thread pool.

        // Return to mstart, which will release the P and exit
        // the thread.
        if GOOS != "plan9" { // See golang.org/issue/22227.
            gogo(&mp.g0.sched)
        } else {
            // Clear lockedExt on plan9 since we may end up re-using
            // this thread.
            mp.lockedExt = 0
        }
    }
        //调度
    schedule()
}
```





对于P的创建

在schedinit中设置好procs，然后再procresize

```
// initialize new P's
    for i := old; i < nprocs; i++ {
        pp := allp[i]
        if pp == nil {
            pp = new(p)
        }
        pp.init(i)
        atomicstorep(unsafe.Pointer(&allp[i]), unsafe.Pointer(pp))
    }
```

大概就是这样了，我觉得主要是资源的分配工作



关于M的创建



```
func newm(fn func(), pp *p, id int64) {
    acquirem()

    mp := allocm(pp, fn, id)
    mp.nextp.set(pp)
    mp.sigmask = initSigmask
    if gp := getg(); gp != nil && gp.m != nil && (gp.m.lockedExt != 0 || gp.m.incgo) && GOOS != "plan9" {
        lock(&newmHandoff.lock)
        if newmHandoff.haveTemplateThread == 0 {
            throw("on a locked thread with no template thread")
        }
        mp.schedlink = newmHandoff.newm
        newmHandoff.newm.set(mp)
        if newmHandoff.waiting {
            newmHandoff.waiting = false
            notewakeup(&newmHandoff.wake)
        }
        unlock(&newmHandoff.lock)
        releasem(getg().m)
        return
    }
    newm1(mp)
    releasem(getg().m)
}
```

说是这里创建的，但是，真正的OS线程在newm1中

这里调用了一个newostproc，实际上在不同os中有不同实现



对于m的休眠

```
func stopm() {
    gp := getg()

    if gp.m.locks != 0 {
        throw("stopm holding locks")
    }
    if gp.m.p != 0 {
        throw("stopm holding p")
    }
    if gp.m.spinning {
        throw("stopm spinning")
    }

    lock(&sched.lock)
    mput(gp.m)
    unlock(&sched.lock)
    mPark()
    acquirep(gp.m.nextp.ptr())
    gp.m.nextp = 0
}
```

这里面调用mPark，然后最终会通过对应OS的系统调用，来使得其进入休眠线程



对于m的唤醒

```
func wakep() {
    if sched.nmspinning.Load() != 0 || !sched.nmspinning.CompareAndSwap(0, 1) {
        return
    }
    mp := acquirem()

    var pp *p
    lock(&sched.lock)
    pp, _ = pidlegetSpinning(0)
    if pp == nil {
        if sched.nmspinning.Add(-1) < 0 {
            throw("wakep: negative nmspinning")
        }
        unlock(&sched.lock)
        releasem(mp)
        return
    }
    unlock(&sched.lock)

    startm(pp, true, false)

    releasem(mp)
}
```

这里主要是判断是否具有自旋的M，也就是

if sched.nmspinning.Load() != 0 || !sched.nmspinning.CompareAndSwap(0, 1) {
        return
    }

这一句

然后startm

```
func startm(pp *p, spinning, lockheld bool) {
    mp := acquirem()
    if !lockheld {
        lock(&sched.lock)
    }
    if pp == nil {
        if spinning {
            throw("startm: P required for spinning=true")
        }
        pp, _ = pidleget(0)
        if pp == nil {
            if !lockheld {
                unlock(&sched.lock)
            }
            releasem(mp)
            return
        }
    }
    nmp := mget()
    if nmp == nil {
        id := mReserveID()
        unlock(&sched.lock)

        var fn func()
        if spinning {
            // The caller incremented nmspinning, so set m.spinning in the new M.
            fn = mspinning
        }
        newm(fn, pp, id)

        if lockheld {
            lock(&sched.lock)
        }
        // Ownership transfer of pp committed by start in newm.
        // Preemption is now safe.
        releasem(mp)
        return
    }
    if !lockheld {
        unlock(&sched.lock)
    }
    if nmp.spinning {
        throw("startm: m is spinning")
    }
    if nmp.nextp != 0 {
        throw("startm: m has p")
    }
    if spinning && !runqempty(pp) {
        throw("startm: p has runnable gs")
    }
    // The caller incremented nmspinning, so set m.spinning in the new M.
    nmp.spinning = spinning
    nmp.nextp.set(pp)
    notewakeup(&nmp.park)
    // Ownership transfer of pp committed by wakeup. Preemption is now
    // safe.
    releasem(mp)
}
```

这里最重要的是，如果没有找到合适的m进行唤醒，那么就会走入创建新的m（也就是newm）的流程





基本的GMP数据结构说完了，然后来讲讲sched

讲sched还是要从初始化讲起

```
TEXT runtime·rt0_go(SB),NOSPLIT|NOFRAME|TOPFRAME,$0
    // copy arguments forward on an even stack
    MOVQ    DI, AX        // argc
    MOVQ    SI, BX        // argv
    SUBQ    $(5*8), SP        // 3args 2auto
    ANDQ    $~15, SP
    MOVQ    AX, 24(SP)
    MOVQ    BX, 32(SP)

    // create istack out of the given (operating system) stack.
    // _cgo_init may update stackguard.
    MOVQ    $runtime·g0(SB), DI
    LEAQ    (-64*1024)(SP), BX
    MOVQ    BX, g_stackguard0(DI)
    MOVQ    BX, g_stackguard1(DI)
    MOVQ    BX, (g_stack+stack_lo)(DI)
    MOVQ    SP, (g_stack+stack_hi)(DI)

    // find out information about the processor we're on
    MOVL    $0, AX
    CPUID
    CMPL    AX, $0
    JE    nocpuinfo

    CMPL    BX, $0x756E6547  // "Genu"
    JNE    notintel
    CMPL    DX, $0x49656E69  // "ineI"
    JNE    notintel
    CMPL    CX, $0x6C65746E  // "ntel"
    JNE    notintel
    MOVB    $1, runtime·isIntel(SB)

notintel:
    // Load EAX=1 cpuid flags
    MOVL    $1, AX
    CPUID
    MOVL    AX, runtime·processorVersionInfo(SB)

nocpuinfo:
    // if there is an _cgo_init, call it.
    MOVQ    _cgo_init(SB), AX
    TESTQ    AX, AX
    JZ    needtls
    // arg 1: g0, already in DI
    MOVQ    $setg_gcc<>(SB), SI // arg 2: setg_gcc
    MOVQ    $0, DX    // arg 3, 4: not used when using platform's TLS
    MOVQ    $0, CX
        
/// ......这里弄了一大堆OS相关代码

    LEAQ    runtime·m0+m_tls(SB), DI
    CALL    runtime·settls(SB)

    // store through it, to make sure it works
    get_tls(BX)
    MOVQ    $0x123, g(BX)
    MOVQ    runtime·m0+m_tls(SB), AX
    CMPQ    AX, $0x123
    JEQ 2(PC)
    CALL    runtime·abort(SB)
ok:
    // set the per-goroutine and per-mach "registers"
    get_tls(BX)
    LEAQ    runtime·g0(SB), CX
    MOVQ    CX, g(BX)
    LEAQ    runtime·m0(SB), AX

        //绑定m0和g0
    // save m->g0 = g0
    MOVQ    CX, m_g0(AX)
    // save m0 to g0->m
    MOVQ    AX, g_m(CX)

    CLD                // convention is D is always left cleared

    // Check GOAMD64 requirements
    // We need to do this after setting up TLS, so that
    // we can report an error if there is a failure. See issue 49586.

/// ......其他一样的一堆配置

    CALL    runtime·check(SB)

    MOVL    24(SP), AX        // copy argc
    MOVL    AX, 0(SP)
    MOVQ    32(SP), AX        // copy argv
    MOVQ    AX, 8(SP)
    CALL    runtime·args(SB)
    CALL    runtime·osinit(SB)
    CALL    runtime·schedinit(SB)

    // create a new goroutine to start program
    MOVQ    $runtime·mainPC(SB), AX        // entry
    PUSHQ    AX
    CALL    runtime·newproc(SB)
    POPQ    AX

    // start this M
    CALL    runtime·mstart(SB)

    CALL    runtime·abort(SB)    // mstart should never return
    RET


```

总而言之，在这里最后会调用runtime的args ，osinit，schedinit

这个schedinit上面也说过了

然后创建mainp和newporc（这里创建新的goroutine）

最后跑mstart，并且一去不回



mstart（不是前面的startm）

```
TEXT runtime·mstart(SB),NOSPLIT|TOPFRAME,$0
    CALL    runtime·mstart0(SB)
    RET // not reached

```

```
func mstart0() {
    gp := getg()

    osStack := gp.stack.lo == 0
    if osStack {
        // Initialize stack bounds from system stack.
        // Cgo may have left stack size in stack.hi.
        // minit may update the stack bounds.
        //
        // Note: these bounds may not be very accurate.
        // We set hi to &size, but there are things above
        // it. The 1024 is supposed to compensate this,
        // but is somewhat arbitrary.
        size := gp.stack.hi
        if size == 0 {
            size = 16384 * sys.StackGuardMultiplier
        }
        gp.stack.hi = uintptr(noescape(unsafe.Pointer(&size)))
        gp.stack.lo = gp.stack.hi - size + 1024
    }
    // Initialize stack guard so that we can start calling regular
    // Go code.
    gp.stackguard0 = gp.stack.lo + stackGuard
    // This is the g0, so we can also call go:systemstack
    // functions, which check stackguard1.
    gp.stackguard1 = gp.stackguard0
    mstart1()

    // Exit this thread.
    if mStackIsSystemAllocated() {
        // Windows, Solaris, illumos, Darwin, AIX and Plan 9 always system-allocate
        // the stack, but put it in gp.stack before mstart,
        // so the logic above hasn't set osStack yet.
        osStack = true
    }
    mexit(osStack)
}
```

然后是mstart1

```
// The go:noinline is to guarantee the getcallerpc/getcallersp below are safe,
// so that we can set up g0.sched to return to the call of mstart1 above.
//
//go:noinline
func mstart1() {
    gp := getg()

    if gp != gp.m.g0 {
        throw("bad runtime·mstart")
    }

    // Set up m.g0.sched as a label returning to just
    // after the mstart1 call in mstart0 above, for use by goexit0 and mcall.
    // We're never coming back to mstart1 after we call schedule,
    // so other calls can reuse the current frame.
    // And goexit0 does a gogo that needs to return from mstart1
    // and let mstart0 exit the thread.
    gp.sched.g = guintptr(unsafe.Pointer(gp))
    gp.sched.pc = getcallerpc()
    gp.sched.sp = getcallersp()

    asminit()
    minit()

    // Install signal handlers; after minit so that minit can
    // prepare the thread to be able to handle the signals.
    if gp.m == &m0 {
        mstartm0()
    }

    if fn := gp.m.mstartfn; fn != nil {
        fn()
    }

    if gp.m != &m0 {
        acquirep(gp.m.nextp.ptr())
        gp.m.nextp = 0
    }
    schedule()
}
```

这里面初始化了m0，需要设置一些信号量

然后才跑到了main

main函数在proc里面，还生成了一个sysmon函数



调度策略

唔，激动人心的调度策略终于要来了吗

这个函数在schedule.go中

他实在是太长了，但是，更重要的是，他更多的是用来处理各种block的原因

在这里用了一个Priority queue



他通过记录一个

score[v.ID]

来决定调度的顺序



他还这里定义了一个edge结构体

每个edge要求其被调度的时候必须有确切的顺序

也就是如果x到y有一条边，那么x一定要先于y去调度



然后根据这个score和edge的信息去做这个优先级队列



但是奇怪的是在proc.go 中，我也找到了一个schedule函数

```
// One round of scheduler: find a runnable goroutine and execute it.
// Never returns.
func schedule() {
    mp := getg().m

    if mp.locks != 0 {
        throw("schedule: holding locks")
    }

    if mp.lockedg != 0 {
        stoplockedm()
        execute(mp.lockedg.ptr(), false) // Never returns.
    }

    // We should not schedule away from a g that is executing a cgo call,
    // since the cgo call is using the m's g0 stack.
    if mp.incgo {
        throw("schedule: in cgo")
    }

top:
    pp := mp.p.ptr()
    pp.preempt = false

    // Safety check: if we are spinning, the run queue should be empty.
    // Check this before calling checkTimers, as that might call
    // goready to put a ready goroutine on the local run queue.
    if mp.spinning && (pp.runnext != 0 || pp.runqhead != pp.runqtail) {
        throw("schedule: spinning with local work")
    }

    gp, inheritTime, tryWakeP := findRunnable() // blocks until work is available

    if debug.dontfreezetheworld > 0 && freezing.Load() {
        // See comment in freezetheworld. We don't want to perturb
        // scheduler state, so we didn't gcstopm in findRunnable, but
        // also don't want to allow new goroutines to run.
        //
        // Deadlock here rather than in the findRunnable loop so if
        // findRunnable is stuck in a loop we don't perturb that
        // either.
        lock(&deadlock)
        lock(&deadlock)
    }

    // This thread is going to run a goroutine and is not spinning anymore,
    // so if it was marked as spinning we need to reset it now and potentially
    // start a new spinning M.
    if mp.spinning {
        resetspinning()
    }

    if sched.disable.user && !schedEnabled(gp) {
        // Scheduling of this goroutine is disabled. Put it on
        // the list of pending runnable goroutines for when we
        // re-enable user scheduling and look again.
        lock(&sched.lock)
        if schedEnabled(gp) {
            // Something re-enabled scheduling while we
            // were acquiring the lock.
            unlock(&sched.lock)
        } else {
            sched.disable.runnable.pushBack(gp)
            sched.disable.n++
            unlock(&sched.lock)
            goto top
        }
    }

    // If about to schedule a not-normal goroutine (a GCworker or tracereader),
    // wake a P if there is one.
    if tryWakeP {
        wakep()
    }
    if gp.lockedm != 0 {
        // Hands off own p to the locked m,
        // then blocks waiting for a new p.
        startlockedm(gp)
        goto top
    }

    execute(gp, inheritTime)
}

```

这里调用了findRunnable函数

这里用死锁给锁死了



而找到runnable的函数里面就是真正选择下一个协程的算法

这个也很长很长

大概是，先从本地协程，再找全局队列，再找就绪的网络协程

我没懂啥叫网络协程

然后进入自旋状态，然后去进行偷取工作




