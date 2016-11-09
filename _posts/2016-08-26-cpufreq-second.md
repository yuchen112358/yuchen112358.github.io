---
layout:     post
title:      "Android CPU动态调频(中)"
subtitle:   "interactive governor源码分析"
date:       2016-08-26
author:     "yuchen"
header-img: "img/post-bg-java.jpg"
tags:
    - DVFS
    - Governor
    - Android
---

**文章声明**：本文内容主要源自[CPU动态调频：interactive governor](http://m.blog.csdn.net/article/details?id=45697221)一文,并在其中加入了自己的理解。其中选频函数介绍引自[CPU动态调频：interactive governor如何选频](http://m.blog.csdn.net/article/details?id=45742053)一文。本文纠正了原文的一些错误，但是本人很佩服原文作者的代码阅读功底及讲解能力，本着尊崇原著的理念，读者也可直接阅读原文。

本文以Android平台通常采用的interactive governor为例，详细分析了Linux/arm 3.10.9 Kernel中提供的DVFS governor。

**注意：**代码若未指明位置则默认在drivers/cpufreq/cpufreq_interactive.c中。

## 详细分析cpufreq_governor_interactive函数(续)

#### 启动Governor

`CPUFREQ_GOV_START`  

> CPUFREQ_GOV_START:   This governor shall start its duty for the CPU
>		     policy->cpu

启动一个governor，看代码:

```c
	case CPUFREQ_GOV_START:
		mutex_lock(&gov_lock);

		freq_table = cpufreq_frequency_get_table(policy->cpu);
		if (!tunables->hispeed_freq)
			tunables->hispeed_freq = policy->max;

		for_each_cpu(j, policy->cpus) {
			pcpu = &per_cpu(cpuinfo, j);
			pcpu->policy = policy;
			pcpu->target_freq = policy->cur;
			pcpu->freq_table = freq_table;
			pcpu->floor_freq = pcpu->target_freq;
			pcpu->floor_validate_time =
				ktime_to_us(ktime_get());
			pcpu->hispeed_validate_time =
				pcpu->floor_validate_time;
			down_write(&pcpu->enable_sem);
			del_timer_sync(&pcpu->cpu_timer);
			del_timer_sync(&pcpu->cpu_slack_timer);
			cpufreq_interactive_timer_start(tunables, j);
			pcpu->governor_enabled = 1;
			up_write(&pcpu->enable_sem);
		}

		snprintf(speedchange_task_name, TASK_NAME_LEN, "cfinteractive%d\n",
					policy->cpu);

		tunables->speedchange_task =
			kthread_create(cpufreq_interactive_speedchange_task, NULL,
				       speedchange_task_name);
		if (IS_ERR(tunables->speedchange_task)) {
			mutex_unlock(&gov_lock);
			return PTR_ERR(tunables->speedchange_task);
		}

		sched_setscheduler_nocheck(tunables->speedchange_task, SCHED_FIFO, &param);
		get_task_struct(tunables->speedchange_task);

#ifdef CONFIG_ARM_EXYNOS_MP_CPUFREQ
		kthread_bind(tunables->speedchange_task, policy->cpu);
#endif

		/* NB: wake up so the thread does not look hung to the freezer */
		wake_up_process(tunables->speedchange_task);

		mutex_unlock(&gov_lock);
		break;
```

首先获取freq_table.  
如果没有设置hispeed_freq的值的话，就设置hispeed_freq为policy->max，和之前介绍hispeed_freq时说的一样.  
接下来是一个for循环，policy->cpus表示所有处于online状态的CPU，for循环遍历所有处于online状态的CPU，在这个循环中：  
get到cpu的cpuinfo结构体并把指针赋值给pcpu，一个struct cpufreq_interactive_cpuinfo结构体指针. 
然后对pcpu的一些成员进行初始化，本质上还是设置online cpus的cpuinfo结构体成员. 
然后调用cpufreq_interactive_timer_start启动相关的定时器 .
启动定时器以后governor就可以工作了，所以设置pcpu->governor_enabled为1.

下面是policy的类型定义：

```c
struct cpufreq_policy {
	/* CPUs sharing clock, require sw coordination */
	cpumask_var_t		cpus;	/* Online CPUs only */
	cpumask_var_t		related_cpus; /* Online + Offline CPUs */

	unsigned int		shared_type; /* ACPI: ANY or ALL affected CPUs
						should set cpufreq */
	unsigned int		cpu;    /* cpu变量记录着用于管理该policy的cpu编号 */
	/*last_cpu则是上一次管理该policy的cpu编号（因为管理policy的cpu可能会被plug out，这时候就要把管理工作迁移到另一个cpu上）*/
	unsigned int		last_cpu; /* cpu nr of previous CPU that managed
					   * this policy */
	struct cpufreq_cpuinfo	cpuinfo;/* see above */

	unsigned int		min;    /* in kHz */
	unsigned int		max;    /* in kHz */
	unsigned int		cur;    /* in kHz, only needed if cpufreq
					 * governors are used */
	unsigned int		policy; /* see above */
	struct cpufreq_governor	*governor; /* see below */
	void			*governor_data;

	struct work_struct	update; /* if update_policy() needs to be
					 * called, but you're in IRQ context */

	struct cpufreq_real_policy	user_policy;

	struct kobject		kobj;
	struct completion	kobj_unregister;
};

```

```c
#define for_each_cpu(cpu, mask)			\
	for ((cpu) = 0; (cpu) < 1; (cpu)++, (void)mask)
```

###### 定时器cpufreq_interactive_timer_start函数

```c

/* The caller shall take enable_sem write semaphore to avoid any timer race.
 * The cpu_timer and cpu_slack_timer must be deactivated when calling this
 * function.
 */
static void cpufreq_interactive_timer_start(
	struct cpufreq_interactive_tunables *tunables, int cpu)
{
	struct cpufreq_interactive_cpuinfo *pcpu = &per_cpu(cpuinfo, cpu);
	unsigned long expires = jiffies +
		usecs_to_jiffies(tunables->timer_rate);
	unsigned long flags;

	if (!tunables->speedchange_task)
		return;

	pcpu->cpu_timer.expires = expires;
	add_timer_on(&pcpu->cpu_timer, cpu);
	if (tunables->timer_slack_val >= 0 &&
	    pcpu->target_freq > pcpu->policy->min) {
		expires += usecs_to_jiffies(tunables->timer_slack_val);
		pcpu->cpu_slack_timer.expires = expires;
		add_timer_on(&pcpu->cpu_slack_timer, cpu);
	}

	spin_lock_irqsave(&pcpu->load_lock, flags);
	pcpu->time_in_idle =
		get_cpu_idle_time(cpu, &pcpu->time_in_idle_timestamp,
				  tunables->io_is_busy);
	pcpu->cputime_speedadj = 0;
	pcpu->cputime_speedadj_timestamp = pcpu->time_in_idle_timestamp;
	spin_unlock_irqrestore(&pcpu->load_lock, flags);
}
```

在进入cpufreq_interactive_timer_start之前有一些deactive的操作：

```c
del_timer_sync(&pcpu->cpu_timer);
del_timer_sync(&pcpu->cpu_slack_timer);
```

cpufreq_interactive_timer_start究竟做的工作：

* 设置定时器的到期时间expire 
* 调用add_timer_on添加定时器，”start a timer on a particular CPU” 。即在指定的CPU上start一个定时器，例如ODROID-XU3上有4个小CPU和4个大CPU，那么将有4个定时器被添加到pcpu->cpu_timer链表中（一个governor只能控制四个小核或四个大核）
* cpu_slack_timer也是同样的操作 
* 然后获取该CPU的idle时间，这个数值在统计更新时间的时候会被用到。

```c
pcpu->time_in_idle = get_cpu_idle_time(cpu, &pcpu->time_in_idle_timestamp,
                                            tunables->io_is_busy);
```

然后调用：

```c
    pcpu->cputime_speedadj = 0;
    pcpu->cputime_speedadj_timestamp = pcpu->time_in_idle_timestamp;
```

time_in_idle_timestamp的数值在get_cpu_idle_time函数中被更新，在代码中形参的名字为last_update_time，可以理解为更新time_in_idle的时间戳。网上有人解释为计算机启动到现在的时间，是一样的。

到这里start governor的工作就完成了，主要就是启动了两个定时器，定时器到期的话，会执行相关的操作最终选定要set的频率。   
本来到这里我们应该回到cpufreq_governor_interactive中分析event为CPUFREQ_GOV_LIMITS的情况。   
但是为了思路的流畅性，我们顺着定时器继续追代码，看定时器如何实现选频：

---

`__init cpufreq_interactive_init`函数中：

```c
 cpu->cpu_timer.function = cpufreq_interactive_timer;
 pcpu->cpu_timer.data = i;
```

定时器到期时，调用 cpufreq_interactive_timer,这里data是cpu的索引号，在cpufreq_interactive_init中cpu_timer的data成员被赋值成为CPU的索引号，之后调用cpu_timer.function的时候作为实参,分段看cpufreq_interactive_timercpufreq_interactive_timer函数：

```c
static void cpufreq_interactive_timer(unsigned long data)
{
	u64 now;
	unsigned int delta_time;
	u64 cputime_speedadj;
	int cpu_load;
	struct cpufreq_interactive_cpuinfo *pcpu =
		&per_cpu(cpuinfo, data);
	struct cpufreq_interactive_tunables *tunables =
		pcpu->policy->governor_data;
	unsigned int new_freq;
	unsigned int loadadjfreq;
	unsigned int index;
	unsigned long flags;
	bool boosted;

	if (!down_read_trylock(&pcpu->enable_sem))
		return;
	if (!pcpu->governor_enabled)
		goto exit;

	spin_lock_irqsave(&pcpu->load_lock, flags);
	now = update_load(data);
	delta_time = (unsigned int)(now - pcpu->cputime_speedadj_timestamp);
	cputime_speedadj = pcpu->cputime_speedadj;
	spin_unlock_irqrestore(&pcpu->load_lock, flags);

	if (WARN_ON_ONCE(!delta_time))
		goto rearm;

	do_div(cputime_speedadj, delta_time);
	loadadjfreq = (unsigned int)cputime_speedadj * 100;
	cpu_load = loadadjfreq / pcpu->target_freq;
	boosted = tunables->boost_val || now < tunables->boostpulse_endtime;
```


###### CPU负载的计算

首先调用update_load，更新工作负载：

```c
static u64 update_load(int cpu)
{
	struct cpufreq_interactive_cpuinfo *pcpu = &per_cpu(cpuinfo, cpu);
	struct cpufreq_interactive_tunables *tunables =
		pcpu->policy->governor_data;
	u64 now;
	u64 now_idle;
	unsigned int delta_idle;
	unsigned int delta_time;
	u64 active_time;

	now_idle = get_cpu_idle_time(cpu, &now, tunables->io_is_busy);
	delta_idle = (unsigned int)(now_idle - pcpu->time_in_idle);
	delta_time = (unsigned int)(now - pcpu->time_in_idle_timestamp);

	if (delta_time <= delta_idle)
		active_time = 0;
	else
		active_time = delta_time - delta_idle;

	pcpu->cputime_speedadj += active_time * pcpu->policy->cur;

	update_cpu_metric(cpu, now, delta_idle, delta_time, pcpu->policy);

	pcpu->time_in_idle = now_idle;
	pcpu->time_in_idle_timestamp = now;
	return now;
}
```

* now_idle:系统启动以后运行的idle的总时间

> the cummulative idle time (since boot) for a given CPU, in microseconds.

* pcpu->time_in_idle：上次统计时的idle的总时间 
* delta_idle：两次统计之间的idle总时间

* now：本次的update time，即本次统计idle时的时间戳

> variable to store update time in.

* pcpu->time_in_idle_timestamp，上次统计idle时的时间戳 
* delta_time：两次统计idle之间系统运行的总时间

若delta_time <= delta_idle，说明运行期间CPU一直在idle，active_time赋值为0. 　　
否则，active_time = delta_time - delta_idle;计算出两次统计之间CPU处于active的总时间。　　

然后更新pcpu的一些成员变量的值: 

* `pcpu->cputime_speedadj` 这个数值的计算方式（`pcpu->cputime_speedadj += active_time * pcpu->policy->cur;`）即是本身加上`active_time * pcpu->policy->cur`，表示从定时开始到当前（可认为是定时结束时）的激活时间与当前频率值的乘积（在定时器启动函数`cpufreq_interactive_timer_start`中`pcpu->cputime_speedadj`被赋值为０，且在定时器重新调度函数`cpufreq_interactive_timer_resched`中又被重新赋值为０，所以这里用`+=`符号表示的任然是在一个定时周期中的激活时间与当前频率值的乘积)。
* `pcpu->time_in_idle = now_idle;`　：更新系统启动后运行的总idle时间。
* `pcpu->time_in_idle_timestamp = now;`：更新统计时的时间戳。

上面这两个数值被更新留作下次update_load使用。


下面是update_cpu_metric函数：

```c
void update_cpu_metric(int cpu, u64 now, u64 delta_idle, u64 delta_time,
		       struct cpufreq_policy *policy)
{
	struct cpu_load *pcpuload = &per_cpu(cpuload, cpu);
	unsigned int load;

	/*
	 * Calculate the active time in the previous time window
	 *
	 * load = active time / total_time * 100
	 */
	if (delta_time <= delta_idle)
		load = 0;
	else
		load = div64_u64((100 * (delta_time - delta_idle)), delta_time);

	pcpuload->load = load;
	pcpuload->frequency = policy->cur;
	pcpuload->last_update = now;
#ifdef CONFIG_CPU_THERMAL_IPA_DEBUG
	trace_printk("cpu_load: cpu: %d freq: %u load: %u\n", cpu, policy->cur, load);
#endif
}
```

---

回到cpufreq_interactive_timer，update_load返回了最新一次统计idle时的时间戳，赋值给now。　　

```c
delta_time = (unsigned int)(now - pcpu->cputime_speedadj_timestamp);
```
这里用的cputime_speedadj_timestamp，在函数cpufreq_interactive_timer_resched和cpufreq_interactive_timer_start中发现cputime_speedadj_timestamp都被赋值为time_in_idle_timestamp，而start函数或reschedule函数都是在定时器刚启动或重启时执行的，所以cputime_speedadj_timestamp是定时器函数启动或重启时调用get_cpu_idle_time函数的时间戳，这即表示了定时每次启动时统计idle的时间戳，所以now - pcpu->cputime_speedadj_timestamp即表示了两次统计idle之间系统运行的总时间。　　

整个定时的时间获取过程描述如下：<br>
定时器每次重新启动（包括第一次启动，对应于cpufreq_interactive_timer_start函数和cpufreq_interactive_timer_resched函数），都通过get_cpu_idle_time函数获取当前系统启动以后运行的idle的总时间（记录在time_in_idle变量中），并记录下本次获取idle的时间戳（保存在time_in_idle_timestamp中，并再将其赋值给cputime_speedadj_timestamp变量）。当定时器计时结束，就会执行cpufreq_interactive_timer函数，在该函数中会重新调用get_cpu_idle_time函数获取当前系统启动以后运行的idle的总时间，通过计算其与定时器启动时得到的idle的总时间的差值来进一步获得定时期间的active时间与当前频率的乘积（保存在cputime_speedadj变量中），并用这次get_cpu_idle_time函数时的时间戳来更新time_in_idle_timestamp变量,然后将该时间戳赋值给now变量。那么:<br>
两次调用get_cpu_idle_time函数（定时器启动和结束时各调用一次）间的时间间隔delta_time=now-cputime_speedadj_timestamp即表示了两次统计idle之间系统运行的总时间。

计算本次统计到定时器启动的运行时间。　　

然后取pcpu->cputime_speedadj赋值给局部变量cputime_speedadj，cpu->cputime_speedadj在update_load中已被计算并更新过了。  

接下来的几行代码都是用来计算cpu_load，把这些数值展开看就变得很清晰了：

```
	/*pcpu->cputime_speedadj 这个数值表示
	一个定时周期中的激活时间与当前频率值的乘积*/
	cputime_speedadj = pcpu->cputime_speedadj;
	do_div(cputime_speedadj, delta_time);
	loadadjfreq = (unsigned int)cputime_speedadj * 100;
	cpu_load = loadadjfreq / pcpu->target_freq;
	
	／*
	do_div(x,y);
	结果保存在x中；余数保存在返回结果中。
	*/
```
对上述过程的理解如下：<br>
设在一个定时周期中两次统计idle（定时开始与结束各统计一次）之间系统运行的总时间为X,在一个定时周期中两次统计idle之间idle时间为Y，则一个定时周期中两次统计idle之间的active时间为(X - Y)。

```c
cputime_speedadj = active_time * policy->cur = （X - Y) *  policy->cur
cputime_speedadj /= delta_time = （X - Y) *  policy->cur / X
loadadjfreq = cputime_speedadj * 100 = （X - Y) *  policy->cur / X * 100
cpu_load = loadadjfreq / pcpu->target_freq = （X - Y) *  policy->cur / X / pcpu->target_freq * 100
= [(X - Y) / X] * [policy->cur / pcpu->target_freq] * 100
= (1 - Y / X) * [policy->cur / pcpu->target_freq] * 100
```

(1 - X / Y) 即表示两次统计间CPU处于非idle的时间比例，policy->cur / pcpu->target_freq 表示当前频率占目标频率的比例。　　

为什么要乘以100?　这是因为内核不支持浮点运算。

影响cpu_load的两个因素：

1. idle时间 
2. 当前频率/目标频率

有一个疑问：   
cpufreq_interactive_timer函数的目的是为了根据当前的workload选频，得到目标频率，然后传给cpufreq driver来设置频率。如果已经有了目标频率，那么直接调driver设置好了，所以这里的pcpu->target_freq不是本次选频得到的target_freq。  

在cpufreq_interactive_timer的后面代码中，我们看到:

```c
pcpu->target_freq = new_freq;

```

new_freq 是本次选频后得到的新频率，最后赋值给pcpu->target_freq，所以在cpufreq_interactive_timer中，该赋值语句之前的所有pcpu->target_freq都表示是上一次选频的target_freq。

所以更正一下，影响cpu_load的两个因素:

1. idle时间 
2. 当前频率/上一次选频频率

疑惑：当前频率/上一次选频频率看起来好像值为１，因为上次选频结果不就是当前频率吗？<br>
然而当前频率/上一次选频频率的值并不为１，因为从频率设定函数cpufreq_interactive_speedchange_task中可知，最终将CPU的频率设置为所有CPU的pcpu->target_freq值中最大的那一个，所以当前频率/上一次选频频率的值应该大于等于１。

现在，记住`pcpu->target_freq都表示是上一次选频的目标频率`这一点。带着这个思路继续分析：　　

当cpu_load大于tunables->go_hispeed_load或者tunables->boosted的值为非0，此时我们需要拉高频率。如果上一次选频频率比tunables->hispeed_freq小，那么直接设置new_freq为tunables->hispeed_freq;如果上一次选频频率不小于tunables->hispeed_freq，调用choose_freq函数选频，若选频后仍然达不到tunables->hispeed_freq，那么直接设置new_freq为tunables->hispeed_freq。 　　　

可以看到，当cpu_load大于等于tunables->go_hispeed_load时，new_freq的频率要不小于tunables->hispeed_freq。　　

当cpu_load小于tunables->go_hispeed_load并且tunables->boosted的值为0，调用choose_freq选频。 　

```c

	if (cpu_load >= tunables->go_hispeed_load || boosted) {
		if (pcpu->target_freq < tunables->hispeed_freq) {
			new_freq = tunables->hispeed_freq;
		} else {
			new_freq = choose_freq(pcpu, loadadjfreq);

			if (new_freq < tunables->hispeed_freq)
				new_freq = tunables->hispeed_freq;
		}
	} else {
		new_freq = choose_freq(pcpu, loadadjfreq);
	}

```