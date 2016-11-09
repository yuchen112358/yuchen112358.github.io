---
layout:     post
title:      "Android CPU动态调频(下)"
subtitle:   "interactive governor源码分析"
date:       2016-08-28
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

###### 选频函数choose_freq

让我们看一下choose_freq函数：

```c
/*
 * If increasing frequencies never map to a lower target load then
 * choose_freq() will find the minimum frequency that does not exceed its
 * target load given the current load.
 */
static unsigned int choose_freq(struct cpufreq_interactive_cpuinfo *pcpu,
		unsigned int loadadjfreq)
{
	unsigned int freq = pcpu->policy->cur;
	unsigned int prevfreq, freqmin, freqmax;
	unsigned int tl;
	int index;

	freqmin = 0;
	freqmax = UINT_MAX;

	do {
		prevfreq = freq;
		/**
		target_loads使得CPU调整频率来影响当前的CPU workload，促使当前的CPU workload向target_loads靠近. 
		通常，target_loads的值越小，CPU就会越频繁地拉高频率使当前workload低于target_loads. 
		例如：频率小于1G时，取85%；1G—-1.7G，取90%；大于1.7G，取99%。默认值取90%.
		tl即为返回的目标负载。
		*/
		//将频率转换为目标负载
		tl = freq_to_targetload(pcpu->policy->governor_data, freq);

		/*
		 * Find the lowest frequency where the computed load is less
		 * than or equal to the target load.
		 */

		if (cpufreq_frequency_table_target(
			    pcpu->policy, pcpu->freq_table, loadadjfreq / tl,
			    CPUFREQ_RELATION_L, &index))
			break;
		freq = pcpu->freq_table[index].frequency;

		if (freq > prevfreq) {
			/* The previous frequency is too low. */
			freqmin = prevfreq;

			if (freq >= freqmax) {
				/*
				 * Find the highest frequency that is less
				 * than freqmax.
				 */
				 /*
				 找出频率表中小于 freqmax - 1的最大值
				 */
				if (cpufreq_frequency_table_target(
					    pcpu->policy, pcpu->freq_table,
					    freqmax - 1, CPUFREQ_RELATION_H,
					    &index))
					break;
				freq = pcpu->freq_table[index].frequency;

				if (freq == freqmin) {
					/*
					 * The first frequency below freqmax
					 * has already been found to be too
					 * low.  freqmax is the lowest speed
					 * we found that is fast enough.
					 */
					freq = freqmax;
					break;
				}
			}
		} else if (freq < prevfreq) {
			/* The previous frequency is high enough. */
			freqmax = prevfreq;

			if (freq <= freqmin) {
				/*
				 * Find the lowest frequency that is higher
				 * than freqmin.
				 */
				if (cpufreq_frequency_table_target(
					    pcpu->policy, pcpu->freq_table,
					    freqmin + 1, CPUFREQ_RELATION_L,
					    &index))
					break;
				freq = pcpu->freq_table[index].frequency;

				/*
				 * If freqmax is the first frequency above
				 * freqmin then we have already found that
				 * this speed is fast enough.
				 */
				if (freq == freqmax)
					break;
			}
		}

		/* If same frequency chosen as previous then done. */
	} while (freq != prevfreq);

	return freq;
}
```
choose_freq函数用来选频，使选频后的系统workload小于或等于target load.   
核心思想是：选择最小的频率来满足target load.   
影响选频结果的因素有两个：   
1.两次统计idle间的系统频率的平均频率loadadjfreq，   
2.系统设定好的target load，在INIT的时候设定，tunables->target_loads = default_target_loads;   

在一个do-while循环中，进行如下操作:  

* 把上次的freq赋值给prevfreq 
* 通过freq_to_targetload得到target load——tl(目标负载)

在前面讲过：target_loads使得CPU调整频率来影响当前的CPU workload，促使当前的CPU workload向target_loads靠近. 通常，target_loads的值越小，CPU就会越频繁地拉高频率使当前workload低于target_loads. 例如：频率小于1G时，取85%；1G—-1.7G，取90%；大于1.7G，取99%。默认值取90%。

* 然后调用cpufreq_frequency_table_target，取大于等于loadadjfreq / tl　(target freq)的最小值. 
loadadjfreq是两次统计idle间的系统频率的平均频率，除以target load就得到target freq.

```c
设在一个定时周期中两次统计idle（定时开始与结束各统计一次）之间系统运行的总时间为X,
在一个定时周期中两次统计idle之间idle时间为Y，则一个定时周期中两次统计idle之间的active时间为(X - Y)。
loadadjfreq = （X - Y) *  policy->cur / X * 100
```

情景可能是这样的，刚开始freq = pcpu->policy->cur，备份到prevfreq，调用cpufreq_frequency_table_target得到新的freq，然后执行下面的if判断，在这个判断中会调整freq，见下文。 　　

到了下一次循环，上一次的freq又被备份到prevfreq，然后又调用cpufreq_frequency_table_target得到新的freq，如此往复循环，prevfreq和freq的数值会越来越接近，直到相等，就完成了选频. 　　

总体思路是这样，那么来看if判断所做的工作：　

* 拿freq和prevfreq比较：
* 若freq > prevfreq，则 freqmin = prevfreq;否则 freqmax = prevfreq;
* 如果freq > prevfreq，说明比上次大，但是不能比之前的记录最大值大，否则调节就没有意义了，所以如果freq >= freqmax，那么调用cpufreq_frequency_table_target，找小于freqmax的最近一个频点，如果该频点正好是最小频点，说明只有freqmax可以用了，直接break；
* 如果freq < prevfreq，说明比上次小，但是不能比之前记录的最小值小，否则调节就没有意义了，所以如果freq <= freqmin，那么调用cpufreq_frequency_table_target，找大于freqmin的最近一个频点，如果该频点正好是最大频点，直接break；
* 最后返回选好的频点freq。[^ref]

[^ref]:[CPU动态调频：interactive governor如何选频](http://m.blog.csdn.net/article/details?id=45742053)



继续探究cpufreq_interactive_timer:


```c

	if (pcpu->target_freq >= tunables->hispeed_freq &&
	    new_freq > pcpu->target_freq &&
	    now - pcpu->hispeed_validate_time <
	    freq_to_above_hispeed_delay(tunables, pcpu->target_freq)) {
		trace_cpufreq_interactive_notyet(
			data, cpu_load, pcpu->target_freq,
			pcpu->policy->cur, new_freq);
		goto rearm;
	}
```

freq_to_above_hispeed_delay，只是返回了tunables->above_hispeed_delay[i]的数值，我们只设置了一个数值default_above_hispeed_delay.   
重点是这个成员的含义，可以回头看一下INIT阶段的解释.  
如果满足`pcpu->target_freq >= tunables->hispeed_freq && new_freq > pcpu->target_freq &&`，上一次选频频率已经大于tunables->hispeed_freq，本次选频频率比上次更大（系统仍然想增加频率），`now - pcpu->hispeed_validate_time < freq_to_above_hispeed_delay(tunables, pcpu->target_freq))`表示now是本次采样时间戳，pcpu->hispeed_validate_time是上次hispeed生效的时间戳，如果两次时间间隔比above_hispeed_delay小，那么直接goto rearm，不调节频率。　　

```c

	pcpu->hispeed_validate_time = now;
```

更新hispeed_validate_time为now。

```c
	if (cpufreq_frequency_table_target(pcpu->policy, pcpu->freq_table,
					   new_freq, CPUFREQ_RELATION_L,
					   &index))
		goto rearm;

	new_freq = pcpu->freq_table[index].frequency;
```

cpufreq_frequency_table_target函数源码如下：


```c
int cpufreq_frequency_table_target(struct cpufreq_policy *policy,
				   struct cpufreq_frequency_table *table,
				   unsigned int target_freq,
				   unsigned int relation,
				   unsigned int *index)
{
	struct cpufreq_frequency_table optimal = {
		.index = ~0,
		.frequency = 0,
	};
	struct cpufreq_frequency_table suboptimal = {
		.index = ~0,
		.frequency = 0,
	};
	unsigned int i;

	pr_debug("request for target %u kHz (relation: %u) for cpu %u\n",
					target_freq, relation, policy->cpu);

	switch (relation) {
	case CPUFREQ_RELATION_H:
		suboptimal.frequency = ~0;
		break;
	case CPUFREQ_RELATION_L:
		optimal.frequency = ~0;
		break;
	}

	for (i = 0; (table[i].frequency != CPUFREQ_TABLE_END); i++) {
		unsigned int freq = table[i].frequency;
		if (freq == CPUFREQ_ENTRY_INVALID)
			continue;
		if ((freq < policy->min) || (freq > policy->max))
			continue;
		switch (relation) {
		case CPUFREQ_RELATION_H:
			if (freq <= target_freq) {
				if (freq >= optimal.frequency) {
					optimal.frequency = freq;
					optimal.index = i;
				}
			} else {
				if (freq <= suboptimal.frequency) {
					suboptimal.frequency = freq;
					suboptimal.index = i;
				}
			}
			break;
		case CPUFREQ_RELATION_L:
			if (freq >= target_freq) {
				if (freq <= optimal.frequency) {
					optimal.frequency = freq;
					optimal.index = i;
				}
			} else {
				if (freq >= suboptimal.frequency) {
					suboptimal.frequency = freq;
					suboptimal.index = i;
				}
			}
			break;
		}
	}
	if (optimal.index > i) {
		if (suboptimal.index > i)
			return -EINVAL;
		*index = suboptimal.index;
	} else
		*index = optimal.index;

	pr_debug("target is %u (%u kHz, %u)\n", *index, table[*index].frequency,
		table[*index].index);

	return 0;
}
```


`cpufreq_frequency_table_target(pcpu->policy, pcpu->freq_table,new_freq,CPUFREQ_RELATION_L,&index)`即是取freq table中大于或等于new_freq的频率中最小的一个频率，返回index，再由index得到new freq，前面已经得到new freq了，这里为什么要再来一次?因为调频不是连续的，只能取频率表中的若干值。（`CPUFREQ_RELATION_H`表示取即是取freq table中小于或等于new_freq的频率中最大的一个频率）。



```c

	/*
	 * Do not scale below floor_freq unless we have been at or above the
	 * floor frequency for the minimum sample time since last validated.
	 */
	if (new_freq < pcpu->floor_freq) {
		if (now - pcpu->floor_validate_time <
				tunables->min_sample_time) {
			trace_cpufreq_interactive_notyet(
				data, cpu_load, pcpu->target_freq,
				pcpu->policy->cur, new_freq);
			goto rearm;
		}
	}
```

当new_freq < pcpu->floor_freq，并且两次floor_validate_time的间隔小于min_sample_time，此时不需要更新频率.网上有大神说，“在最小抽样周期间隔内，CPU的频率是不会变化的”。

```c
	/*
	 * Update the timestamp for checking whether speed has been held at
	 * or above the selected frequency for a minimum of min_sample_time,
	 * if not boosted to hispeed_freq.  If boosted to hispeed_freq then we
	 * allow the speed to drop as soon as the boostpulse duration expires
	 * (or the indefinite boost is turned off).
	 */

	if (!boosted || new_freq > tunables->hispeed_freq) {
		pcpu->floor_freq = new_freq;
		pcpu->floor_validate_time = now;
	}
```


以上做一些更新数据的工作。

```c

	if (pcpu->policy->cur == new_freq) {
		trace_cpufreq_interactive_already(
			data, cpu_load, pcpu->target_freq,
			pcpu->policy->cur, new_freq);
		goto rearm_if_notmax;
	}
```

```c
rearm_if_notmax:
	/*
	 * Already set max speed and don't see a need to change that,
	 * wait until next idle to re-evaluate, don't need timer.
	 */
	if (pcpu->target_freq == pcpu->policy->max)
		goto exit;
```

如果两次选频频率一样并且上一次选频频率不大于当前频率，那么进入rearm_if_notmax判断是否pcpu->target_freq == pcpu->policy->max，如果相等，那么直接退出，不需要调频，当前频率已经处于max speed。

```c
	trace_cpufreq_interactive_target(data, cpu_load, pcpu->target_freq,
					 pcpu->policy->cur, new_freq);

	pcpu->target_freq = new_freq;
	spin_lock_irqsave(&speedchange_cpumask_lock, flags);
	cpumask_set_cpu(data, &speedchange_cpumask);
	spin_unlock_irqrestore(&speedchange_cpumask_lock, flags);
	wake_up_process(tunables->speedchange_task);
```

###### 调频线程cpufreq_interactive_speedchange_task

将new_freq赋值给target_freq，更新目标频率的数值.   
设置需要调节频率的CPUcore的cpumask,唤醒speedchange_task线程，改变CPU频率。   
speedchange_task的定义如下：

```c
struct cpufreq_interactive_tunables {
......

/* realtime thread handles frequency scaling */
static struct task_struct *speedchange_task;

......
};

```

对应的线程如下：

```c

tunables->speedchange_task =
	kthread_create(cpufreq_interactive_speedchange_task, NULL,
			   speedchange_task_name);

```

cpufreq_interactive_speedchange_task函数定义如下：

```c

static int cpufreq_interactive_speedchange_task(void *data)
{
	unsigned int cpu;
	cpumask_t tmp_mask;
	unsigned long flags;
	struct cpufreq_interactive_cpuinfo *pcpu;

	while (!kthread_should_stop()) {
		set_current_state(TASK_INTERRUPTIBLE);
		spin_lock_irqsave(&speedchange_cpumask_lock, flags);

		if (cpumask_empty(&speedchange_cpumask)) {
			spin_unlock_irqrestore(&speedchange_cpumask_lock,
					       flags);
			schedule();

			if (kthread_should_stop())
				break;

			spin_lock_irqsave(&speedchange_cpumask_lock, flags);
		}

		set_current_state(TASK_RUNNING);
		tmp_mask = speedchange_cpumask;
		cpumask_clear(&speedchange_cpumask);
		spin_unlock_irqrestore(&speedchange_cpumask_lock, flags);

		for_each_cpu(cpu, &tmp_mask) {
			unsigned int j;
			unsigned int max_freq = 0;
#ifdef CONFIG_ARM_EXYNOS_MP_CPUFREQ
			unsigned int smp_id = smp_processor_id();

			if (exynos_boot_cluster == CA7) {
				if ((smp_id == 0 && cpu >= NR_CA7) ||
					(smp_id == NR_CA7 && cpu < NR_CA7))
					continue;
			} else {
				if ((smp_id == 0 && cpu >= NR_CA15) ||
					(smp_id == NR_CA15 && cpu < NR_CA15))
					continue;
			}
#endif

			pcpu = &per_cpu(cpuinfo, cpu);

			if (!down_read_trylock(&pcpu->enable_sem))
				continue;
			if (!pcpu->governor_enabled) {
				up_read(&pcpu->enable_sem);
				continue;
			}

			for_each_cpu(j, pcpu->policy->cpus) {
				struct cpufreq_interactive_cpuinfo *pjcpu =
					&per_cpu(cpuinfo, j);

				if (pjcpu->target_freq > max_freq)
					max_freq = pjcpu->target_freq;
			}

			if (max_freq != pcpu->policy->cur)
				__cpufreq_driver_target(pcpu->policy,
							max_freq,
							CPUFREQ_RELATION_H);
			trace_cpufreq_interactive_setspeed(cpu,
						     pcpu->target_freq,
						     pcpu->policy->cur);

			up_read(&pcpu->enable_sem);
		}
	}

	return 0;
}
```

一个while循环中，遍历speedchange_cpumask相关的CPU，然后再次遍历所有online CPU，得到最大的target_freq，将target_freq赋值给max_freq，即我们需要设置的CPU频率. 
若max_freq != pcpu->policy->cur,说明当前频率不等于我们需要设置的频率，调用`__cpufreq_driver_target`完成频率设置. 
`__cpufreq_driver_target`会调用对应的callback完成频率设置，具体和cpufreq driver相关，需要driver工程师根据自己的平台实现。　　　

**关于`CPUFREQ_RELATION_H／CPUFREQ_RELATION_L`:**　　

`CPUFREQ_RELATION_H`:取小于目标值的最大值；<br>
`CPUFREQ_RELATION_L`取大于目标值的最小值。　　

cpufreq_interactive_timer函数的尾巴：

```c
rearm:
	if (!timer_pending(&pcpu->cpu_timer))
		cpufreq_interactive_timer_resched(pcpu);

exit:
	up_read(&pcpu->enable_sem);
	return;
}
```

**注意：** 定时器在rearm标识处被重新调度：　　

通过调用timer_pending（如果正在等待，将返回 1）来发现计时器是否正在等待（还没有发出），如果不是在等待，则重新调度定时器。在cpufreq_interactive_timer_resched函数中使用了mod_timer_pinned函数来更改已经激活的定时器超时时间并启动定时器。　　

> mod_timer_pinned is a way to update the expire field of an active timer (if the timer is inactive it will be activated) and not allow the timer to be migrated to a different CPU.
>
>mod_timer_pinned(timer, expires) is equivalent to:
>
>del_timer(timer); timer->expires = expires; add_timer(timer);

cpufreq_interactive_timer_resched函数定义如下：

```c

static void cpufreq_interactive_timer_resched(
	struct cpufreq_interactive_cpuinfo *pcpu)
{
	struct cpufreq_interactive_tunables *tunables =
		pcpu->policy->governor_data;
	unsigned long expires;
	unsigned long flags;

	if (!tunables->speedchange_task)
		return;

	spin_lock_irqsave(&pcpu->load_lock, flags);
	pcpu->time_in_idle =
		get_cpu_idle_time(smp_processor_id(),
				  &pcpu->time_in_idle_timestamp,
				  tunables->io_is_busy);
	pcpu->cputime_speedadj = 0;
	pcpu->cputime_speedadj_timestamp = pcpu->time_in_idle_timestamp;
	expires = jiffies + usecs_to_jiffies(tunables->timer_rate);
	mod_timer_pinned(&pcpu->cpu_timer, expires);

	if (tunables->timer_slack_val >= 0 &&
	    pcpu->target_freq > pcpu->policy->min) {
		expires += usecs_to_jiffies(tunables->timer_slack_val);
		mod_timer_pinned(&pcpu->cpu_slack_timer, expires);
	}

	spin_unlock_irqrestore(&pcpu->load_lock, flags);
}
```

------



回顾一下之前的工作，我们分析了interactive governor的创建，初始化。　　<br>
如果CPUFREQ core想要启用interactive governor，就要调用interactive governor提供的interface（cpufreq_governor结构体中定义的函数指针governor，其被初始化为`.governor = cpufreq_governor_interactive,`）。<br>
在这个回调函数governor中，分析了governor在policy方面的初始化，start一个governor，然后调频的工作就交给了定时器（定时器在start governor的时候被启动）。<br>
在定时器中，计算cpu_load，然后根据cpu_load来选频，然后更新pcpu的一些数据，选频得到的频率交由CPUFREQ driver来设置到硬件中去。<br>

顺便说一下：<br>
当一个governor被policy选定后，核心层会通过`__cpufreq_set_policy`函数对该cpu的policy进行设定。如果policy认为这是一个新的governor（和原来使用的旧的governor不相同），policy会通过`__cpufreq_governor`函数，并传递CPUFREQ_GOV_POLICY_INIT参数，而`__cpufreq_governor`函数实际上是调用cpufreq_governor结构中的governor回调函数。　　

核心层会通过`__cpufreq_set_policy`函数，通过CPUFREQ_GOV_POLICY_INIT参数，完成了对governor的初始化工作，紧接着，`__cpufreq_set_policy`会通过CPUFREQ_GOV_START参数，和初始化governor的流程一样启动一个governor。<br>
下面是`__cpufreq_set_policy`函数的定义：

```c

static int __cpufreq_set_policy(struct cpufreq_policy *data,
				struct cpufreq_policy *policy)
{
	int ret = 0, failed = 1;

	pr_debug("setting new policy for CPU %u: %u - %u kHz\n", policy->cpu,
		policy->min, policy->max);

	memcpy(&policy->cpuinfo, &data->cpuinfo,
				sizeof(struct cpufreq_cpuinfo));

	if ((policy->min > data->max || policy->max < data->min) &&
		(policy->max < policy->min)) {
		ret = -EINVAL;
		goto error_out;
	}

	/* verify the cpu speed can be set within this limit */
	ret = cpufreq_driver->verify(policy);
	if (ret)
		goto error_out;

	/* adjust if necessary - all reasons */
	blocking_notifier_call_chain(&cpufreq_policy_notifier_list,
			CPUFREQ_ADJUST, policy);

	/* adjust if necessary - hardware incompatibility*/
	blocking_notifier_call_chain(&cpufreq_policy_notifier_list,
			CPUFREQ_INCOMPATIBLE, policy);

	/* verify the cpu speed can be set within this limit,
	   which might be different to the first one */
	ret = cpufreq_driver->verify(policy);
	if (ret)
		goto error_out;

	/* notification of the new policy */
	blocking_notifier_call_chain(&cpufreq_policy_notifier_list,
			CPUFREQ_NOTIFY, policy);

	data->min = policy->min;
	data->max = policy->max;

	pr_debug("new min and max freqs are %u - %u kHz\n",
					data->min, data->max);

	if (cpufreq_driver->setpolicy) {
		data->policy = policy->policy;
		pr_debug("setting range\n");
		ret = cpufreq_driver->setpolicy(policy);
	} else {
		if (policy->governor != data->governor) {
			/* save old, working values */
			struct cpufreq_governor *old_gov = data->governor;

			pr_debug("governor switch\n");

			/* end old governor */
			if (data->governor) {
				__cpufreq_governor(data, CPUFREQ_GOV_STOP);
				unlock_policy_rwsem_write(policy->cpu);
				__cpufreq_governor(data,
						CPUFREQ_GOV_POLICY_EXIT);
				lock_policy_rwsem_write(policy->cpu);
			}

			/* start new governor */
			data->governor = policy->governor;
			if (!__cpufreq_governor(data, CPUFREQ_GOV_POLICY_INIT)) {
				if (!__cpufreq_governor(data, CPUFREQ_GOV_START)) {
					failed = 0;
				} else {
					unlock_policy_rwsem_write(policy->cpu);
					__cpufreq_governor(data,
							CPUFREQ_GOV_POLICY_EXIT);
					lock_policy_rwsem_write(policy->cpu);
				}
			}

			if (failed) {
				/* new governor failed, so re-start old one */
				pr_debug("starting governor %s failed\n",
							data->governor->name);
				if (old_gov) {
					data->governor = old_gov;
					__cpufreq_governor(data,
							CPUFREQ_GOV_POLICY_INIT);
					__cpufreq_governor(data,
							   CPUFREQ_GOV_START);
				}
				ret = -EINVAL;
				goto error_out;
			}
			/* might be a policy change, too, so fall through */
		}
		pr_debug("governor: change or update limits\n");
		__cpufreq_governor(data, CPUFREQ_GOV_LIMITS);
	}

error_out:
	return ret;
}
```

---

#### 停止Governor

`CPUFREQ_GOV_STOP`　　


现在，回到coufreq_gov_interactive.governor这个callbak，继续向下分析：

```c
	case CPUFREQ_GOV_STOP:
		mutex_lock(&gov_lock);
		//回顾一下：policy->cpus指online的CPU
		for_each_cpu(j, policy->cpus) {
			pcpu = &per_cpu(cpuinfo, j);
			down_write(&pcpu->enable_sem);
			pcpu->governor_enabled = 0;
			del_timer_sync(&pcpu->cpu_timer);
			del_timer_sync(&pcpu->cpu_slack_timer);
			up_write(&pcpu->enable_sem);
		}

		kthread_stop(tunables->speedchange_task);
		put_task_struct(tunables->speedchange_task);
		tunables->speedchange_task = NULL;

		mutex_unlock(&gov_lock);
		break;
```

遍历所有online的cpu： 　　

* 获取cpuinfo 
* 设置pcpu->governor_enabled为0 
* 删除两个定时器

---

#### 更改Governor的上下限值

`CPUFREQ_GOV_LIMITS`  


```c
	case CPUFREQ_GOV_LIMITS:
		if (policy->max < policy->cur)
			__cpufreq_driver_target(policy,
					policy->max, CPUFREQ_RELATION_H);
		else if (policy->min > policy->cur)
			__cpufreq_driver_target(policy,
					policy->min, CPUFREQ_RELATION_L);
		for_each_cpu(j, policy->cpus) {
			pcpu = &per_cpu(cpuinfo, j);

			/* hold write semaphore to avoid race */
			down_write(&pcpu->enable_sem);
			if (pcpu->governor_enabled == 0) {
				up_write(&pcpu->enable_sem);
				continue;
			}

			/* update target_freq firstly */
			if (policy->max < pcpu->target_freq)
				pcpu->target_freq = policy->max;
			else if (policy->min > pcpu->target_freq)
				pcpu->target_freq = policy->min;

			/* Reschedule timer.
			 * Delete the timers, else the timer callback may
			 * return without re-arm the timer when failed
			 * acquire the semaphore. This race may cause timer
			 * stopped unexpectedly.
			 */
			del_timer_sync(&pcpu->cpu_timer);
			del_timer_sync(&pcpu->cpu_slack_timer);
			cpufreq_interactive_timer_start(tunables, j);
			up_write(&pcpu->enable_sem);
		}
		break;
	}
	return 0;
}
```

该event被调用的场景是：change or update limits(即改变频率的最大值/最小值)。 　　

当policy的max或min被改变时，会调用cpufreq_update_policy—>cpufreq_set_policy—>__cpufreq_governor，在__cpufreq_governor中policy->governor->governor调用governor的governor callback。　　

然后执行CPUFREQ_GOV_LIMITS下的代码。此时传入cpufreq_governor_interactive的policy指针已经是min或max被改变后的新policy了 
对于新policy的处理如下：

* 改变当前频率，使其符合新policy的范围 　　
* 遍历所有online CPU： 　　
	+ 判断pcpu->target_freq的值，确保其在新policy的范围内 　　
	+ 删除两个定时器链表
	+ 调用cpufreq_interactive_timer_start,重新add定时器 　　


## 附录

#### cpufreq driver

通过clock framework提供的API，将CPU的频率设置为对应的值。  
通过regulator framework提供的API，将CPU的电压设置为对应的值。

#### cpufreq governor
gov_check_cpu  计算cpu负载的回调函数，通常会直接调用公共层提供的dbs_check_cpu函数完成实际的计算工作。