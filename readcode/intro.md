## 概述
### 关于 jemalloc
在对称多处理器(SMP)成为主流，多线程变成越来越广泛的情况下，内存分配在很多时候
成为制约程序性能的重要因素之一。
Jason Evans 开发的 jemalloc 就是为了提供一个可扩展的并发内存分配器。
在 http://people.freebsd.org/~jasone/jemalloc/bsdcan2006/jemalloc.pdf 中，
Jason Evans 解释了 jemalloc 的主要考虑和核心设计。
(http://blog.csdn.net/stillingpb/article/details/50937366 是该论文的中文翻译)

下面总结一下文章中关于 jemalloc 设计的一些考虑：
* 更好的缓存局部性：首先尝试内存最小化原则，在不违背该原则的前提下，
尝试进行连续分配，从而得到更好的缓存局部性，因为短时间内分配出来的对象
往往会被一起使用
* 避免 false sharing：cache 的 false sharing 特性会导致严重的性能下降，
需要使用填充对齐的方法避免这一问题。填充的方式会产生碎片，而 jemalloc 的
arena、bin 等设计可以减少碎片产生
* 减少锁竞争：引入 arena 的设计，大幅减少锁的竞争 

下面总结一下文章中关于 jemalloc 实现的一些特点：
* 单处理器一个arena，多处理器处理核数四倍的arena
* 使用循环方式为线程分配arena：使得每个arena被分配大致相同数量的线程
* 使用 pthread 库的 TSD 机制实现线程本地数据存储：线程需要存储一些本地的数据，
高效的本地数据存储对某些功能的实现十分重要
* 内存分类：对内存按照尺寸分类，分为 small,large,huge。small,large,huge 中又
划分成很多子类。按照尺寸分类对管理内存、减少碎片都有很大的帮助
* run 分配器数据和应用数据分离：减少分配器数据被破坏的可能；增加数据访问的
局部性；避免头部尺寸不对齐带来的碎片
* 为每种尺寸提供多个run，并按使用量状态分类：按照使用量状态分类并选出下一个
当前run，这样可以规避 run 的频繁创建、销毁 (jemalloc-4.2.1中不是这样实现的，
而是按照低地址优先的方法选取 run)

上述文章是十年前(2006年)写的，现在的 jemalloc 在此基础上又引入了一些新的特性、
算法，但是大致思想、设计还是基本一致的。当前 jemalloc 的具体实现可以看后面
的源码分析。

jemalloc 借鉴了大量其他内存分配器的优点，并且经过精心的设计、优秀的实现，
性能十分优异，已经被用在了很多项目及公司内，比如 FreeBSD, Firefox, Redis, 
Facebook 等。

### 关于本文档
本文档是对 jemalloc-4.2.1 源码的说明，是本人在阅读、学习 jemalloc 源码的时候的总结。
由于时间、精力有限，我只看了初始化、malloc、free的过程，其他的过程没有细看，
并且内存分析（prof）没有看，所以文档中这些内容不会涉及。

本文档的最新版本可以在 https://github.com/leebaok/jemalloc-4.2.1-readcode 获得，
该仓库中还有一份添加了注释的源码，建议将文档与代码结合起来阅读，这样学习起来
更加透彻，而且可以使用 GDB 对源码进行跟踪，了解每一个细节，关于GDB跟踪 jemalloc，
可以参见附录。

下面附上本文档的更新说明：
* readcode-0.1 : 
	- 完成 jemalloc 执行流程初稿
* readcode-0.2 : 
	- 将 jemalloc 执行流程切分成多个子过程
* readcode-0.3 : 
	- 为重要数据结构添加配图
* readcode-0.4 :
	- 为重要执行流程添加配图
	- 在文档中添加重要过程的源码解释

### 声明
由于本人水平有限，不保证本文档中的内容全是正确的，如果你发现有地方有问题，可以在
https://github.com/leebaok/jemalloc-4.2.1-readcode 中提交 issue 或者 pull request，
谢谢！

