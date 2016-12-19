[TOC "float:right"]
## Real-Time Scheduling on Multicore Platforms[^1]
<!--
![Alt Text](http://bit.ly/1drEdWK "Title" "width:30px;height:80px;float:right")
-->
***

#### Introduction
Thermal and power problems limit the performance that singleprocessor chips can deliver. Multicore architectures, or chip
multiprocessors, which include several processors on a single
chip, are being widely touted as a solution to this problem.  

In many proposed multicore platforms, different cores share
either on- or off-chip caches. To effectively exploit the available
parallelism on these platforms, shared caches must not become
performance bottlenecks.  

论文焦点是==“多核平台上实时应用的调度”==。在该论文中，作者将关注的多核体系结构限制为如下模型：  

<div align="center">![Multicore architecture](/images/posts/rtas00.png "Multicore architecture" "width:440px;height:300px")</div>  
<div align="center">Multicore architecture</div>

==注意：==L2 misses对性能的影响远大于L1 misses。这是因为处理L2 miss（需要访问内存）可能需要高达100-300 cycles，而处理L1 miss（通过访问L2）则只需要10-30 cycles。[^2]

* ###### 所研究的问题
Wishing to know whether, in realtime systems, tasks that generate significant memory-to-L2 traffic can be discouraged from being co-scheduled while ensuring real-time constraints.
* ###### 所提出的方法
Two-step process:  
(1) combine tasks that may induce significant memory-to-L2 traffic into groups;  
(2) at runtime, use a scheduling policy that reduces concurrency within groups.  
* ###### Megatask
基于组识别的调度策略是一个基于megatask（巨型任务）概念的分层调度方法。一个megatask
表示为一个任务组，其被当作一个单一调度实体。上层的调度器给一个megatask分配一个或多个处理器，megatask再轮流将处理器分配给它的组件任务。假设γ是一个
megatask，其组件任务的总利用率（处理器利用率）是I + f，其中I是整数部分，f是小数部分（ 0 < f < 1）。这样组件任务需要I到I+1个处理器以满足它们的deadline。在论文中，为避免L2 cache发生抖动（thrashing）,在γ中至多可以并发调度（co-scheduled）I+1个任务。
* ###### 例子
Consider a ==four-core system== in which the objective
is to ensure that the combined working-set size of the tasks
that are ==co-scheduled does not exceed the capacity of the L2
cache==. Let the task set τ be comprised of ==three tasks of weight
(i.e., utilization) 0.6== and with a working-set size of ==200 KB==
(Group A), and ==four tasks of weight 0.3== and with a working-set
size of ==50 KB== (Group B). Let the capacity
of the ==L2 cache be 512 KB==. The total weight of τ is 3(==3\*0.6+4\*0.3==), so co-scheduling ==at least three== of its tasks is unavoidable. However,
since the combined working-set size of the tasks in Group A exceeds the L2 capacity(==3*200>512==), it is desirable that the three co-scheduled
tasks not all be from this group. Because the total utilization of
==Group A is 1.8==, by combining the tasks in Group A into a single
megatask, it can be ensured that ==at most two tasks== from it are
ever co-scheduled.

#### Pfair Scheduling
Pfair scheduling can be used to schedule a periodic,
intra-sporadic (IS), or generalized-intra-sporadic (GIS) task system τ on M ≥ 1 processors. Each task T
of τ is assigned a rational weight wt(T) ∈ (0, 1] that denotes the processor share it requires. For a periodic task T ,
wt(T) = T.e/T.p, where T.e and T.p are the (integral) execution cost and period of T . A task is light if its weight is less
than 1/2, and heavy, otherwise.  
Pfair algorithms allocate processor time in discrete quanta;
the time interval [t, t + 1), where t ∈ N (the set of nonnegative integers), is called slot t.  

<div align="center">![Periodic,IS,GIS task](/images/posts/rtas01.png "Periodic,IS,GIS task" "width:540px;height:300px")</div>  
<div align="center"><font size="4">Periodic,IS,GIS task 示意图</font></div>

==注意：==周期性任务系统也是IS任务系统，进而也是GIS任务系统，所以构建出适合GIS任务的模型也适合于其他任务模型。
* 调度算法
	- Pfair scheduling算法的调度原则：earliest-deadline-first。
	- 以防两个子任务出现相同的deadline，需要使用Tie-breaking rules。
	- 所知的最有效的最优算法是PD^2^，它使用两个tie-breaks。
	- PD^2^ is optimal,it correctly schedules any GIS task system τ for which T ∈τ wt(T ) ≤ M holds.
* ==需要注意：==即使使用最优的PD^2^算法作为第二层调度器，组件任务的deadline也可能得不到保证，为此作者提出了两种方法：1）Reweighting a megatask；2）Tardiness bounds without reweighting。  
(上述两种方法全是理论证明和理论公式，在此就不详细介绍。)

#### Experimental Results
为了获取megatasking在减少cache竞争的效率，作者使用SESC模拟器进行了实验。   

* 所模拟的体系结构由不同数量的核组成，每一个均拥有16K L1
data and instruction caches (4- and 2-way set associative, respectively) with random and LRU replacement policies, respectively, and a shared 8-way set associative 512K on-chip L2
cache with an LRU replacement policy。  
* 每一个cache拥有一个 64-byte line size。  
* 在一个给定的working-set size(WSS)下，每一个调度任务均被赋予一个utilization and memory block。
任务是顺序访问内存块的，且访问到末尾时会再次回到开始处。
* 所有的调度、抢占、迁移开销均在模拟中被考虑进去。  

下面是作者在两种任务集上进行了模拟实验。在两个实验中，均将Pfair
scheduling with megatasks 与 partitioned[^3]
EDF and ordinary Pfair scheduling (without megatasks)进行了比较。

##### Hand-Crafted Task Sets
* 所产生的hand-crafted task sets如下：

<div align="center">![Properties of example task sets](/images/posts/rtas02.png "Properties of example task sets" "width:540px;height:200px")</div>  
<div align="center">Properties of example task sets.</div>

备注：1 quanta = 1 ms.

* 模拟运行结果如下：

<div align="center">![实验结果](/images/posts/rtas03.png "实验结果" "width:540px;height:360px")</div>  
<div align="center">L2 cache miss ratios per task set and (Min., Avg., Max.) per task memory accesses completed, in millions, for example task sets.</div>

在获得这些结果时，megatask并没有进行reweighted，因为此处更关注cache的行为而不是时间属性。
1. BASIC由三个heavy-weight任务组成，运行其中两个不会产生L2 cache抖动，但是运行三个会产生。Partitioning 和 Pfair调度算法使用超过了2个核，产生了抖动，而将所有的任务并进一个megatask中，每次调度最多只会调度2（3\*0.6=1.8,1+1=2)个任务，抖动消失。这个在BASIC中表现的十分明显。  
2. SMALL_BASIC的效果与BASIC相同，但是表现的不如BASIC明显。
3. 将ONE_MEGA视为一个megatask时，一次最多可以调度4（5\*0.7=3.5,3+1=4）个任务，不会产生抖动；而将ONE_MEGA视为2个megatask（2.1和1.4）可能会出现一次调度5（2.1->3,1.4->2,3+2）个任务的情况。
4. 将TWO_MEGA的整体视为一个megatask，可以保证最多一次调度4个任务，但是可能调用的包含三个WSS 190K的任务，这样便会产生L2 cache抖动；若仅仅只是将WSS 190K的三个任务视为一个megatask，三个WSS 60K的任务作为 free task，情况会仅仅好一些（因为会出现 190\*2+60\*3=560>512的情况）。然而将TWO_MEGA分成两个
megatasks, one eachfor 190K & 60K tasks，这样最多只会出现2个WSS 190K和2个WSS 60K的任务共同被调度，不会产生L2 cache抖动（190\*2+60\*2=500<512）。
5. 从相同时间内（从某一行看），每个任务完成的内存访问数来看，megatask的运行结果十分优越。
6. 解释：For example，the number of memory accesses (in millions) for the tasks in SMALL BASIC was
{0.614, 4.123, 0.613, 4.103, 0.613} under partitioning, but
{3.755, 3.765, 3.743, 3.717, 3.723} for megatasking.这样的非均匀的分布导致，在某些案例下，partioning拥有更高的最大完成内存访问数的值。

##### Randomly-Generated Task Sets
* In generating task sets at
random, the authors limited attention to a four-core system, and considered total WSSs of 768K, 896K, and 1024K, which correspond
to 1.5, 1.75, and 2.0 times the size of the L2 cache.
* Total system utilizations were allowed to range between 2.0
and 3.5. 
* Task utilizations were generated uniformly over a range from some specified minimum to one, exclusive. The minimum task utilization was varied from 1/10 (which makes finding a feasible
partitioning easier) to 1/2 (which makes partitioning harder).
* In total, 552 task sets were generated.
* 产生这些任务集之后需要进行包装：
	- 对于partitioning，注意使用首次适用策略（使用两次，第一次的标准依据WSS递减，第二次的标准依据是利用率递减，第一次失败则使用第二次）；
	- 对于megatask，详细见论文，具体原则是“If the current
task could be added to the current megatask without pushing
the megatask’s weight beyond the next integer boundary，then pack it to one.”。
	- 上述两种包装均会产生不合格的任务，不合格任务在实验时将不被包含。
* 在产生megatasks之后，每个megatask将被reweighted。

###### Randomly-Generated Task Sets运行结果
**L2 cache miss rates数据分析如下：**  
使用三种调度算法，每一个合格的任务将被执行20 quanta，记录L2 cache miss rate。

<div align="center">![实验结果](/images/posts/rtas004.png "实验结果")</div>
<div align="center">L2 cache miss rate versus both total system utilization (top) and minimum task utilization (bottom). The different columns correspond
(left to right) to total WSSs of 1.5, 1.75, and 2.0 times the L2 cache capacity, respectively.Each point is an average obtained from between 19 and 48 task sets.</div>

==注意：==较小的L2 cache miss rate不同将会导致很大的性能不同。下图给出了对于图(b)的 cycles-permemory-reference的数量.

<div align="center">![cycles-permemory-reference](/images/posts/rtas05.png "cycles-permemory-reference")</div>  
<div align="center">Cycles-per-memory-reference for the data in (b).</div>

* As seen in the bottom-row plots, the L2 miss rate increases with increasing task utilizations.这是因为heaviest tasks拥有最大的WSSs，因此很难在较少的核上运行。
* The top-row plots show a similar trend as the total system utilization increases from 2.0 to 2.5.
	- 在总系统利用率超过2.5之后，miss rate 开始下降。一个可能的解释：
The task-generation process may leave little room to improve miss rates at total utilizations beyond 2.5.
* 对于总的WSS来说，在1.5倍L2 cache大小下，megatask表现最好，且明显。
在1.75倍、2倍L2 cache大小下，megatask在大多数情况下表现仍然最好，但是不明显。这是因为在1.75倍、2倍L2 cache大小下，这三种调度模式都不能很好的改善L2 cache性能，这在2倍L2 cache大小下体现尤为明显。

**内存访问统计数据（数据不包括调度代码本身）如下：**

<div align="center">![Instructions and memory accesses](/images/posts/realtime00.png "Instructions and memory accesses")</div>  
<div align="center">(Min., Avg., Max.) instructions and memory accesses completed over all non-disqualified task sets for each scheduling policy, in
millions. </div>

* Megatasking is the clear winner by 5-6% on average and by as much as 30% in the worst case (as seen by the minimum values).

#### 下周任务
* 继续阅读与SESC模拟器相关的论文
* 继续学习SESC的使用及部分核心源码的阅读


[^1]:Anderson J H, Calandrino J M, Devi U M C. Real-time scheduling on multicore platforms[C]//Real-Time and Embedded Technology and Applications Symposium, 2006. Proceedings of the 12th IEEE. IEEE, 2006: 179-190.
[^2]:A. Fedorova, M. Seltzer, C. Small, and D. Nussbaum. Performance of multithreaded chip multiprocessors and implications for operating system design. Proc. of the USENIX 2005 Annual Technical Conf., 2005. (See also Technical Report TR-17-04, Div. of Engineering and Applied Sciences, Harvard Univ. Aug., 2004.)
[^3]:A cache-partitioning scheme is presented that uniformly distributes the impact of cache contention among co-scheduled threads.
