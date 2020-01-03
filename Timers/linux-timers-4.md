리눅스 커널의 타이머 및 시간 관리. Part 4.
================================================================================

타이머
--------------------------------------------------------------------------------

이것은 리눅스 커널에서 타이머와 시간 관리 관련 내용을 설명하는 [챕터](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)의 네 번째 파트이며, 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-3.html)에서 우리는 리눅스 커널의 `tick broadcast` 프레임워크와 `NO_HZ` 모드를 알았습니다. 이번 파트에서는 리눅스 커널의 시간 관리 관련 사항에 대해 계속 살펴보고 리눅스 커널의 또 다른 개념- `timers`에 대해 알아볼 것입니다. 리눅스 커널의 타이머에 대해서 보기 전에, 우리는 이 개념에 대해 몇몇 이론을 배워야 합니다. 이 파트에서는 소프트웨어 타이머를 고려할 것이라는 것을 알아두세요.

리눅스 커널은 커널 함수가 나중에 실행될 수 있도록 `software timer` 개념을 제공합니다. 타이머는 리눅스 커널에 전반적으로 사용됩니다. 예를 들어 net/netfilter/ipset/ip_set_list_set.c 소스 코드 파일을 들여다봅시다. 이 소스 코드 파일은 [IP](https://en.wikipedia.org/wiki/Internet_Protocol)주소 그룹의 관리를 위하여 이 프레임워크의 구현을 제공합니다.

우리는 이 소스 코드 파일에서 `gc`가 들어 있는 `list_set` 구조체를 찾을 수 있습니다.

```C
struct list_set {
	...
	struct timer_list gc;
	...
};
```

`gc`가 `timer_list` 타입을 가지고 있는 것은 아닙니다. 이 구조체는 [include/linux/timer.h](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98eb93d973/include/linux/timer.h) 헤더 파일에 정의되어 있으며 이 구조체의 주요 목적은 `dynamic` 타이머를 리눅스 커널에 저장하는 것입니다. 사실 리눅스 커널은 동적 타이머(dynamic timer)와 간격 타이머(interval timer)라는 두 가지 유형의 타이머를 제공합니다. 첫 번째 타이머 유형은 커널에서 사용되고 두 번째 유형은 사용자 모드에서 사용될 수 있습니다. `timer_list` 구조체에는 실제 `dynamic` 타이머가 포함되어 있습니다. `list_set`은 `gc` 타이머를 가지고 있으며 우리의 예시에서는 가비지 콜렉션을 위한 타이머를 나타냅니다. 이 타이머는 `list_set_gc_init` 함수에서 초기화됩니다.

```C
static void
list_set_gc_init(struct ip_set *set, void (*gc)(unsigned long ul_set))
{
	struct list_set *map = set->data;
	...
	...
	...
	map->gc.function = gc;
	map->gc.expires = jiffies + IPSET_GC_PERIOD(set->timeout) * HZ;
	...
	...
	...
}
```

`gc` 포인터가 가리키는 함수는 `map->gc. expires`와 동일한 타임아웃 후 호출됩니다.

자, 우리는 이번 예시에서 [netfilter](https://en.wikipedia.org/wiki/Netfilter)에 대해 자세히 설명하지 않겠습니다. 이 장은 [network](https://en.wikipedia.org/wiki/Computer_network)과 관련된 내용이 아니기 때문입니다. 하지만 우리는 타이머가 리눅스 커널에서 널리 사용되고 있다는 것을 알게 되었고, 타이머가 미래에 함수를 호출할 수 있는 개념을 나타낸다는 것을 알게 되었습니다.

이제 이전 챕터에서 했던 것처럼 타이머와 시간 관리와 관련된 리눅스 커널의 소스 코드를 계속 연구해 보겠습니다. 

리눅스 커널의 동적 타이머 개요
--------------------------------------------------------------------------------

제가 이미 적었듯이, 우리는 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-3.html)를 통해 `tick broadcast` 프레임워크와 `NO_HZ` 모드에 대해 알았습니다. 이들은 [init/main.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98d973/init/main.c) 소스 코드 파일에서 `tick_init` 함수를 호출하여 초기화됩니다. 이 소스 코드 파일을 보면, 다음 시간 관리 관련 함수가:

```C
init_timers();
```

임을 확인할 수 있습니다. 이 함수는 [kernel/time/timer.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/timer.c) 소스 코드 파일에 정의되어 있으며 다음 4가지 함수의 호출을 포함합니다.

```C
void __init init_timers(void)
{
	init_timer_cpus();
	init_timer_stats();
	timer_register_cpu_notifier();
	open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
}
```

각 함수의 구현을 살펴보겠습니다. 첫 번째 함수는 [동일한](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/timer.c) 소스 코드 파일에 정의된 `init_timer_cpus`이며 그저 시스템에서 '사용가능'(possible)한 각 프로세서에 대해 `init_timer_cpu` 함수를 호출합니다.

```C
static void __init init_timer_cpus(void)
{
	int cpu;

	for_each_possible_cpu(cpu)
		init_timer_cpu(cpu);
}
```

`possible`이 무엇인지 모르거나 기억하지 못하는 경우, 리눅스 커널에서 `cpumask` 개념을 설명하는 이 책의 특별 [파트](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)를 읽어보세요. 간단히 말해서, `possible` 프로세서는 시스템 수명(the life of the system) 동안 언제든지 plug in 될 수 있는 프로세서입니다.

`init_timer_cpu` 함수는 각 프로세서에 대해 `tvec_base` 구조체의 초기화를 실행합니다. 이 구조체는 [kernel/time/timer.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98d973/kernel/time/c) 소스 코드 파일에 정의되어 있으며 특정 프로세서의 `dynamic` 타이머와 관련된 데이터를 저장합니다. 이 구조체의 정의를 살펴보겠습니다.

```C
struct tvec_base {
	spinlock_t lock;
	struct timer_list *running_timer;
	unsigned long timer_jiffies;
	unsigned long next_timer;
	unsigned long active_timers;
	unsigned long all_timers;
	int cpu;
	bool migration_enabled;
	bool nohz_active;
	struct tvec_root tv1;
	struct tvec tv2;
	struct tvec tv3;
	struct tvec tv4;
	struct tvec tv5;
} ____cacheline_aligned;
```

`thec_base` 구조체에는 다음 필드가 포함됩니다: `tvec_base` 보호를 위한 `lock`, 다음으로 특정 프로세서의 현재 실행중인 타이머를 가리키는 `running_timer` 필드, 가장 이른 만료 시간을 나타내는 `timer_jiffies` 필드(이것은 리눅스 커널이 이미 만료된 타이머를 찾는 데 사용됩니다). 다음 필드- `next_timer`에는 프로세서가 절전 모드로 전환되고 리눅스 커널에서 `NO_HZ` 모드가 활성화된 경우 다음 타이머 [인터럽트](https://en.wikipedia.org/wiki/Interrupt)에 대한 다음 보류(pending) 타이머가 포함되어 있습니다. `active_timers` 필드는 지연 불가능한(non-deferrable) 타이머에 대한 설명을 제공하고 이는 즉 프로세서가 절전 모드로 전환된 동안 모든 타이머는 멈추지 않는다는 것입니다. `all_timers` 필드에서는 타이머의 총수 또는 '`active_timers` + 지연 가능한 타이머'를 추적합니다. `cpu` 필드는 타이머를 소유한 프로세서의 번호를 나타냅니다. `migration_enabled` 및 `nohz_active` 필드는 타이머를 다른 프로세서로 마이그레이션할 수 있는지의 여부와 `NO_HZ` 모드의 상태를 나타냅니다.

`tvec_base` 구조체의 마지막 5개 필드는 동적 타이머 목록을 나타냅니다. 첫 번째 `tv1` 필드는 다음 타입을 가지고 있습니다:

```C
#define TVR_SIZE (1 << TVR_BITS)
#define TVR_BITS (CONFIG_BASE_SMALL ? 6 : 8)

...
...
...

struct tvec_root {
	struct hlist_head vec[TVR_SIZE];
};
```

`TVR_SIZE`의 값은 비활성화되었을 경우 커널 데이터 구조의 크기를 줄이는 `CONFIG_BASE_SMALL` 커널 구성 옵션에 따라 달라짐에 유의하세요:

![base small](http://i68.tinypic.com/aylkt2.png)

`v1`은  `64` 또는 `256` 요소를 포함할 수 있는 배열이며 각 요소는 다음 `255` 시스템 타이머 인터럽트 내에서 소멸되는 동적 타이머를 나타냅니다. 다음 3개 필드인 `tv2`, `tv3`, `tv4` 역시 동적 타이머 목록이지만, 각각 다음 `2^14 - 1`, `2^20 - 1`, `2^26`에 소멸되는 동적 타이머를 저장합니다. 마지막 `tv5` 필드는 만료 기간이 긴 동적 타이머를 저장하는 목록을 나타냅니다.

자, 우리는 `tvec_base`의 구조체와 구조체의 각 필드에 대한 설명을 보았으며, 이제 `init_timer_cpu` 함수의 구현을 살펴볼 수 있습니다. 제가 이미 쓴 것처럼, 이 함수는 [kernel/time/timer.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98eb93d973/kernel/time/c) 소스 코드 파일에 정의되어있으며  `tvec_bases`의 초기화를 실행합니다.

```C
static void __init init_timer_cpu(int cpu)
{
	struct tvec_base *base = per_cpu_ptr(&tvec_bases, cpu);

	base->cpu = cpu;
	spin_lock_init(&base->lock);

	base->timer_jiffies = jiffies;
	base->next_timer = base->timer_jiffies;
}
```

`tvec_bases`는 주어진 프로세서의 동적 타이머에 대한 메인 자료구조를 나타내는 [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) 변수를 나타냅니다. 이 `per-cpu` 변수는 동일한 소스 코드 파일에 정의되어 있습니다.

```C
static DEFINE_PER_CPU(struct tvec_base, tvec_bases);
```

가장 먼저 우리는 주어진 프로세서의 `tvec_bases`의 주소를 `base` 변수에 얻어오고, 이를 얻는 대로 `init_timer_cpu` 함수의 `tvec_base` 필드 일부를 초기화하기 시작합니다.  [jiffies](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html)으로 `per-cpu` 동적 타이머를 초기화한 후 가능한(possible) 프로세서 수를 `init_timer_stats` 함수에서 `tstats_lookup_lock` [spinlock](https://en.wikipedia.org/wiki/Spinlock)으로 초기화해야 합니다.

```C
void __init init_timer_stats(void)
{
	int cpu;

	for_each_possible_cpu(cpu)
		raw_spin_lock_init(&per_cpu(tstats_lookup_lock, cpu));
}
```

`tstats_lookcup_lock` 변수는 `per-cpu` raw spinlock을 나타냅니다.

```C
static DEFINE_PER_CPU(raw_spinlock_t, tstats_lookup_lock);
```

이것은 [procfs](https://en.wikipedia.org/wiki/Procfs)을 통해 액세스할 수 있는 타이머 통계를 사용하여 작업을 보호하는 데 사용됩니다.

```C
static int __init init_tstats_procfs(void)
{
	struct proc_dir_entry *pe;

	pe = proc_create("timer_stats", 0644, NULL, &tstats_fops);
	if (!pe)
		return -ENOMEM;
	return 0;
}
```

예를 들어:

```
$ cat /proc/timer_stats
Timerstats sample period: 3.888770 s
  12,     0 swapper          hrtimer_stop_sched_tick (hrtimer_sched_tick)
  15,     1 swapper          hcd_submit_urb (rh_timer_func)
   4,   959 kedac            schedule_timeout (process_timeout)
   1,     0 swapper          page_writeback_init (wb_timer_fn)
  28,     0 swapper          hrtimer_stop_sched_tick (hrtimer_sched_tick)
  22,  2948 IRQ 4            tty_flip_buffer_push (delayed_work_timer_fn)
  ...
  ...
  ...
```

`tstats_lookup_lock` 스핀록 초기화 후 다음 단계는 `timer_register_cpu_notifier` 함수의 호출입니다. 이 함수는 `CONFIG_HOTPLUG_CPU` 커널 구성 옵션에 따라 달라지며 이 옵션은 Linux 커널에서 [hotplug](https://en.wikipedia.org/wiki/Hot_swapping) 프로세서를 지원을 활성화합니다.

프로세서가 논리적으로 오프라인 상태가 될 경우, `cpu_notifier` 매크로의 호출에 의해 `CPU_DEAD` 또는 `CPU_DEAD_FROZEN` 이벤트로 Linux 커널에 알림이 전송됩니다.

```C
#ifdef CONFIG_HOTPLUG_CPU
...
...
static inline void timer_register_cpu_notifier(void)
{
	cpu_notifier(timer_cpu_notify, 0);
}
...
...
#else
...
...
static inline void timer_register_cpu_notifier(void) { }
...
...
#endif /* CONFIG_HOTPLUG_CPU */
```

이 경우 `timer_cpu_notify`가 호출되어 이벤트 유형을 확인하고 `migrate_timers` 함수를 호출합니다.

```C
static int timer_cpu_notify(struct notifier_block *self,
	                        unsigned long action, void *hcpu)
{
	switch (action) {
	case CPU_DEAD:
	case CPU_DEAD_FROZEN:
		migrate_timers((long)hcpu);
		break;
	default:
		break;
	}

	return NOTIFY_OK;
}
```

이 챕터에서는 리눅스 커널 소스 코드의 `hotplug` 관련 이벤트에 대해 설명하지 않을 것이지만, 이러한 것들에 관심이 있으시면 [time/time/timer.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/timer.c) 소스 코드 파일에서 `migrate_timers` 함수의 구현을 찾아볼 수 있습니다.

`init_timers` 함수의 마지막 단계는 다음 함수의 호출입니다:

```C
open_softirq(TIMER_SOFTIRQ, run_timer_softirq);
```

리눅스 커널의 인터럽트 및 인터럽트 처리에 대한 9번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-9.html)를 읽어본 경우 `open_softirq` 함수는 이미 익숙할 것입니다. 간단히 말하자면, [kernel/softirq.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98d973/softirq.c) 소스 코드 파일에 정의된 `open_softirq` 함수는 지연 인터럽트 핸들러의 초기화를 실행합니다.

우리의 경우 지연된 함수는 `run_timer_softirq` 함수이며, [arch/x86/kernel/irq.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98eb93d973/arch/x86/irq.c) 소스 코드 파일에 정의된 `do_IRQ` 함수에서 하드웨어 인터럽트를 수행한 후 호출됩니다. 이 함수의 핵심은 소프트웨어 동적 타이머를 처리하는 것입니다. 왜냐하면 이것은 시간이 소모되는 작업이기에 리눅스 커널은 하드웨어 타이머 인터럽트를 처리하는 동안 이러한 일을 하지 않기 때문입니다.

`run_timer_softirq` 함수의 구현에 대해 살펴봅시다:

```C
static void run_timer_softirq(struct softirq_action *h)
{
	struct tvec_base *base = this_cpu_ptr(&tvec_bases);

	if (time_after_eq(jiffies, base->timer_jiffies))
		__run_timers(base);
}
```

`run_timer_softirq` 함수의 첫 부분에서 우리는 현재 프로세서에 대한 `dynamic` 타이머를 얻고  현재 구조체에 대한 `timer_jiffies`를 통해 [jiffies](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html)의 현재 값과 비교합니다. 이는 `time_after_eq` 매크로의 호출을 통해 이루어지며 매크로는 [include/linux/jiffies.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/jiffies.h)헤더 파일에 정의되어 있습니다.

```C
#define time_after_eq(a,b)          \
    (typecheck(unsigned long, a) && \
     typecheck(unsigned long, b) && \
    ((long)((a) - (b)) >= 0))
```

`tvec_base` 구조체의 `timer_jiffies` 필드는 지정된 타이머에 의해 지연된 함수가 실행될 상대 시간을 나타냅니다. 그래서 이 두 가지 값을 비교해 보고, `jiffies`로 대표되는 현재의 시간이 `base->timer_jiffies`보다 크면, 같은 소스 코드 파일에 정의된 `__run_timers` 함수를 호출합니다. 이 함수의 구현을 살펴보겠습니다.

제가 방금 쓴 것처럼 `__run_timers` 함수는 특정 프로세서에 대해 만료된 타이머를 모두 실행합니다. 이 함수는 `tvec_base` 구조체를 보호하기 위한 `tvec_base's` 잠금을 획득하는 것으로 시작합니다.

```C
static inline void __run_timers(struct tvec_base *base)
{
	struct timer_list *timer;

	spin_lock_irq(&base->lock);
	...
	...
	...
	spin_unlock_irq(&base->lock);
}
```

이 후 루프가 시작되고 여기서 `timer_jiffies`는 `jiffies`보다 크지 않습니다.

```C
while (time_after_eq(jiffies, base->timer_jiffies)) {
	...
	...
	...
}
```

우리는 루프에서 많은 다양한 조작을 찾을 수 있지만, 핵심은 만료된 타이머를 찾아 지연된 기능을 호출한다는 것입니다. 우선 처리될 다음 타이머를 저장할 `base->tv1` 목록의 `index`를 다음 식으로 계산해야 합니다.

```C
index = base->timer_jiffies & TVR_MASK;
```

여기서 `TVR_MASK`는 `tvec_root->vec` 원소를 얻기 위한 마스크입니다. 다음으로 처리되어야 할 타이머와 함께 index를 얻었으면, 그 값을 확인합니다. index가 0이면 캐스케이드 테이블(cascade table)인 `tv2`, `tv3` 등에 있는 모든 목록을 살펴보고, `cascade` 함수의 호출을 통해 이를 rehashing 합니다.

```C
if (!index &&
	(!cascade(base, &base->tv2, INDEX(0))) &&
		(!cascade(base, &base->tv3, INDEX(1))) &&
				!cascade(base, &base->tv4, INDEX(2)))
		cascade(base, &base->tv5, INDEX(3));
```

그 후에는 `base->timer_jiffies`의 값을 증가시킵니다.

```C
++base->timer_jiffies;
```

마지막으로 다음 루프의 목록에서 각 타이머에 해당하는 함수를 실행합니다.

```C
hlist_move_list(base->tv1.vec + index, head);

while (!hlist_empty(head)) {
	...
	...
	...
	timer = hlist_entry(head->first, struct timer_list, entry);
	fn = timer->function;
	data = timer->data;

	spin_unlock(&base->lock);
	call_timer_fn(timer, fn, data);
	spin_lock(&base->lock);

	...
	...
	...
}
```

여기서 `call_timer_fn` 그냥 지정된 함수를 호출하기만 합니다:

```C
static void call_timer_fn(struct timer_list *timer, void (*fn)(unsigned long),
	                      unsigned long data)
{
	...
	...
	...
	fn(data);
	...
	...
	...
}
```

이것으로 끝입니다. 리눅스 커널에는 이제 `dynamic timers`를 위한 기반이 있습니다. 우리는 이 흥미로운 주제에 대해서 깊이 생각하지 않을 것입니다. 제가 이미 썼듯 `timers`는 리눅스 커널에서 [폭넓게](http://lxr.free-electrons.com/ident?i=timer_list) 사용되는 개념이고 한 파트 분량이나 두 파트 분량으로는 어떻게 구현하고 어떻게 작동하는지 이해하지 못할 것입니다. 하지만 우리는 이제 이 개념에 대해, 왜 리눅스 커널의 내부에 그것이 필요한지 그리고 그것과 관련된 몇가지 자료구조를 알고 있습니다.

이제 리눅스 커널에서 `dynamic timers`의 사용법을 살펴보겠습니다.

동적 타이머의 사용
--------------------------------------------------------------------------------

이미 알고계시겠지만, 리눅스 커널이 어떤 개념을 제공하면 이 개념을 관리할 수 있는 API도 제공하며 `dynamic timers` 개념도 예외는 아닙니다. 리눅스 커널 코드의 타이머를 사용하려면 `timer_list` 타입의 변수를 정의해야 합니다. `timer_list` 구조는 두 가지 방법으로 초기화될 수 있습니다. 첫 번째 방법은 [include/linux/timer.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/timer.h) 헤더에 정의된 `init_timer` 매크로를 사용하는 것입니다.

```C
#define init_timer(timer)    \
	__init_timer((timer), 0)

#define __init_timer(_timer, _flags)   \
         init_timer_key((_timer), (_flags), NULL, NULL)
```

여기서 `init_timer_key` 함수는 그저 다음 함수를 호출합니다.

```C
do_init_timer(timer, flags, name, key);
```

여기서 이 필드에는 기본값으로 지정된 `timer`가 입력됩니다. 두 번째 방법은:

```C
#define TIMER_INITIALIZER(_function, _expires, _data)		\
	__TIMER_INITIALIZER((_function), (_expires), (_data), 0)
```

매크로를 사용하는 것이며 이 매크로는 지정된 `timer_list` 구조체도 초기화하는 매크로입니다.

`dynamic timer`가 초기화되면 다음 함수의 호출을 통해 이 `timer`를 시작할 수 있습니다:

```C
void add_timer(struct timer_list * timer);
```

그리고 멈추기 위해선 아래 함수를 사용합니다:

```C
int del_timer(struct timer_list * timer);
```

끝입니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널의 타이머 및 타이머 관리 관련 내용을 설명하는 챕터의 네 번째 파트는 끝입니다. 이전 파트에서는 두 가지 새로운 개념인 `tick broadcast` 프레임워크와 `NO_HZ` 모드를 알았습니다. 이번 파트에서는 계속해서 시간 관리와 관련된 내용을 살펴본 후 새로운 개념인 `dynamic timer` 또는 소프트웨어 타이머에 대해 알게 되었습니다. 본 파트에서 `dynamic timers` 관리 코드의 구현을 자세히 보지는 못했지만, 이 개념을 중심으로 데이터 구조와 API를 보았습니다.

다음 파트에서는 리눅스 커널의 타이머 관리 관련 사항에 대해 계속해서 자세히 알아보고 `timers`라는 새로운 개념을 살펴보겠습니다.

만약 질문이나 의견이 있으시다면, 트위터에서 [0xAX](https://twitter.com/0xAX) 저를 핑해주시거나, [이메일](anotherworldofworld@gmail.com)을 보내주시거나, 또는 그냥 [이슈](https://github.com/0xAX/linux-insides/issues/new)를 생성해주세요.

**영어는 제 모국어가 아닙니다, 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수들을 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한국 번역](https://github.com/junsooo/linux-insides-ko)로 PR을 보내주세요.**

참고 링크
-------------------------------------------------------------------------------

* [IP](https://en.wikipedia.org/wiki/Internet_Protocol)
* [netfilter](https://en.wikipedia.org/wiki/Netfilter)
* [network](https://en.wikipedia.org/wiki/Computer_network)
* [cpumask](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [interrupt](https://en.wikipedia.org/wiki/Interrupt)
* [jiffies](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html)
* [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [spinlock](https://en.wikipedia.org/wiki/Spinlock)
* [procfs](https://en.wikipedia.org/wiki/Procfs)
* [previous part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-3.html)
