# 写在前面

主要是深感最近细碎的知识点和事务太多，不写个备忘录，容易忘记，但是专门整理成一个目录，又嫌内容太少，因此，本文档按照每日流水账的模式写。



# 2023/8/14

上周四周五折腾redox的编译问题（编完19个g的代码我是半点不想碰），今天来了解MIT的biscuit小饼干操作系统，按照陈老板的说法是Go语言写的——虽然我连一行helloworld的go我都没写过，但是看看论文，过过编译吧。

（编译过了，没有ide-driver，试图重新编译qemu也找不到这个驱动，不知道哪里去弄，没这驱动真没辙，怀疑有一定可能性是wsl的问题，改天拿host环境试一下）

论文 https://www.usenix.org/system/files/osdi18-cutler.pdf ，状态，待读。

说起来，上次陈老板让我读的那篇文章还没整理完，嗯，今天可以整理完。（done）

但是读完了又产生了新的问题，因为在Nexus操作系统中进行了Linux代码的迁移，直接把原有驱动拿来用了（稍微改了改以适应安全性要求），那么这和我们需要做的事情是类似的。

说起论文，那天看redox的时候，还有一个对于微内核性能的评测，不过很老了，（老意味着经典是吧）

https://os.inf.tu-dresden.de/pubs/sosp97/ 待读。



今日还需要查找Rust for Linux内核代码最新和未来的进展，也不知道内核邮件列表能不能翻出来。

（关于这个东西，翻出来个[Rust abstractions for network device drivers](https://lore.kernel.org/rust-for-linux/20230615191931.4e4751ac@kernel.org/#) 这玩意，我真是醉了，好好地，给自己添什么堵啊）

简单来说，本来LDD小组的工作就有为Rust网络设备，做一层抽象的设计目标，这玩意儿又给了很好的范例，好家伙，当我这一嘴没说。到最后倒霉的不还是我么？？？好吧，反正rust我不熟，看米神的。

关于Rust for Linux内核代码的未来，翻出来是这么说的

他主要放在Rust support这一系列内核邮件列表中（然而这玩意的主线也有好久没有更新了，最新版本的v10的patch好像是去年十月份的事情。额emmm）

他有好几个版本，但是我看了一圈，感觉最重要的还是下面这段话

```
Please note that the Rust support is intended to enable writing
drivers and similar "leaf" modules in Rust, at least for the
foreseeable future. In particular, we do not intend to rewrite
the kernel core nor the major kernel subsystems (e.g. `kernel/`,
`mm/`, `sched/`...). Instead, the Rust support is built on top
of those.
```

反正意思就是，仅仅是给写内核模块提供一个接口工具。

除此之外，我还是不知道，我该怎么订阅内核邮件列表，包括翻邮件列表，虽然网上教程一大堆，但是看上去都比较复杂。更不用说怎么直接发邮件了。

除此除此之外，额，好像还有一大堆其他驱动模块的工作，比如drm，还有puzzleFS这样一些第三方主导的驱动模块。

但是他们都是还只是RFC



好吧，我今天算是知道了原来RFC是Request For Comments的意思。



最后需要做训练营的ppt（拖延症晚期，我现在很慌张。。。）

大概就是今天的工作了（但愿可以按时完成）

其他：姜坤昨天找我问qemu中如何进行控制台交互。

陈庭润那边，训练营的示例代码需要完成。除此之外还需要自己加一个用于基本环境配置和体验。

（我怎么这么菜啊，屁都不会。。。还有一大堆东西要看，还拖延，呜呜呜）



CVE：

大概就是漏洞一个收集机构（？）然后每个系统漏洞都会给个编号



# 2023/8/15

啊先不说别的，让我赶紧肝ppt吧

大概算了一下，需要200页ppt+e1000网卡驱动分析（C和rust版本，对照），但愿今天做的完。



PCIE的msi和msi-x中断

emmm感觉就是为了增加中断。

# 2023/8/16

搞国汽项目的什么文档分拆

找ctr搞训练营的课程作业



# 2023/8/17

昨天陈庭润发了一版结果，上午验证一下

（算了等心情好了再验证吧）

下午先看看周一想读的两篇论文

结果是摸鱼调试了个Makefile

然后了解了一下LFS，大概就是只有个内核是玩不转的，还需要一堆额外的东西，LFS就是加上这些Linux下的环境如何配置的，从而构建一个完整可用的最小的基本Linux系统。

给个链接 https://linux.cn/lfs/LFS-BOOK-7.7-systemd/index.html

# 2023/8/18

上午读go实现操作系统那篇文章，同时在看他们的项目文档。（见Paper目录下读论文的进度。。。反正不指望一天能读完就是了，明天继续吧）

下午看之前看到的LFS，大概介绍一下

第一是建立一个挂载的目录，然后，在该目录下用宿主机编译好binutils，gcc等一堆东西。这里面需要两遍编译，第一遍用宿主机上面的去编译，第二遍用编译好的自编译。哎，bootstrap确实太难了。

第二步是chroot到该挂载的目录，用之前编译的gcc等东西去安装glibc等一堆东西，然后编译gcc，（交叉编译情况下这尤其正常），在此基础上，做了一堆测试

自编译环境是最重要的，其他的其实问题不大，只要有编译器，其他可以自己编译就完事了。