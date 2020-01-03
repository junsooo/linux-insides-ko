커널 초기화. Part 9.
================================================================================

RCU 초기화
================================================================================

이것은 [리눅스 커널 초기화 프로세스](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)의 아홉번째 파트입니다. 또한 이전파트에서 우리는 [스케줄러 초기화](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-8.html)에서 멈췄습니다. 이 부분에서 우리는 리눅스 커널 초기화 과정을 계속해서 진행할 것이며이 부분의 주요 목적은 [RCU](http://en.wikipedia.org/wiki/Read-copy-update)의 초기화에 대해 배우는 것입니다. `sched_init` 이후의 [init / main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 의 다음 단계는 `preempt_enable`, ` preempt_disable` 두 가지 매크로가 있습니다.

* `preempt_disable`
* `preempt_enable`

선점 비활성화 및 활성화. 우선 운영 체제 커널의 맥락에서 '선점'이 무엇인지 이해하려고 노력하십시오. 간단히 말해서, 선점은 운영 체제 커널이 현재 작업을 선점하여 우선 순위가 높은 작업을 실행하는 능력입니다. 여기서 우리는 초기 부팅 시간 동안 단 하나의`init` 프로세스 만 가질 것이고 prection을 비활성화 할 필요가 있으며,`cpu_idle` 함수를 호출하기 전에 중지 할 필요가 없습니다. `preempt_disable` 매크로는 [include / linux / preempt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/preempt.h)에 정의되어 있으며 `CONFIG_PREEMPT_COUNT` 커널 구성 옵션에 따라 다릅니다. 이 매크로는 다음과 같이 구현됩니다.
```C
#define preempt_disable() \
do { \
        preempt_count_inc(); \
        barrier(); \
} while (0)
```

`CONFIG_PREEMPT_COUNT`가 설정되지 않은 경우 :

```C
#define preempt_disable()                       barrier()
```

살펴 봅시다. 우선 우리는 이러한 매크로 구현들 사이의 한 가지 차이점을 볼 수 있습니다. `CONFIG_PREEMPT_COUNT`가 설정된`preempt_disable`에는`preempt_count_inc`의 호출이 포함됩니다. 보류 된 잠금 수와`preempt_disable` 호출을 저장하는 특수한 percpu 변수가 있습니다.

```C
DECLARE_PER_CPU(int, __preempt_count);
```

`preempt_disable`의 첫 번째 구현에서이`__preempt_count`를 증가시킵니다. `__preempt_count`의 값을 반환하는 API가 있으며,`preempt_count` 함수입니다. 우리가`preempt_disable`을 불렀을 때, 우선 우리는 preempt_count_inc 매크로를 사용하여 선점 카운터를 증가시킵니다 :

```
#define preempt_count_inc() preempt_count_add(1)
#define preempt_count_add(val)  __preempt_count_add(val)
```

여기서 preempt_count_add는 `raw_cpu_add_4` 매크로를 호출하여 우리의 경우 주어진 `percpu` 변수 (`__preempt_count`)에 `1`을 추가합니다. (`precpu` 변수에 대한 자세한 정보는 [CPU 단위 변수](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)에서 읽을 수 있습니다) . 자, 우리는 `__preempt_count`를 증가 시켰고 다음 단계에서 두 매크로에서 `barrier` 매크로의 호출을 볼 수 있습니다. `barrier` 매크로는 최적화 장벽을 삽입합니다. `x86_64` 아키텍처를 가진 프로세서에서 독립적 인 메모리 액세스 작업은 어떤 순서로도 수행 될 수 있습니다. 그렇기 때문에 순서대로 컴파일러와 프로세서를 가리킬 수있는 기회가 필요합니다. 이 메커니즘은 메모리 장벽입니다. 간단한 예를 살펴봅시다.

```C
preempt_disable();
foo();
preempt_enable();
```

Compiler can rearrange it as:

```C
preempt_disable();
preempt_enable();
foo();
```

이 경우, 선점 불가능 함수 `foo`를 선점 할 수 있습니다. 우리가 `preempt_disable`과 `preempt_enable` 매크로에 `barrier` 매크로를 넣으면 컴파일러가 다른 명령문과 `preempt_count_inc`를 바꾸지 못하게됩니다. 장벽에 대한 자세한 내용은 [여기](http://en.wikipedia.org/wiki/Memory_barrier) 및 [여기](https://www.kernel.org/doc/Documentation/memory-barriers.txt)를 참조하십시오.

다음 단계에서 다음 진술을 볼 수 있습니다.

```C
if (WARN(!irqs_disabled(),
	 "Interrupts were enabled *very* early, fixing it\n"))
	local_irq_disable();
```

[IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 상태를 확인하고 활성화 된 경우 비활성화 ( 'x86_64'에 대한`cli '명령 사용)합니다.

그게 전부입니다. 선점은 비활성화되어 있으며 계속 진행할 수 있습니다.

정수 ID 관리 초기화
--------------------------------------------------------------------------------

다음 단계에서 [lib/idr.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/idr.c)에 정의 된 `idr_init_cache` 함수의 호출을 볼 수 있습니다. `idr` 라이브러리는 리눅스 커널의 다양한 [장소](http://lxr.free-electrons.com/ident?i=idr_find)에서 정수 `ID` 를 객체에 할당하고 ID에 의해 객체를 찾는 것을 관리하기 위해 사용됩니다.

`idr_init_cache` 함수의 구현을 살펴봅시다:

```C
void __init idr_init_cache(void)
{
        idr_layer_cache = kmem_cache_create("idr_layer_cache",
                                sizeof(struct idr_layer), 0, SLAB_PANIC, NULL);
}
```

여기서 우리는 `kmem_cache_create`의 호출을 볼 수 있습니다. 우리는 이미 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c#L485) 에서 `kmem_cache_init`를 호출했습니다. 이 함수는 `kmem_cache_alloc`을 사용하여 일반화 된 캐시를 다시 생성합니다. ([Linux 커널 메모리 관리](https://0xax.gitbooks.io/linux-insides/content/MM/index.html)에서 볼 수있는 캐시에 대한 자세한 내용) ). 우리의 경우, 우리는 [slab](http://en.wikipedia.org/wiki/Slab_allocation) 할당 자에 의해 사용될 `kmem_cache_t`를 사용하고 `kmem_cache_create`가 그것을 만듭니다. 보다시피 우리는`kmem_cache_create`에 5 개의 매개 변수를 전달합니다 :

* 캐시 이름;
* 캐시에 저장할 객체의 크기;
* 페이지에서 첫 번째 객체의 오프셋;
* 플래그;
* 객체의 생성자.

정수 ID에 대해 `kmem_cache`를 만듭니다. 정수 `ID`는 정수 ID 세트를 포인터 세트에 맵핑하기 위해 일반적으로 사용되는 패턴입니다. [i2c](http://en.wikipedia.org/wiki/I%C2%B2C) 드라이버 서브 시스템에서 정수 ID의 사용법을 볼 수 있습니다. 예를 들어, [drivers/i2c/i2c-core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/i2c/i2c-core.c)는 `i2c` 서브 시스템의 핵심을 나타냅니다 `DEFINE_IDR` 매크로를 사용하여 `i2c` 어댑터의 `ID`를 정의합니다.

```C
static DEFINE_IDR(i2c_adapter_idr);
```

and then uses it for the declaration of the `i2c` adapter:

```C
static int __i2c_add_numbered_adapter(struct i2c_adapter *adap)
{
  int     id;
  ...
  ...
  ...
  id = idr_alloc(&i2c_adapter_idr, adap, adap->nr, adap->nr + 1, GFP_KERNEL);
  ...
  ...
  ...
}
```
id2_adapter_idr은 동적으로 계산 된 버스 번호를 나타냅니다.

정수 ID 관리에 대한 자세한 내용은 [here] (https://lwn.net/Articles/103209/)를 참조하십시오.

RCU 초기화
--------------------------------------------------------------------------------

다음 단계는 `rcu_init` 함수를 사용한 [RCU](http://en.wikipedia.org/wiki/Read-copy-update) 초기화이며 두 가지 커널 구성 옵션에 따라 구현됩니다.

* `CONFIG_TINY_RCU`
* `CONFIG_TREE_RCU`


첫 번째 경우 `rcu_init`는 [kernel / rcu / tiny.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/rcu/tiny.c)에 있고 두 번째 경우에 있습니다. [kernel / rcu / tree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/rcu/tree.c)에 정의됩니다. 우리는 `tree rcu`의 구현을 보게 될 것입니다. 그러나 우선 `RCU`에 관한 것입니다.

`RCU` 또는 읽기-복사 업데이트는 Linux 커널에서 구현되는 확장 가능한 고성능 동기화 메커니즘입니다. 초기 단계에서 리눅스 커널은 동시에 실행되는 애플리케이션에 대한 지원과 환경을 제공했지만 모든 실행은 단일 글로벌 잠금을 사용하여 커널에서 직렬화되었습니다. 오늘날 리눅스 커널에는 단일 전역 잠금이 없지만 [자유없는 데이터 구조](http://en.wikipedia.org/wiki/Concurrent_data_structure), [percpu](https ://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)를 포함한 다른 메커니즘을 제공합니다.  데이터 구조 및 기타. 이러한 메커니즘 중 하나는 '읽기-복사 업데이트'입니다. `RCU` 기법은 거의 수정되지 않은 데이터 구조를 위해 설계되었습니다. `RCU`의 아이디어는 간단하다. 예를 들어 거의 수정되지 않은 데이터 구조가 있습니다. 누군가이 데이터 구조를 변경하려면이 데이터 구조를 복사하고 사본을 모두 변경하십시오. 동시에 데이터 구조의 다른 모든 사용자는 이전 버전의 데이터 구조를 사용합니다. 다음으로, 데이터 구조의 원본 버전에 사용자가없는 안전한 순간을 선택하고 수정 된 사본으로 이를 업데이트해야합니다.

물론 RCU에 대한이 설명은 매우 간단합니다. `RCU`에 대한 세부 사항을 이해하려면 우선 용어를 배워야합니다. `RCU`의 데이터 리더는 [critical section](http://en.wikipedia.org/wiki/Critical_section)에서 실행되었습니다. 데이터 리더가 임계 구역에 도달 할 때마다 임계 구역에서 나올 때`rcu_read_lock` 및`rcu_read_unlock`을 호출합니다. 스레드가 임계 섹션에 없으면 `정지 상태`라고 하는 상태가 됩니다. 모든 스레드가 `유예 기간` 이라고 하는 `대기 상태`에 있는 순간. 스레드가 데이터 구조에서 요소를 제거하려는 경우 두 단계로 발생합니다. 첫 번째 단계는 `제거` - 데이터 구조에서 요소를 원자 적으로 제거하지만 물리적 메모리는 해제하지 않습니다. 이 스레드 작성기가 발표되고 완료 될 때까지 기다립니다. 이 순간부터 스레드 판독기에서 제거 된 요소를 사용할 수 있습니다. `유예 기간`이 끝나면 요소 제거의 두 번째 단계가 시작되고 실제 메모리에서 요소가 제거됩니다.

`RCU`의 몇 가지 구현이 있습니다. 오래된 `RCU`는 클래식이라하고, 새로운 구현은 `tree` RCU입니다. 이미 알고 있듯이 `CONFIG_TREE_RCU` 커널 설정 옵션은 트리 `RCU`를 활성화합니다. 또 다른 하나는 `CONFIG_TINY_RCU`와 `CONFIG_SMP = n`에 의존하는 `tiny` RCU입니다. 동기화 프리미티브에 대한 별도의 장에서 일반적으로 `RCU`에 대한 자세한 내용을 볼 수 있지만 이제 [kernel/rcu/tree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/rcu/tree.c) 에서`rcu_init` 구현을 살펴 봅시다.

```C
void __init rcu_init(void)
{
         int cpu;

         rcu_bootup_announce();
         rcu_init_geometry();
         rcu_init_one(&rcu_bh_state, &rcu_bh_data);
         rcu_init_one(&rcu_sched_state, &rcu_sched_data);
         __rcu_init_preempt();
         open_softirq(RCU_SOFTIRQ, rcu_process_callbacks);

         /*
          * We don't need protection against CPU-hotplug here because
          * this is called early in boot, before either interrupts
          * or the scheduler are operational.
          */
         cpu_notifier(rcu_cpu_notify, 0);
         pm_notifier(rcu_pm_notify, 0);
         for_each_online_cpu(cpu)
                 rcu_cpu_notify(NULL, CPU_UP_PREPARE, (void *)(long)cpu);

         rcu_early_boot_tests();
}
```

`rcu_init` 함수의 시작에서 우리는`cpu` 변수를 정의하고`rcu_bootup_announce`를 호출합니다. `rcu_bootup_announce` 함수는 매우 간단합니다 :

```C
static void __init rcu_bootup_announce(void)
{
        pr_info("Hierarchical RCU implementation.\n");
        rcu_bootup_announce_oddness();
}
```

It just prints information about the `RCU` with the `pr_info` function and `rcu_bootup_announce_oddness` which uses `pr_info` too, for printing different information about the current `RCU` configuration which depends on different kernel configuration options like `CONFIG_RCU_TRACE`, `CONFIG_PROVE_RCU`, `CONFIG_RCU_FANOUT_EXACT`, etc. In the next step, we can see the call of the `rcu_init_geometry` function. This function is defined in the same source code file and computes the node tree geometry depends on the amount of CPUs. Actually `RCU` provides scalability with extremely low internal RCU lock contention. What if a data structure will be read from the different CPUs? `RCU` API provides the `rcu_state` structure which presents RCU global state including node hierarchy. Hierarchy is presented by the:

```
struct rcu_node node[NUM_RCU_NODES];
```

array of structures. As we can read in the comment of above definition:

```
The root (first level) of the hierarchy is in ->node[0] (referenced by ->level[0]), the second
level in ->node[1] through ->node[m] (->node[1] referenced by ->level[1]), and the third level
in ->node[m+1] and following (->node[m+1] referenced by ->level[2]).  The number of levels is
determined by the number of CPUs and by CONFIG_RCU_FANOUT.

Small systems will have a "hierarchy" consisting of a single rcu_node.
```

The `rcu_node` structure is defined in the [kernel/rcu/tree.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/rcu/tree.h) and contains information about current grace period, is grace period completed or not, CPUs or groups that need to switch in order for current grace period to proceed, etc. Every `rcu_node` contains a lock for a couple of CPUs. These `rcu_node` structures are embedded into a linear array in the `rcu_state` structure and represented as a tree with the root as the first element and covers all CPUs. As you can see the number of the rcu nodes determined by the `NUM_RCU_NODES` which depends on number of available CPUs:

```C
#define NUM_RCU_NODES (RCU_SUM - NR_CPUS)
#define RCU_SUM (NUM_RCU_LVL_0 + NUM_RCU_LVL_1 + NUM_RCU_LVL_2 + NUM_RCU_LVL_3 + NUM_RCU_LVL_4)
```

where levels values depend on the `CONFIG_RCU_FANOUT_LEAF` configuration option. For example for the simplest case, one `rcu_node` will cover two CPU on machine with the eight CPUs:

```
+-----------------------------------------------------------------+
|  rcu_state                                                      |
|                 +----------------------+                        |
|                 |         root         |                        |
|                 |       rcu_node       |                        |
|                 +----------------------+                        |
|                    |                |                           |
|               +----v-----+       +--v-------+                   |
|               |          |       |          |                   |
|               | rcu_node |       | rcu_node |                   |
|               |          |       |          |                   |
|         +------------------+     +----------------+             |
|         |                  |        |             |             |
|         |                  |        |             |             |
|    +----v-----+    +-------v--+   +-v--------+  +-v--------+    |
|    |          |    |          |   |          |  |          |    |
|    | rcu_node |    | rcu_node |   | rcu_node |  | rcu_node |    |
|    |          |    |          |   |          |  |          |    |
|    +----------+    +----------+   +----------+  +----------+    |
|         |                 |             |               |       |
|         |                 |             |               |       |
|         |                 |             |               |       |
|         |                 |             |               |       |
+---------|-----------------|-------------|---------------|-------+
          |                 |             |               |
+---------v-----------------v-------------v---------------v--------+
|                 |                |               |               |
|     CPU1        |      CPU3      |      CPU5     |     CPU7      |
|                 |                |               |               |
|     CPU2        |      CPU4      |      CPU6     |     CPU8      |
|                 |                |               |               |
+------------------------------------------------------------------+
```

So, in the `rcu_init_geometry` function we just need to calculate the total number of `rcu_node` structures. We start to do it with the calculation of the `jiffies` till to the first and next `fqs` which is `force-quiescent-state` (read above about it):

```C
d = RCU_JIFFIES_TILL_FORCE_QS + nr_cpu_ids / RCU_JIFFIES_FQS_DIV;
if (jiffies_till_first_fqs == ULONG_MAX)
        jiffies_till_first_fqs = d;
if (jiffies_till_next_fqs == ULONG_MAX)
        jiffies_till_next_fqs = d;
```

where:

```C
#define RCU_JIFFIES_TILL_FORCE_QS (1 + (HZ > 250) + (HZ > 500))
#define RCU_JIFFIES_FQS_DIV     256
```

As we calculated these [jiffies](http://en.wikipedia.org/wiki/Jiffy_%28time%29), we check that previous defined `jiffies_till_first_fqs` and `jiffies_till_next_fqs` variables are equal to the [ULONG_MAX](http://www.rowleydownload.co.uk/avr/documentation/index.htm?http://www.rowleydownload.co.uk/avr/documentation/ULONG_MAX.htm) (their default values) and set they equal to the calculated value. As we did not touch these variables before, they are equal to the `ULONG_MAX`:

```C
static ulong jiffies_till_first_fqs = ULONG_MAX;
static ulong jiffies_till_next_fqs = ULONG_MAX;
```

In the next step of the `rcu_init_geometry`, we check that `rcu_fanout_leaf` didn't change (it has the same value as `CONFIG_RCU_FANOUT_LEAF` in compile-time) and equal to the value of the `CONFIG_RCU_FANOUT_LEAF` configuration option, we just return:

```C
if (rcu_fanout_leaf == CONFIG_RCU_FANOUT_LEAF &&
    nr_cpu_ids == NR_CPUS)
    return;
```

After this we need to compute the number of nodes that an `rcu_node` tree can handle with the given number of levels:

```C
rcu_capacity[0] = 1;
rcu_capacity[1] = rcu_fanout_leaf;
for (i = 2; i <= MAX_RCU_LVLS; i++)
    rcu_capacity[i] = rcu_capacity[i - 1] * CONFIG_RCU_FANOUT;
```

And in the last step we calculate the number of rcu_nodes at each level of the tree in the [loop](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/rcu/tree.c#L4094).

As we calculated geometry of the `rcu_node` tree, we need to go back to the `rcu_init` function and next step we need to initialize two `rcu_state` structures with the `rcu_init_one` function:

```C
rcu_init_one(&rcu_bh_state, &rcu_bh_data);
rcu_init_one(&rcu_sched_state, &rcu_sched_data);
```

The `rcu_init_one` function takes two arguments:

* Global `RCU` state;
* Per-CPU data for `RCU`.

Both variables defined in the [kernel/rcu/tree.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/rcu/tree.h) with its `percpu` data:

```
extern struct rcu_state rcu_bh_state;
DECLARE_PER_CPU(struct rcu_data, rcu_bh_data);
```

About this states you can read [here](http://lwn.net/Articles/264090/). As I wrote above we need to initialize `rcu_state` structures and `rcu_init_one` function will help us with it. After the `rcu_state` initialization, we can see the call of the ` __rcu_init_preempt` which depends on the `CONFIG_PREEMPT_RCU` kernel configuration option. It does the same as previous functions - initialization of the `rcu_preempt_state` structure with the `rcu_init_one` function which has `rcu_state` type. After this, in the `rcu_init`, we can see the call of the:

```C
open_softirq(RCU_SOFTIRQ, rcu_process_callbacks);
```

function. This function registers a handler of the `pending interrupt`. Pending interrupt or `softirq` supposes that part of actions can be delayed for later execution when the system is less loaded. Pending interrupts is represented by the following structure:

```C
struct softirq_action
{
        void    (*action)(struct softirq_action *);
};
```

which is defined in the [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h) and contains only one field - handler of an interrupt. You can check about `softirqs` in the your system with the:

```
$ cat /proc/softirqs
                    CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7
          HI:          2          0          0          1          0          2          0          0
       TIMER:     137779     108110     139573     107647     107408     114972      99653      98665
      NET_TX:       1127          0          4          0          1          1          0          0
      NET_RX:        334        221     132939       3076        451        361        292        303
       BLOCK:       5253       5596          8        779       2016      37442         28       2855
BLOCK_IOPOLL:          0          0          0          0          0          0          0          0
     TASKLET:         66          0       2916        113          0         24      26708          0
       SCHED:     102350      75950      91705      75356      75323      82627      69279      69914
     HRTIMER:        510        302        368        260        219        255        248        246
         RCU:      81290      68062      82979      69015      68390      69385      63304      63473
```

The `open_softirq` function takes two parameters:

* index of the interrupt;
* interrupt handler.

and adds interrupt handler to the array of the pending interrupts:

```C
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
        softirq_vec[nr].action = action;
}
```

In our case the interrupt handler is - `rcu_process_callbacks` which is defined in the [kernel/rcu/tree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/rcu/tree.c) and does the `RCU` core processing for the current CPU. After we registered `softirq` interrupt for the `RCU`, we can see the following code:

```C
cpu_notifier(rcu_cpu_notify, 0);
pm_notifier(rcu_pm_notify, 0);
for_each_online_cpu(cpu)
    rcu_cpu_notify(NULL, CPU_UP_PREPARE, (void *)(long)cpu);
```

Here we can see registration of the `cpu` notifier which needs in systems which supports [CPU hotplug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt) and we will not dive into details about this theme. The last function in the `rcu_init` is the `rcu_early_boot_tests`:

```C
void rcu_early_boot_tests(void)
{
        pr_info("Running RCU self tests\n");

        if (rcu_self_test)
                 early_boot_test_call_rcu();
         if (rcu_self_test_bh)
                 early_boot_test_call_rcu_bh();
         if (rcu_self_test_sched)
                early_boot_test_call_rcu_sched();
}
```

which runs self tests for the `RCU`.

That's all. We saw initialization process of the `RCU` subsystem. As I wrote above, more about the `RCU` will be in the separate chapter about synchronization primitives.

Rest of the initialization process
--------------------------------------------------------------------------------

Ok, we already passed the main theme of this part which is `RCU` initialization, but it is not the end of the linux kernel initialization process. In the last paragraph of this theme we will see a couple of functions which work in the initialization time, but we will not dive into deep details around this function for different reasons. Some reasons not to dive into details are following:

* They are not very important for the generic kernel initialization process and depend on the different kernel configuration;
* They have the character of debugging and not important for now;
* We will see many of this stuff in the separate parts/chapters.

After we initialized `RCU`, the next step which you can see in the [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) is the - `trace_init` function. As you can understand from its name, this function initialize [tracing](http://en.wikipedia.org/wiki/Tracing_%28software%29) subsystem. You can read more about linux kernel trace system - [here](http://elinux.org/Kernel_Trace_Systems).

After the `trace_init`, we can see the call of the `radix_tree_init`. If you are familiar with the different data structures, you can understand from the name of this function that it initializes kernel implementation of the [Radix tree](http://en.wikipedia.org/wiki/Radix_tree). This function is defined in the [lib/radix-tree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/radix-tree.c) and you can read more about it in the part about [Radix tree](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-2.html).

In the next step we can see the functions which are related to the `interrupts handling` subsystem, they are:

* `early_irq_init`
* `init_IRQ`
* `softirq_init`

We will see explanation about this functions and their implementation in the special part about interrupts and exceptions handling. After this many different functions (like `init_timers`, `hrtimers_init`, `time_init`, etc.) which are related to different timing and timers stuff. We will see more about these function in the chapter about timers.

The next couple of functions are related with the [perf](https://perf.wiki.kernel.org/index.php/Main_Page) events - `perf_event-init` (there will be separate chapter about perf), initialization of the `profiling` with the `profile_init`. After this we enable `irq` with the call of the:

```C
local_irq_enable();
```

which expands to the `sti` instruction and making post initialization of the [SLAB](http://en.wikipedia.org/wiki/Slab_allocation) with the call of the `kmem_cache_init_late` function (As I wrote above we will know about the `SLAB` in the [Linux memory management](https://0xax.gitbooks.io/linux-insides/content/MM/index.html) chapter).

After the post initialization of the `SLAB`, next point is initialization of the console with the `console_init` function from the [drivers/tty/tty_io.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/tty/tty_io.c).

After the console initialization, we can see the `lockdep_info` function which prints information about the [Lock dependency validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt). After this, we can see the initialization of the dynamic allocation of the `debug objects` with the `debug_objects_mem_init`, kernel memory leak [detector](https://www.kernel.org/doc/Documentation/kmemleak.txt) initialization with the `kmemleak_init`, `percpu` pageset setup with the `setup_per_cpu_pageset`, setup of the [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access) policy with the `numa_policy_init`, setting time for the scheduler with the `sched_clock_init`, `pidmap` initialization with the call of the `pidmap_init` function for the initial `PID` namespace, cache creation with the `anon_vma_init` for the private virtual memory areas and early initialization of the [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) with the `acpi_early_init`.

This is the end of the ninth part of the [linux kernel initialization process](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) and here we saw initialization of the [RCU](http://en.wikipedia.org/wiki/Read-copy-update). In the last paragraph of this part (`Rest of the initialization process`) we will go through many functions but did not dive into details about their implementations. Do not worry if you do not know anything about these stuff or you know and do not understand anything about this. As I already wrote many times, we will see details of implementations in other parts or other chapters.

Conclusion
--------------------------------------------------------------------------------

It is the end of the ninth part about the linux kernel [initialization process](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html). In this part, we looked on the initialization process of the `RCU` subsystem. In the next part we will continue to dive into linux kernel initialization process and I hope that we will finish with the `start_kernel` function and will go to the `rest_init` function from the same [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) source code file and will see the start of the first process.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [lock-free data structures](http://en.wikipedia.org/wiki/Concurrent_data_structure)
* [kmemleak](https://www.kernel.org/doc/Documentation/kmemleak.txt)
* [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [IRQs](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [RCU](http://en.wikipedia.org/wiki/Read-copy-update)
* [RCU documentation](https://github.com/torvalds/linux/tree/master/Documentation/RCU)
* [integer ID management](https://lwn.net/Articles/103209/)
* [Documentation/memory-barriers.txt](https://www.kernel.org/doc/Documentation/memory-barriers.txt)
* [Runtime locking correctness validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [Per-CPU variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [Linux kernel memory management](https://0xax.gitbooks.io/linux-insides/content/MM/index.html)
* [slab](http://en.wikipedia.org/wiki/Slab_allocation)
* [i2c](http://en.wikipedia.org/wiki/I%C2%B2C)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-8.html)
