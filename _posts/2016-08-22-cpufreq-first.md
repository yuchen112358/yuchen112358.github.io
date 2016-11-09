---
layout:     post
title:      "Android CPU动态调频(上)"
subtitle:   "interactive governor源码分析"
date:       2016-08-22
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


## interactive governor结构体的定义

```c
struct cpufreq_governor cpufreq_gov_interactive = {
	.name = "interactive",
	.governor = cpufreq_governor_interactive,
	.max_transition_latency = 10000000,//10000000 nano secs = 10 ms
	.owner = THIS_MODULE,
};

```

## cpufreq_governor的定义

```c
//cpufreq.h
struct cpufreq_governor {
	char	name[CPUFREQ_NAME_LEN];
	int	initialized;
	int	(*governor)	(struct cpufreq_policy *policy,
				 unsigned int event);
	ssize_t	(*show_setspeed)	(struct cpufreq_policy *policy,
					 char *buf);
	int	(*store_setspeed)	(struct cpufreq_policy *policy,
					 unsigned int freq);
	unsigned int max_transition_latency; /* HW must be able to switch to
			next freq faster than this value in nano secs or we
			will fallback to performance governor */
	struct list_head	governor_list;
	struct module		*owner;
};

```

* name：governor的名字，此处被赋值为interactive 
* initialized：初始化标志位 
* governor：这个calback用于控制governor的行为，比较重要
* max_transition_latency：硬件从当前频率切换到下一个频率时所用的时间必须比max_transition_latency规定的时间小，否则governor将切换到performance.该数值以纳秒为单位 
* governor_list：所有注册的governor都会被add到这个链表里面 

## 在governor正常工作之前所需做的初始化工作

```c
static int __init cpufreq_interactive_init(void)
{
	unsigned int i;
	struct cpufreq_interactive_cpuinfo *pcpu;

	/* Initalize per-cpu timers */
	for_each_possible_cpu(i) {
		pcpu = &per_cpu(cpuinfo, i);
		init_timer_deferrable(&pcpu->cpu_timer);
		pcpu->cpu_timer.function = cpufreq_interactive_timer;
		pcpu->cpu_timer.data = i;
		init_timer(&pcpu->cpu_slack_timer);
		pcpu->cpu_slack_timer.function = cpufreq_interactive_nop_timer;
		spin_lock_init(&pcpu->load_lock);
		init_rwsem(&pcpu->enable_sem);
	}

	spin_lock_init(&speedchange_cpumask_lock);
	mutex_init(&gov_lock);

	return cpufreq_register_governor(&cpufreq_gov_interactive);
}
```
1. 遍历可能的CPU 
2. get到每个CPU的cpuinfo成员 
3. 初始化可延时定时器 
4. 设置定时器的function，定时器超时时会调用该函数 
5. 设置定时器的data，这里表示CPU ID 
6. 初始化slack定时器 
7. 设置该定时器的function，定时器超时时会调用该函数 
8. 初始化spin_lock 类型的load_lock
9. 初始化可读信号量 
10. 调用cpufreq_register_governor注册interactive governor

## 注册interactive governor的函数cpufreq_register_governor

```c
//cpufreq.c
int cpufreq_register_governor(struct cpufreq_governor *governor)
{
	int err;

	if (!governor)
		return -EINVAL;

	if (cpufreq_disabled())
		return -ENODEV;

	mutex_lock(&cpufreq_governor_mutex);

	governor->initialized = 0;
	err = -EBUSY;
	if (__find_governor(governor->name) == NULL) {
		err = 0;
		list_add(&governor->governor_list, &cpufreq_governor_list);
	}

	mutex_unlock(&cpufreq_governor_mutex);
	return err;
}
```

> `cpufreq_governor_list用来保存已注册的governor`;  
> `__find_governor会在cpufreq_governor_list中遍历寻找是否有与需要register的governor重名的governor，如果没有则将该governor添加到cpufreq_governor_list中。`

以上便是interactive governor的定义，初始化和注册步骤。  

**********

现在我们已经拥有了一个interactive governor，cpufreq core如果想操作governor进行选频，那么interactive governor必须对外提供一个interface以供调用，这就是cpufreq_governor结构体中的governor callback，下面来以这个interface为切入点分析governor是如何工作的。   

```Markdown
The governor->governor callback is called with the current (or to-be-set)
cpufreq_policy struct for that CPU, and an unsigned int event. The
following events are currently defined:
CPUFREQ_GOV_POLICY_INIT:POLICY初始化
CPUFREQ_GOV_POLICY_EXIT：POLICY卸载
CPUFREQ_GOV_START:   This governor shall start its duty for the CPU
             policy->cpu
CPUFREQ_GOV_STOP:    This governor shall end its duty for the CPU
             policy->cpu
CPUFREQ_GOV_LIMITS:  The limits for CPU policy->cpu have changed to
             policy->min and policy->max.
```			 

## 详细分析cpufreq_governor_interactive函数

由cpufreq_gov_interactive定义`.governor = cpufreq_governor_interactive,`可知，governor callback被定义为cpufreq_governor_interactive函数。cpufreq_governor_interactive函数过长，下面进行分段分析：

```c
static int cpufreq_governor_interactive(struct cpufreq_policy *policy,
		unsigned int event)
{
	int rc;
	unsigned int j;
	struct cpufreq_interactive_cpuinfo *pcpu;
	struct cpufreq_frequency_table *freq_table;
	struct cpufreq_interactive_tunables *tunables;
	struct sched_param param = { .sched_priority = MAX_RT_PRIO-1 };
	char speedchange_task_name[TASK_NAME_LEN];
```

pcpu变量描述了cpu相关信息，结构体定义如下：

```c
struct cpufreq_interactive_cpuinfo {
	struct timer_list cpu_timer;
	struct timer_list cpu_slack_timer;
	spinlock_t load_lock; /* protects the next 4 fields */
	u64 time_in_idle;
	u64 time_in_idle_timestamp;
	u64 cputime_speedadj;
	u64 cputime_speedadj_timestamp;
	struct cpufreq_policy *policy;
	struct cpufreq_frequency_table *freq_table;
	unsigned int target_freq;
	unsigned int floor_freq;
	u64 floor_validate_time;
	u64 hispeed_validate_time;
	struct rw_semaphore enable_sem;
	int governor_enabled;
};
```

freq_table表示频率表，结构体定义如下：

```c
//cpufreq.h
struct cpufreq_frequency_table {
	unsigned int	index;     /* any */
	unsigned int	frequency; /* kHz - doesn't need to be in ascending
				    * order */
};
```

可以发现cpufreq_frequency_table是一个node，每个node代表一个频率点，很多node关联在一起就成了一个table。  

让我们继续看cpufreq_governor_interactive函数，struct cpufreq_interactive_tunables *tunables;这个结构体很重要，贯穿了整个governor callback，先给出结构体，接下来在函数中边看边分析。

```c
struct cpufreq_interactive_tunables {
	int usage_count;
	/* Hi speed to bump to from lo speed when load burst (default max) */
	unsigned int hispeed_freq;
	/* Go to hi speed when CPU load at or above this value. */
#define DEFAULT_GO_HISPEED_LOAD 99
	unsigned long go_hispeed_load;
	/* Target load. Lower values result in higher CPU speeds. */
	spinlock_t target_loads_lock;
	unsigned int *target_loads;
	int ntarget_loads;
	/*
	 * The minimum amount of time to spend at a frequency before we can ramp
	 * down.
	 */
#define DEFAULT_MIN_SAMPLE_TIME (80 * USEC_PER_MSEC)
	unsigned long min_sample_time;
	/*
	 * The sample rate of the timer used to increase frequency
	 */
	unsigned long timer_rate;
	/*
	 * Wait this long before raising speed above hispeed, by default a
	 * single timer interval.
	 */
	spinlock_t above_hispeed_delay_lock;
	unsigned int *above_hispeed_delay;
	int nabove_hispeed_delay;
	/* Non-zero means indefinite speed boost active */
	int boost_val;
	/* Duration of a boot pulse in usecs */
	int boostpulse_duration_val;
	/* End time of boost pulse in ktime converted to usecs */
	u64 boostpulse_endtime;
	/*
	 * Max additional time to wait in idle, beyond timer_rate, at speeds
	 * above minimum before wakeup to reduce speed, or -1 if unnecessary.
	 */
#define DEFAULT_TIMER_SLACK (4 * DEFAULT_TIMER_RATE)
	int timer_slack_val;
	bool io_is_busy;

#define TASK_NAME_LEN 15
	/* realtime thread handles frequency scaling */
	struct task_struct *speedchange_task;

	/* handle for get cpufreq_policy */
	unsigned int *policy;
};

```

继续看cpufreq_governor_interactive函数:

```c
if (have_governor_per_policy())
		tunables = policy->governor_data;
	else
		tunables = common_tunables;
```		

have_governor_per_policy判断是否每个policy都有自己的governor，ODROID-XU3中policy并非都采用interactive，所以这里tuables被赋值为policy->governor_data。  
common_tunables的定义如下：

```c
/* For cases where we have single governor instance for system */
struct cpufreq_interactive_tunables *common_tunables;
```
在此，只是定义，并没有分配内存和初始化。

继续看cpufreq_governor_interactive函数:

```c
switch (event) {
	case CPUFREQ_GOV_POLICY_INIT:
		if (have_governor_per_policy()) {
			WARN_ON(tunables);
		} else if (tunables) {
			tunables->usage_count++;
			policy->governor_data = tunables;
			return 0;
		}

		tunables = kzalloc(sizeof(*tunables), GFP_KERNEL);
		if (!tunables) {
			pr_err("%s: POLICY_INIT: kzalloc failed\n", __func__);
			return -ENOMEM;
		}

		if (!tuned_parameters[policy->cpu]) {
			tunables->above_hispeed_delay = default_above_hispeed_delay;
			tunables->nabove_hispeed_delay =
				ARRAY_SIZE(default_above_hispeed_delay);
			tunables->go_hispeed_load = DEFAULT_GO_HISPEED_LOAD;
			tunables->target_loads = default_target_loads;
			tunables->ntarget_loads = ARRAY_SIZE(default_target_loads);
			tunables->min_sample_time = DEFAULT_MIN_SAMPLE_TIME;
			tunables->timer_rate = DEFAULT_TIMER_RATE;
			tunables->boostpulse_duration_val = DEFAULT_MIN_SAMPLE_TIME;
			tunables->timer_slack_val = DEFAULT_TIMER_SLACK;
		} else {
			memcpy(tunables, tuned_parameters[policy->cpu], sizeof(*tunables));
			kfree(tuned_parameters[policy->cpu]);
		}
		tunables->usage_count = 1;

		/* update handle for get cpufreq_policy */
		tunables->policy = &policy->policy;

		spin_lock_init(&tunables->target_loads_lock);
		spin_lock_init(&tunables->above_hispeed_delay_lock);

		rc = sysfs_create_group(get_governor_parent_kobj(policy),
				get_sysfs_attr());
		if (rc) {
			kfree(tunables);
			return rc;
		}

		change_sysfs_owner(policy);

		if (!policy->governor->initialized) {
			idle_notifier_register(&cpufreq_interactive_idle_nb);
			cpufreq_register_notifier(&cpufreq_notifier_block,
					CPUFREQ_TRANSITION_NOTIFIER);
		}

		policy->governor_data = tunables;
		if (!have_governor_per_policy())
			common_tunables = tunables;

		break;
```
判断event的类型，并根据event进行不同的操作。  
在include/linux/cpufreq.h中定义了几种Governor Events：

```c
#define CPUFREQ_GOV_START	1
#define CPUFREQ_GOV_STOP	2
#define CPUFREQ_GOV_LIMITS	3
#define CPUFREQ_GOV_POLICY_INIT	4
#define CPUFREQ_GOV_POLICY_EXIT	5
```

---

#### 初始化GOV_POLICY

`CPUFREQ_GOV_POLICY_INIT`  

CPUFREQ_GOV_POLICY_INIT表示要init governor policy. 
首先判断have_governor_per_policy()，前面分析过了，返回false，并且tunables并没有被分配内存，所以执行下一条语句为tunables分配内存：

```c
tunables = kzalloc(sizeof(*tunables), GFP_KERNEL);
```

接下来就是对tunables的初始化：

```c		
if (!tuned_parameters[policy->cpu]) {
	tunables->above_hispeed_delay = default_above_hispeed_delay;
	tunables->nabove_hispeed_delay =
		ARRAY_SIZE(default_above_hispeed_delay);
	tunables->go_hispeed_load = DEFAULT_GO_HISPEED_LOAD;
	tunables->target_loads = default_target_loads;
	tunables->ntarget_loads = ARRAY_SIZE(default_target_loads);
	tunables->min_sample_time = DEFAULT_MIN_SAMPLE_TIME;
	tunables->timer_rate = DEFAULT_TIMER_RATE;
	tunables->boostpulse_duration_val = DEFAULT_MIN_SAMPLE_TIME;
	tunables->timer_slack_val = DEFAULT_TIMER_SLACK;
} else {
	memcpy(tunables, tuned_parameters[policy->cpu], sizeof(*tunables));
	kfree(tuned_parameters[policy->cpu]);
}
tunables->usage_count = 1;

/* update handle for get cpufreq_policy */
tunables->policy = &policy->policy;

spin_lock_init(&tunables->target_loads_lock);
spin_lock_init(&tunables->above_hispeed_delay_lock);
```		

* usage_count表示引用计数，初始化的时候设置为1
* above_hispeed_delay，内核文档： /Documention/cpu-freq/governors.txt

> above_hispeed_delay: When speed is at or above hispeed_freq, wait for
this long before raising speed in response to continued high load.
The format is a single delay value, optionally followed by pairs of
CPU speeds and the delay to use at or above those speeds.  Colons can
be used between the speeds and associated delays for readability.  For
example:
>
>   80000 1300000:200000 1500000:40000
>
> uses delay 80000 uS until CPU speed 1.3 GHz, at which speed delay
200000 uS is used until speed 1.5 GHz, at which speed (and above)
delay 40000 uS is used.  If speeds are specified these must appear in
ascending order.  Default is 20000 uS.

当CPU频率大于等于hispeed_freq，并且此时workload仍在不停增加（continued high load），系统将等待一个above_hispeed_delay的时间。above_hispeed_delay一般是这样一种格式，一个单个的延时数值，后面跟上一组由CPU speeds 和 delay组成的数组，由冒号隔开。例如：  
80000 1300000:200000 1500000:40000  
当频率低于1.3G时，above_hispeed_delay的值取80000，1.3G到1.5G之间取20000，大于1.5G取40000.默认取20000us.如果频率被指定，那么这些数值必须必须是升序的。  

```c
#define DEFAULT_TIMER_RATE (20 * USEC_PER_MSEC)
#define DEFAULT_ABOVE_HISPEED_DELAY DEFAULT_TIMER_RATE
static unsigned int default_above_hispeed_delay[] = {
	DEFAULT_ABOVE_HISPEED_DELAY };
//include/linux/time.h
#define USEC_PER_MSEC	1000L
```

可以看到default_above_hispeed_delay是一个数组，我的环境下只有一个数值20000，above_hispeed_delay的数值就是20000(20*1000)。

* nabove_hispeed_delay:default_above_hispeed_delays数组中元素的个数。
* go_hispeed_load: The CPU load at which to ramp to hispeed_freq.Default is 99%. 
高频阈值。当系统的负载超过该值，升频，否则降频。

```c
#define DEFAULT_GO_HISPEED_LOAD 99
```

调频的时候会用到这个数值，见后文。

By the way, hispeed_freq: Hi speed to bump to from lo speed when load burst (default max) 
当workload达到 go_hispeed_load时，频率将被拉高到这个值，默认的大小由policy来决定。

> hispeed_freq: An intermediate "hi speed" at which to initially ramp
when CPU load hits the value specified in go_hispeed_load.  If load
stays high for the amount of time specified in above_hispeed_delay,
then speed may be bumped higher.  Default is the maximum speed
allowed by the policy at governor initialization time.

* target_loads:使得CPU调整频率来影响当前的CPU workload，促使当前的CPU workload向target_loads靠近.通常，target_loads的值越小，CPU就会越频繁地拉高频率使当前workload低于target_loads.  
例如：频率小于1G时，取85%；1G—-1.7G，取90%；大于1.7G，取99%。默认值取90%.

> target_loads: CPU load values used to adjust speed to influence the
current CPU load toward that value.  In general, the lower the target
load, the more often the governor will raise CPU speeds to bring load
below the target.  The format is a single target load, optionally
followed by pairs of CPU speeds and CPU loads to target at or above
those speeds.  Colons can be used between the speeds and associated
target loads for readability.  For example:
>
>   85 1000000:90 1700000:99
>
>targets CPU load 85% below speed 1GHz, 90% at or above 1GHz, until
1.7GHz and above, at which load 99% is targeted.  If speeds are
specified these must appear in ascending order.  Higher target load
values are typically specified for higher speeds, that is, target load
values also usually appear in an ascending order. The default is
target load 90% for all speeds.

```c
/* Target load.  Lower values result in higher CPU speeds. */
#define DEFAULT_TARGET_LOAD 90
static unsigned int default_target_loads[] = {DEFAULT_TARGET_LOAD};
```

* ntarget_loads：target_loads的个数。

* min_sample_time：最小采样时间，默认是80000us。


> The minimum amount of time to spend at the current
frequency before ramping down. Default is 80000 uS.

* boostpulse_duration_val:

> boost: If non-zero, immediately boost speed of all CPUs to at least
hispeed_freq until zero is written to this attribute.  If zero, allow
CPU speeds to drop below hispeed_freq according to load as usual.
Default is zero.
>
> boostpulse: On each write, immediately boost speed of all CPUs to
hispeed_freq for at least the period of time specified by
boostpulse_duration, after which speeds are allowed to drop below
hispeed_freq according to load as usual.
>
> boostpulse_duration: Length of time to hold CPU speed at hispeed_freq
on a write to boostpulse, before allowing speed to drop according to
load as usual.  Default is 80000 uS.

boost即超频，操作方法是：

```bash
echo 1 > /sys/devices/system/cpu/cpufreq/interactive/boost
```

此时会立即将所有CPU的频率提高到至少hispeed_freq.写入0时，根据workload降低频率.默认为0.  
boostpulse，每次触发boost功能时，立即拉高所有CPU的频率到hispeed_freq并保持在该频率至少boostpulse_duration的时间，在这段时间以后，根据当前的workload，频率才允许被降低。  
boostpulse_duration：默认值80000 uS.ODROID-XU3采用的便是默认值，如下所示：

```c
#define DEFAULT_MIN_SAMPLE_TIME (80 * USEC_PER_MSEC)
#define USEC_PER_MSEC	1000L
```

* timer_rate和timer_slack_val:当CPU不处于idel状态时，timer_rate作为采样速率来计算CPU的workload. 当CPU处于idel状态，此时使用一个可延时定时器，会导致CPU不能从idel状态苏醒来响应定时器. 定时器的最大的可延时时间用timer_slack表示，默认值80000 uS.此处采用默认值，如下所示：

```c
#define DEFAULT_TIMER_RATE (20 * USEC_PER_MSEC)
#define USEC_PER_MSEC	1000L
#define DEFAULT_TIMER_SLACK (4 * DEFAULT_TIMER_RATE)
```

> timer_rate: Sample rate for reevaluating CPU load when the CPU is not
idle.  A deferrable timer is used, such that the CPU will not be woken
from idle to service this timer until something else needs to run.
(The maximum time to allow deferring this timer when not running at
minimum speed is configurable via timer_slack.)  Default is 20000 uS.
> 
> timer_slack: Maximum additional time to defer handling the governor
sampling timer beyond timer_rate when running at speeds above the
minimum.  For platforms that consume additional power at idle when
CPUs are running at speeds greater than minimum, this places an upper
bound on how long the timer will be deferred prior to re-evaluating
load and dropping speed.  For example, if timer_rate is 20000uS and
timer_slack is 10000uS then timers will be deferred for up to 30msec
when not at lowest speed.  A value of -1 means defer timers
indefinitely at all speeds.  Default is 80000 uS.

继续向下看继续看cpufreq_governor_interactive函数:

```c

spin_lock_init(&tunables->target_loads_lock);
spin_lock_init(&tunables->above_hispeed_delay_lock);

rc = sysfs_create_group(get_governor_parent_kobj(policy),
		get_sysfs_attr());
if (rc) {
	kfree(tunables);
	return rc;
}

change_sysfs_owner(policy);

if (!policy->governor->initialized) {
	idle_notifier_register(&cpufreq_interactive_idle_nb);
	cpufreq_register_notifier(&cpufreq_notifier_block,
			CPUFREQ_TRANSITION_NOTIFIER);
}

policy->governor_data = tunables;
if (!have_governor_per_policy())
	common_tunables = tunables;

break;
```

* 初始化tunables结构体中的两个自旋锁. 
* 将tunables指针赋值给policy->governor_data 
* 将tunables指针赋值给common_tunables，这个全局变量会在一些文件的show和store函数中被调用。


`rc = sysfs_create_group(get_governor_parent_kobj(policy),get_sysfs_attr());`中的get_governor_parent_kobj和get_sysfs_attr定义如下：

```c
//get_governor_parent_kobj
static struct kobject *get_governor_parent_kobj(struct cpufreq_policy *policy)
{
	if (have_governor_per_policy())
		return &policy->kobj;
	else
		return cpufreq_global_kobject;
}

//get_sysfs_attr
static struct attribute_group *get_sysfs_attr(void)
{
	if (have_governor_per_policy())
		return &interactive_attr_group_gov_pol;
	else
		return &interactive_attr_group_gov_sys;
}

/* One Governor instance for entire system */
static struct attribute *interactive_attributes_gov_sys[] = {
	&target_loads_gov_sys.attr,
	&above_hispeed_delay_gov_sys.attr,
	&hispeed_freq_gov_sys.attr,
	&go_hispeed_load_gov_sys.attr,
	&min_sample_time_gov_sys.attr,
	&timer_rate_gov_sys.attr,
	&timer_slack_gov_sys.attr,
	&boost_gov_sys.attr,
	&boostpulse_gov_sys.attr,
	&boostpulse_duration_gov_sys.attr,
	&io_is_busy_gov_sys.attr,
	NULL,
};

static struct attribute_group interactive_attr_group_gov_sys = {
	.attrs = interactive_attributes_gov_sys,
	.name = "interactive",
};

```

把上述代码简化一下得到:

```c
      rc = sysfs_create_group(cpufreq_global_kobject,
                interactive_attr_group_gov_sys);
```

在cpufreq_global_kobject所对应的目录cpufreq下创建一个名为interactive的目录，并创建与之关联的属性文件。通过以下方式可以看到这些属性文件：

```bash
ls /sys/devices/system/cpu/cpufreq/interactive/
```

最后注册了两个notification，分别是idle相关和频率改变相关。

**回顾一下CPUFREQ_GOV_POLICY_INIT所做的工作：   
1. 定义并初始化了一个cpufreq_interactive_tunables结构体，将该结构体指针赋值给policy->governor_data，在struct cpufreq_policy结构体中，policy->governor_data为void *指针，现在我们知道它的作用是指向tunables，而tunablesa对应的内存中存放了governor调节频率的参数，这就是policy->governor_data的作用。   
2. 创建对应的目录和属性文件。**

---

#### 卸载GOV_POLICY_EXIT

`CPUFREQ_GOV_POLICY_EXIT`    

这个event对应的操作比较简单一些，主要是做一些policy和governor的“善后”工作。