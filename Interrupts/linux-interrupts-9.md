인터럽트와 인터럽트 처리. Part 9.
================================================================================

연기된 인터럽트(deferred interrupts) 개요 (Softirq, Tasklets 그리고 Workqueues)
--------------------------------------------------------------------------------

이번 파트는 리눅스 커널에서 인터럽트 및 인터럽트 처리 [챕터](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)의 9번째 파트입니다. [지난 파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-8.html)에서는 [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irqinit.c) 소스 코드 파일에 정의된 `init_IRQ`의 구현을 보았습니다. 그래서 이번 파트에서는 외부 하드웨어 인터럽트와 관련된 초기화 관련 내용을 계속해서 살펴볼 것입니다.

인터럽트는 여러가지 중요한 특성들을 가지고 있으며 그 중 두 가지가 다음과 같습니다.

* 인터럽트 핸들러는 빠르게 실행되어야 함.
* 때때로 인터럽트 핸들러는 많은 양의 작업을 수행해야 함.

아시다시피, 두 특성을 모두 만족하도록 만드는 것은 거의 불가능합니다. 그 때문에, 이전에는 인터럽트 처리가 두 부분으로 나뉘었습니다.

* 상반부(Top half);
* 하반부(Bottom half);

과거에는 리눅스 커널에서 인터럽트 처리를 연기(defer)하기 위한 방법이 하나 있었습니다. 그리고 그것은 프로세서의 '하반부'(`the bottom half`)라고 불렸지만 이젠 본래의 뜻과는 다릅니다. 이제 이 용어는 인터럽트의 연기 처리를 구성하는 모든 다른 방법을 가리키는 일반 명사로 쓰입니다. 인터럽트의 연기 처리는 시스템의 부하가 적을 때 인터럽트에 대한 처리 중 일부가 나중으로 연기 될 수 있음을 의미합니다. 쉽게 유추하실 수 있겠지만, 인터럽트 핸들러는 인터럽트가 비활성화된 상황에서 실행될 때에 허용량 이상인 많은 양의 작업을 수행할 수 있습니다. 그래서 인터럽트 처리를 두 부분으로 나눕니다 (나눌 수 있습니다). 첫번째 부분에서 인터럽트의 메인 핸들러는 최소한의 작업만 수행합니다. 그런 다음 두번째 부분을 예약하고 작업을 마칩니다. 시스템 사용량이 적고 프로세서 컨텍스트가 인터럽트를 처리 할 수 있게 되면 두 번째 파트가 작업을 시작하고 인터럽트의 연기된 나머지 부분을 마무리합니다.

리눅스 커널의 연기된 인터럽트에는 세 가지 유형이 있습니다:

* `softirqs`;
* `tasklets`;
* `workqueues`;

그리고 우리는 이번 파트에서 이 세 유형 모두에 대한 설명을 볼 것입니다. 제가 말씀드렸 듯이, 우리가 여태 이 주제에 대해 본 것은 조금에 불과하므로, 이제 이 주제에 대한 세부 사항들을 자세히 살펴볼 시간입니다.

Softirqs
----------------------------------------------------------------------------------

리눅스 커널에서 병렬화가 등장함에 따라, 하반부 핸들러의 새로운 구현 방식은 `ksoftirqd` (아래에서 자세히 다룰 것임)라는 프로세서 특정 (processor specific) 커널 스레드의 성능을 기반으로합니다. 각 프로세서에는 `ksoftirqd/n`이라고 하는 스레드가 있으며, 여기서 `n`은 프로세서의 번호입니다. `systemd-cgls` util 출력에서 이를 확인해 볼 수 있습니다:

```
$ systemd-cgls -k | grep ksoft
├─   3 [ksoftirqd/0]
├─  13 [ksoftirqd/1]
├─  18 [ksoftirqd/2]
├─  23 [ksoftirqd/3]
├─  28 [ksoftirqd/4]
├─  33 [ksoftirqd/5]
├─  38 [ksoftirqd/6]
├─  43 [ksoftirqd/7]
```

`Spawn_ksoftirqd` 함수는 이 스레드를 시작합니다. 보시다시피 이 함수는 early_[initcall](https://kernelnewbies.org/Documents/InitcallMechanism) 이라고 불립니다:

```C
early_initcall(spawn_ksoftirqd);
```

Softirqs는 리눅스 커널의 컴파일 시 정적으로 결정되며 `open_softirq` 함수는 `softirq` 초기화 작업을 처리합니다. `open_softirq` 함수는 [kernel/softirq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c)에 정의되어 있습니다:


```C
void open_softirq(int nr, void (*action)(struct softirq_action *))
{
	softirq_vec[nr].action = action;
}
```

그리고 이 함수가 두 가지 매개 변수를 사용하는 것을 볼 수 있습니다:

* `softirq_vec` 배열의 인덱스;
* softirq 함수를 실행하기 위한 포인터;

먼저 `softirq_vec` 배열을 살펴보겠습니다.:

```C
static struct softirq_action softirq_vec[NR_SOFTIRQS] __cacheline_aligned_in_smp;
```

이는 동일한 소스 코드 파일에 정의되어 있습니다. 보시다시피, `softirq_vec` 배열에는 `softirq_action` 타입의 `NR_SOFTIRQS` 또는 `10` 타입의 `softirq`가 들어있을 수 있습니다. 우선 그것의 요소들에 대해서 다뤄봅시다. 현재 버전의 Linux 커널에는 10개의 softirq 벡터가 정의되어 있습니다. tasklet 처리를 위해 2개, 네트워킹을 위해 2개, 블록 계층(block layer)을 위해 2개, 타이머를 위해 2개, 스케줄러 및 읽기-복사-업데이트 처리를 위한 각각 1개입니다. 이러한 모든 유형은 다음과 같은 열거형으로 표시됩니다.

```C
enum
{
        HI_SOFTIRQ=0,
        TIMER_SOFTIRQ,
        NET_TX_SOFTIRQ,
        NET_RX_SOFTIRQ,
        BLOCK_SOFTIRQ,
        BLOCK_IOPOLL_SOFTIRQ,
        TASKLET_SOFTIRQ,
        SCHED_SOFTIRQ,
        HRTIMER_SOFTIRQ,
        RCU_SOFTIRQ,
        NR_SOFTIRQS
};
```

이러한 종류의 softirqs의 모든 이름은 다음 배열로 표시됩니다.

```C
const char * const softirq_to_name[NR_SOFTIRQS] = {
        "HI", "TIMER", "NET_TX", "NET_RX", "BLOCK", "BLOCK_IOPOLL",
        "TASKLET", "SCHED", "HRTIMER", "RCU"
};
```

또는 `/proc/softirqs` 출력에서 확인할 수 있습니다.

```
~$ cat /proc/softirqs 
                    CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7       
          HI:          5          0          0          0          0          0          0          0
       TIMER:     332519     310498     289555     272913     282535     279467     282895     270979
      NET_TX:       2320          0          0          2          1          1          0          0
      NET_RX:     270221        225        338        281        311        262        430        265
       BLOCK:     134282         32         40         10         12          7          8          8
BLOCK_IOPOLL:          0          0          0          0          0          0          0          0
     TASKLET:     196835          2          3          0          0          0          0          0
       SCHED:     161852     146745     129539     126064     127998     128014     120243     117391
     HRTIMER:          0          0          0          0          0          0          0          0
         RCU:     337707     289397     251874     239796     254377     254898     267497     256624
```

보시다시피 `softirq_vec` 배열에는 `softirq_action` 타입이 있습니다. 이는 `softirq` 메커니즘과 관련된 메인 데이터 구조이므로, 모든 `softirq`는 `softirq_action`구조체로 표현됩니다. `softirq_action` 구조체는 단일 필드로만 구성되어 있습니다. softirq 함수에 대한 action 포인터입니다.

```C
struct softirq_action
{
         void    (*action)(struct softirq_action *);
};
```

따라서, 우리는 `open_softirq` 함수가 `softirq_vec` 어레이를 주어진 `softirq_action`으로 채우는 것으로 이해할 수 있습니다. ( `open_softirq`함수 호출과 함께 ) 실행 대기중인 연기된 인터럽트는`raise_softirq` 함수 호출에 의해 활성화되어야합니다. 이 함수는 하나의 매개 변수 --  softirq 인덱스 `nr` -- 만 사용합니다. 그 구현에 대해 한 번 살펴 봅시다.

```C
void raise_softirq(unsigned int nr)
{
        unsigned long flags;

        local_irq_save(flags);
        raise_softirq_irqoff(nr);
        local_irq_restore(flags);
}
```

여기서 우리는 `local_irq_save`와 `local_irq_restore` 매크로 사이에서 `raise_softirq_irqoff` 함수의 호출을 볼 수 있습니다. [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqflags.h) 헤더 파일에 정의 된`local_irq_save`는 [eflags](https://en.wikipedia.org/wiki/FLAGS_register) 레지스터의 [IF](https://en.wikipedia.org/wiki/Interrupt_flag) 플래그의 상태를 저장하고 로컬 프로세서에서 인터럽트를 비활성화합니다. `local_irq_restore` 매크로는 동일한 헤더 파일에 정의되어 있으며 정반대의 일을 하죠: `interrupt flag`를 복원하고 인터럽트를 활성화합니다. 여기서 인터럽트를 비활성화하는 이유는 `softirq` 인터럽트가 인터럽트 컨텍스트에서 실행되고 그래서 단 하나의 softirq가 실행되어야 하기 때문입니다.

`raise_softirq_irqoff` 함수는 로컬 프로세서의 `softirq` 비트 마스크 (`__softirq_pending`)에서 주어진 인덱스 `nr`에 해당하는 비트를 설정하여 softirq를 다른 것들과 다르다고 표시합니다. 함수는 이러한:

```C
__raise_softirq_irqoff(nr);
```

매크로의 도움으로 그것을 수행합니다. 그런 다음, 함수는 `irq_count` 값을 리턴하는 `in_interrupt`의 결과를 점검합니다. 이 장의 첫 번째 [part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-1.html)에서 `irq_count`를 이미 보았으며, 이것은 CPU가 이미 인터럽트 스택에 있는지 여부를 확인하기 위해 사용됩니다. 우리는 방금 `raise_softirq_irqoff`를 빠져나왔으며, 우리가 인터럽트 컨텍스트에 있으면 `IF` 플래그를 복원하고 로컬 프로세서에서 인터럽트를 활성화하고 인터럽트 컨텍스트에 있지 않으면 `wakeup_softirqd`를 호출합니다.

```C
if (!in_interrupt())
	wakeup_softirqd();
```

여기서 `wakeup_softirqd` 함수는 로컬 프로세서의 `ksoftirqd` 커널 스레드를 활성화합니다:

```C
static void wakeup_softirqd(void)
{
	struct task_struct *tsk = __this_cpu_read(ksoftirqd);

    if (tsk && tsk->state != TASK_RUNNING)
        wake_up_process(tsk);
}
```

각 `ksoftirqd` 커널 스레드는 연기된 인터럽트의 존재를 점검하고 점검 결과에 따라 `__do_softirq` 함수를 호출하는 `run_ksoftirqd` 함수를 실행합니다. 이 함수는 로컬 프로세서의`__softirq_pending` softirq 비트 마스크를 읽고 모든 비트 집합에 해당하는 연기 가능한 함수를 실행합니다. 연기된 함수를 실행하는 동안 새로운 보류중인 'softirqs'가 발생할 수 있습니다. 여기서 가장 큰 문제는  `__do_softirq` 함수가 연기된 인터럽트를 처리하는 동안 사용자 공간(userspace) 코드 실행이 오랫동안 지연될수 있다는 것입니다. 이러한 목적에서, 이것은 처리를 완료해야 하는 시간 제한을 가지고 있습니다:

```C
unsigned long end = jiffies + MAX_SOFTIRQ_TIME;
...
...
...
restart:
while ((softirq_bit = ffs(pending))) {
	...
	h->action(h);
	...
}
...
...
...
pending = local_softirq_pending();
if (pending) {
	if (time_before(jiffies, end) && !need_resched() &&
		--max_restart)
            goto restart;
}
...			
```

연기된 인터럽트의 존재 여부를 확인하는 것은 주기적으로 이루어집니다. 이러한 점검이 발생하는 몇 가지 지점이 있습니다. 그 중 핵심은 [arch/x86/kernel/irq.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98eb93d973/arch/x86/irq.c)에 정의된 `do_IRQ` 함수의 호출이며, 이는 리눅스에서 실제 인터럽트 처리를 위한 주요 수단을 제공합니다. `do_IRQ`가 인터럽트 처리를 완료하면, [arch/x86/include/asm/apic.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/apic.h)에서 `exiting_irq` 함수를 호출하며, `exiting_irq`는 `irq_exit` 함수의 호출로 확장됩니다. `irq_exit`는 연기된 인터럽트와 현재 컨텍스트를 확인하고 `invoke_softirq` 함수를 호출합니다.

```C
if (!in_interrupt() && local_softirq_pending())
    invoke_softirq();
```

또한 `__do_softirq`를 실행합니다. 요약하자면, 각 `softirq`는 다음 단계를 거칩니다.
 * `open_softirq` 함수로 `softirq`를 등록.
 * `raise_softirq` 함수로 이것이 연기된 것으로 표시하여 `softirq`를 활성화.
 * 그 후, 다음에 리눅스 커널이 연기 가능한 함수의 실행을 예약 할 때 표시된 모든 `softirqs`가 트리거됩니다. 그리고 같은 타입을 가진 연기된 함수들의 실행.

저 위에 써놓은 것처럼, `softirqs`는 정적으로 할당되어 있으며 이는 로드 가능한 커널 모듈의 문제입니다. `softirq`를 기반으로 만들어진 두 번째 개념 --`tasklets`가 이 문제를 해결합니다.

Tasklets
--------------------------------------------------------------------------------

`softirq`와 관련된 Linux 커널의 소스 코드를 읽어보시면 이것이 매우 드물게 사용된다는 것을 알 수 있습니다. 연기 가능한 함수를 구현하는 바람직한 방법은 `tasklets`입니다. 위에서 이미 언급했듯이`tasklet`은 `softirq` 개념 위에, 그리고 일반적으로는 두 개의 `softirqs` 위에 구축됩니다.

* `TASKLET_SOFTIRQ`;
* `HI_SOFTIRQ`.

간단히 말해, `tasklet`은 런타임에 할당 및 초기화 될 수있는 `softirqs`이며, `softirqs`와 달리 동일한 유형의 tasklet은 한 번에 여러 프로세서에서 실행될 수 없습니다. 자, 이제 우리는 `softirqs`에 대해 조금 알고 있습니다. 물론 위의 내용들은 이것에 관한 모든 측면을 다루지는 않지만, 이제 코드를 직접 살펴보고 실습에서 단계별로 `softirqs`에 대해 더 자세히 알고 `tasklets`에 대해 알 수 있습니다. 이번 파트의 시작 부분에 언급했던 `softirq_init` 함수의 구현으로 돌아가 보겠습니다. 이 함수는 [kernel/softirq.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98d973/kernel/softirq.c) 소스 코드 파일에 정의되어 있습니다. 구현에 대해 살펴보겠습니다.

```C
void __init softirq_init(void)
{
        int cpu;

        for_each_possible_cpu(cpu) {
                per_cpu(tasklet_vec, cpu).tail =
                        &per_cpu(tasklet_vec, cpu).head;
                per_cpu(tasklet_hi_vec, cpu).tail =
                        &per_cpu(tasklet_hi_vec, cpu).head;
        }

        open_softirq(TASKLET_SOFTIRQ, tasklet_action);
        open_softirq(HI_SOFTIRQ, tasklet_hi_action);
}
```

`softirq_init` 함수의 시작 부분에서 정수(integer) `cpu` 변수의 정의를 확인할 수 있습니다. 다음으로 이것을 시스템에서 가능한 모든 프로세서를 하나씩 처리하는 `for_each_possible_cpu` 매크로의 매개 변수로 사용할 것입니다. `possible processor`(가능한 프로세서)가 낯설다면 [CPU masks](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html) 챕터에서 읽어보시길 바랍니다. 간단히 말해서, `possible cpus`는 시스템 부트된 기간 동안 언제든지 연결(plug)할 수 있는 프로세서의 집합입니다. 모든 `possible processors`는 `cpu_possible_bits` 비트맵에 저장되어있으며, [kernel/cpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cpu.c)에서 그 정의를 찾아볼 수 있습니다:

```C
static DECLARE_BITMAP(cpu_possible_bits, CONFIG_NR_CPUS) __read_mostly;
...
...
...
const struct cpumask *const cpu_possible_mask = to_cpumask(cpu_possible_bits);
```

자, 정수 `cpu` 변수를 정의했고 가능한 모든 프로세서를 `for_each_possible_cpu` 매크로를 통해 살펴보고 다음 두 개의 [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) 변수를 초기화합니다.

* `tasklet_vec`;
* `tasklet_hi_vec`;

이 두 `per-cpu` 변수는 `softirq_init` 함수와 동일한 소스 [코드](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/softirq.c) 파일에 정의되어 있으며 두 개의 `tasklet_head` 구조체입니다.

```C
static DEFINE_PER_CPU(struct tasklet_head, tasklet_vec);
static DEFINE_PER_CPU(struct tasklet_head, tasklet_hi_vec);
```

여기서 `tasklet_head` 구조체는 `Tasklets` 목록을 나타내며 두 개의 필드(head 및 tail)을 가지고 있습니다.

```C
struct tasklet_head {
        struct tasklet_struct *head;
        struct tasklet_struct **tail;
};
```

`tasklet_structure`는 [include/linux/interrupt.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/interrupt.h)에 정의되어 있으며 `Tasklet`을 나타냅니다. 이 단어는 이전에 우리가 이 책에서 보지 못한 단어입니다. `tasklet`이 뭔지 이해해봅시다. 사실, 태스크릿은 지연된 인터럽트를 처리하는 메커니즘 중 하나입니다. `tasklet_structure` 구조체의 구현에 대해 알아보겠습니다.

```C
struct tasklet_struct
{
        struct tasklet_struct *next;
        unsigned long state;
        atomic_t count;
        void (*func)(unsigned long);
        unsigned long data;
};
```

이 구조체에는 5개의 필드가 포함되어 있습니다.

* 스케줄링 대기열의 다음 tasklet;
* tasklet의 상태;
* tasklet의 현재 상태를 나타냄, 활성화되었는지 아닌지;
* tasklet의 메인 콜백;
* 콜백의 매개변수.

이 경우, `softirq_init` 함수의 `tasklet_vec`와 `tasklet_hi_vec`의 두 개의 taskelt 배열만 초기화하도록 설정했습니다. 우선 순위가 높은 tasklet과 tasklet은 각각 `tasklet_vec` 및 `tasklet_hi_vec` 배열에 저장됩니다. 우리는 이러한 어레이를 초기화했으며 이제 '`softirq_init` 함수의 끝에서 [kernel/softirq.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d0b9893d973/softq.c) 소스 파일에 정의된 `open_softirq` 함수의 호출 두 번을 볼 수 있습니다.

```C
open_softirq(TASKLET_SOFTIRQ, tasklet_action);
open_softirq(HI_SOFTIRQ, tasklet_hi_action);
```

`open_softirq` 함수의 주된 목적은 `softirq`의 초기화입니다. `open_softirq` 함수의 구현에 대해 알아보겠습니다.

, in our case they are: `tasklet_action` and the `tasklet_hi_action` or the `softirq` function associated with the `HI_SOFTIRQ` softirq is named `tasklet_hi_action` and `softirq` function associated with the `TASKLET_SOFTIRQ` is named `tasklet_action`. Linux 커널은 `tasklet`을 조작하기 위한 API를 제공합니다. 첫번째는 `tasklet_struct`를 받는 `tasklet_init`함수이며, 이 함수는 주어진 `tasklet_struct`를 주어진 데이터로 초기화합니다:

```C
void tasklet_init(struct tasklet_struct *t,
                  void (*func)(unsigned long), unsigned long data)
{
    t->next = NULL;
    t->state = 0;
    atomic_set(&t->count, 0);
    t->func = func;
    t->data = data;
}
```

아래 두 매크로를 사용하여 tasklet을 정적으로 초기화할 수 있는 추가적인 방법이 있습니다.

```C
DECLARE_TASKLET(name, func, data);
DECLARE_TASKLET_DISABLED(name, func, data);	
```

리눅스 커널은 태스크릿을 실행 준비됨으로 표시하기 위한 다음 세 가지 함수를 제공합니다.

```C
void tasklet_schedule(struct tasklet_struct *t);
void tasklet_hi_schedule(struct tasklet_struct *t);
void tasklet_hi_schedule_first(struct tasklet_struct *t);
```

첫 번째 함수는 일반 우선 순위로 tasklet을 예약하고, 두 번째 함수는 높은 우선순위로, 세 번째 함수는 순서없이 예약합니다. 이 세 가지 함수의 구현이 모두 비슷하므로 첫 번째 함수인 `tasklet_schedule`만 고려하겠습니다. 그 구현을 살펴봅시다.

```C
static inline void tasklet_schedule(struct tasklet_struct *t)
{
    if (!test_and_set_bit(TASKLET_STATE_SCHED, &t->state))
        __tasklet_schedule(t);
}

void __tasklet_schedule(struct tasklet_struct *t)
{
        unsigned long flags;

        local_irq_save(flags);
        t->next = NULL;
        *__this_cpu_read(tasklet_vec.tail) = t;
        __this_cpu_write(tasklet_vec.tail, &(t->next));
        raise_softirq_irqoff(TASKLET_SOFTIRQ);
        local_irq_restore(flags);
}
```

보시다시피 이 함수는 지정된 tasklet의 상태를 확인하고 `TASKLET_STATE_SCHED`로 설정하고 지정된 tasklet으로 `__tasklet_chedule`을 실행합니다. `__tasklet_schedule`은 위에서 본 `raise_softirq` 함수와 매우 유사합니다. 시작 시 `interrupt flag`를 저장하고 인터럽트를 비활성화합니다. 이후, 새 tasklet으로 `tasklet_vec`를 업데이트하고 위에서 본 `raise_softirq_irqoff`함수를 호출합니다. Linux 커널 스케줄러가 연기된 함수를 실행하기로 결정하면, `TASKLET_SOFTIRQ`와 관련된 연기된 함수에 대해서 `taskle_action`, 그리고  `HI_SOFTIRQ`와 관련된 연기된 함수에 대해  `tasklet_hi_action`함수가 호출됩니다. 이 함수들은 매우 유사하며 단 하나의 차이점이 있습니다--  `tasklet_action`은 `tasklet_vec`를, `tasklet_hi_action`은 `tasklet_hi_vec`를 사용합니다.

`tasklet_action` 함수의 구현에 대해 알아봅시다.

```C
static void tasklet_action(struct softirq_action *a)
{
    local_irq_disable();
    list = __this_cpu_read(tasklet_vec.head);
    __this_cpu_write(tasklet_vec.head, NULL);
    __this_cpu_write(tasklet_vec.tail, this_cpu_ptr(&tasklet_vec.head));
    local_irq_enable();

    while (list) {
		if (tasklet_trylock(t)) {
	        t->func(t->data);
            tasklet_unlock(t);
	    }
		...
		...
		...
    }
}
```

`tasklet_action` 함수의 시작 부분에서 `local_irq_disable` 매크로의 도움을 받아 로컬 프로세서에 대한 인터럽트를 사용하지 않도록 설정합니다 (이 장의 두 번째 [part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-2.html)에서 이 매크로에 대해 읽을 수 있습니다). 다음으로, 모든 tasklet을 일반적인 방법으로 실행해야 하므로 일반 우선순위가 있는 태스크릿을 포함하는 리스트의 맨 앞(주소)을 취해, 이 per-cpu 리스트를 "NULL"로 설정합니다. 이후 로컬 프로세서에 대해 인터럽트를 사용하도록 설정하고 루프에서 tasklet 목록을 처리합니다. 루프를 반복할 때마다 주어진 tasklet에 대해 `tasklet_trylock` 함수를 호출하여 `TASKLET_STATE_RUN`에서 지정된 tasklet의 상태를 업데이트합니다.

```C
static inline int tasklet_trylock(struct tasklet_struct *t)
{
    return !test_and_set_bit(TASKLET_STATE_RUN, &(t)->state);
}
```

이 작업이 성공적이라면, 우리는 tasklet의 액션(`tasklet_init`에서 설정됨)을 실행하고 tasklet의 `TASKLET_STATE_RUN` 상태를 clear하는 `tasklet_unlock` 함수를 호출합니다.

일반적으로 여기까지가 `tasklet`의 개념에 관한 모든 것입니다. 물론 이것은 완전한 `tasklet`를 다루지는 않지만, 저는 이정도면 이 개념에 대해 계속 배울 수 있는 좋은 지점이 되었다고 생각합니다.

'태스크렛'은 리눅스 커널에서 [광범위하게](http://lxr.free-electrons.com/ident?i=tasklet_init) 사용되는 개념입니다만, 이 파트의 시작부분에서 설명한 바와 같이, 연기된 함수에 대한 세 번째 메커니즘인 '워크큐'가 아직 남아있습니다. 다음 단락에서 살펴봅시다.

Workqueues (작업 대기열)
--------------------------------------------------------------------------------

Workqueues는 연기된 함수를 처리하기 위한 또 다른 개념입니다. 그것은 약간의 차이는 있어도 `tasklet`과 유사합니다. Workqueue 함수는 커널 프로세스 컨텍스트에서 실행되지만 `tasklet` 함수는 소프트웨어 인터럽트 컨텍스트에서 실행됩니다. 즉, `workqueue` 함수는 `tasklet` 함수처럼 atomic(미세해)서는 안 됩니다. Tasklet은 항상 원래 제출된 프로세서에서 실행됩니다. Workqueue는 기본적으로는 동일한 방식으로 작동합니다. `workqueue`의 개념은 다음 구조체로 나타내어집니다.

```C
struct worker_pool {
    spinlock_t              lock;
    int                     cpu;
    int                     node;
    int                     id;
    unsigned int            flags;

    struct list_head        worklist;
    int                     nr_workers;
...
...
...
```

이 구조체는 Linux 커널의 [kernel/workqueue.c](https://gitub.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98eb93d973/kernel/workqueueue.c) 소스 코드 파일에 정의된 구조체입니다. 이 구조체의 소스 코드는 필드가 상당히 많기 때문에 여기에 쓰지 않겠습니다. 하지만 우리는 이 필드들 중 일부는 살펴볼 것입니다.

가장 기본적인 양식에서 work queue 하위 시스템(subsystem)은 다른 곳에서 대기 중인 작업을 처리할 커널 스레드를 생성하기 위한 인터페이스입니다. 이러한 커널 스레드는 모두 일꾼 스레드(`worker threads`)라고 합니다. 작업 대기열은 [include/linux/workqueue.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d18e0bd98d973/include/linux/workqueueue.h)에 정의된 `work_struct`에 의해 유지됩니다. 이 구조체를 살펴봅시다.

```C
struct work_struct {
    atomic_long_t data;
    struct list_head entry;
    work_func_t func;
#ifdef CONFIG_LOCKDEP
    struct lockdep_map lockdep_map;
#endif
};
```

여기서 눈여겨 보아야 할 것은 두개입니다:`func` -- `workqueue`에 의해서 예약될 함수 그리고 `data`- 이 함수의 매개 변수. Linux 커널은 `kworker`라고 하는 특수한 cpu별 스레드를 제공합니다.

```
systemd-cgls -k | grep kworker
├─    5 [kworker/0:0H]
├─   15 [kworker/1:0H]
├─   20 [kworker/2:0H]
├─   25 [kworker/3:0H]
├─   30 [kworker/4:0H]
...
...
...
```

이 프로세스는 (마치 `softirqs`의 `ksoftirqd`같이) workqueue의 연기된 함수를 스케줄링하는 데 사용할 수 있습니다. 이 외에도, 우리는 `workqueue`을 위한 새로운 분리된 일꾼 스레드를 만들 수 있습니다. Linux 커널은 정적으로 workqueue를 생성하기 위해 다음과 같은 매크로를 제공합니다.

```C
#define DECLARE_WORK(n, f) \
    struct work_struct n = __WORK_INITIALIZER(n, f)
```

이 매크로는 두 가지 매개 변수를 요구합니다: workqueue의 이름과 workqueue 함수. 런타임에 workqueue를 생성하려면 다음 매크로를 사용합니다:

```C
#define INIT_WORK(_work, _func)       \
    __INIT_WORK((_work), (_func), 0)

#define __INIT_WORK(_work, _func, _onstack)                     \
    do {                                                        \
            __init_work((_work), _onstack);                     \
            (_work)->data = (atomic_long_t) WORK_DATA_INIT();   \
            INIT_LIST_HEAD(&(_work)->entry);                    \
             (_work)->func = (_func);                           \
    } while (0)
```

이 매크로는 생성할 `work_struct` 구조체와 이 workqueue에서 스케줄링해야 하는 함수를 요구합니다. 이 매크로들 중 하나로 `work`가 만들어진 후에는 이를 `workqueue`에 넣어야 합니다. 이는 `queue_work` 또는 `que_delayed_work` 함수의 도움을 받아 수행할 수 있습니다.

```C
static inline bool queue_work(struct workqueue_struct *wq,
                              struct work_struct *work)
{
    return queue_work_on(WORK_CPU_UNBOUND, wq, work);
}
```

`queue_work` 함수는 그저 특정 프로세서의 대기열에 작업을 넣는 `que_work_on` 함수를 호출하기만 합니다. 우리의 경우에는 `WORK_CPU_UNBOUND`를 `que_work_on` 함수로 전달하는 것을 알아두세요. 이것은 [include/linux/workqueue.h](https://github.com/torvalds/linux/blob/16f73d7e1765ccab3d20e0d98d973/include/linux/workqueue.h)에 정의된 `enum`의 일부이며, 어느 특정 프로세서에도 묶여있지 않는 workqueue를 나타냅니다. `queue_work_on` 함수는 테스트를 수행하고 주어진 `work`의 `WORK_STRUCT_PENDING_BIT`를 설정하고, 주어진 프로세서와 `work`에 대한 `workqueue`로`__queue_work` 함수를 실행합니다.

```C
bool queue_work_on(int cpu, struct workqueue_struct *wq,
           struct work_struct *work)
{
    bool ret = false;
    ...
    if (!test_and_set_bit(WORK_STRUCT_PENDING_BIT, work_data_bits(work))) {
        __queue_work(cpu, wq, work);
        ret = true;
    }
    ...
    return ret;
}
```

`__queue_work` 함수는 `work pool`을 가져옵니다. 네, `workpool`은 `workqueue`가 아닙니다. 실제로, 모든 `work`는 `workqueue`에 배치되는 것이 아니라 Linux 커널의 `worker_pool` 구조로 표현되는 `work pool`에도 배치됩니다. 위에서 볼 수 있듯이 `workque_struct`는 `worker_pools` 리스트인 `pwqs` 필드를 가지고 있습니다. `workqueue`를 만들 때 각 프로세서마다 `pool_workque`가 눈에 띕니다. `pool_workqueue`는 `worker_pool`과 관련되어 있으며, `worker_pool`은 동일한 프로세서에 할당되며 우선 순위 대기열 유형에 해당합니다. 이들을 통해 `workqueue`는 `worker_pool`과 상호 작용합니다. 따라서 `__queue_work` 함수에서는 `raw_smp_processor_id`(Linux 커널 초기화 프로세스 챕터의 네 번째 [part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)에서 이 매크로에 대한 정보를 찾을 수 있습니다)를 사용하여 cpu를 현재 프로세서로 설정하고 주어진 `workqueue_struct`에 대한 `pool_workqueue`를 얻고, 주어진 `work`를 주어진 `workqueue`에 삽입합니다.

```C
static void __queue_work(int cpu, struct workqueue_struct *wq,
                         struct work_struct *work)
{
...
...
...
if (req_cpu == WORK_CPU_UNBOUND)
    cpu = raw_smp_processor_id();

if (!(wq->flags & WQ_UNBOUND))
    pwq = per_cpu_ptr(wq->cpu_pwqs, cpu);
else
    pwq = unbound_pwq_by_node(wq, cpu_to_node(cpu));
...
...
...
insert_work(pwq, work, worklist, work_flags);
```

`work`과 `workqueue`를 만들 수 있는 만큼, 이것들이 언제 실행되는지 알아야 합니다. 제가 이미 썼듯이, 모든 `work`는 커널 스레드에 의해 실행됩니다. 이 커널 스레드가 예약되면 주어진 `workqueue`에서 `work`를 실행하기 시작합니다. 각 일꾼 스레드는 `worker_thread` 함수 내의 루프를 실행합니다. 이 스레드는 많은 차이를 만들지만 이것들의 일부는 우리가 이 부분에서 이전에 보았던 것과 비슷합니다. 이것은 실행을 시작하면서 `work_struct` 또는 `works`를 `workqueue`에서 모두 제거합니다.

이것으로 끝입니다

결론
--------------------------------------------------------------------------------

이것으로 [Interrupts and Interrupt Handling](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html) 챕터의 9번째 파트는 끝이며, 이번 파트에서는 외부 하드웨어 인터럽트를 계속 살펴봤습니다. 이전 파트에서는 `IRQ`와 메인 `irq_desc` 구조체의 초기화를 보았습니다. 이번 파트에서는 연기되는 함수들에 사용되는 `softirq`, `tasklet` 및 `workqueue`의 세 가지 개념을 보았습니다.

다음 파트에서는 '인터럽트와 인터럽트 처리' 챕터의 마지막 부분으로, 실제 하드웨어 드라이버를 살펴보고 인터럽트 서브시스템에서 어떻게 작동하는지 알아보겠습니다.

만약 질문이나 의견이 있으시다면, [트위터](https://twitter.com/0xAX)에서 저를 핑해주시거나, 코멘트를 달아주세요.

**영어는 제 모국어가 아닙니다, 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수들을 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한국 번역](https://github.com/junsooo/linux-insides-ko)로 PR을 보내주세요.**

참고 링크
--------------------------------------------------------------------------------

* [initcall](https://kernelnewbies.org/Documents/InitcallMechanism)
* [IF](https://en.wikipedia.org/wiki/Interrupt_flag)
* [eflags](https://en.wikipedia.org/wiki/FLAGS_register)
* [CPU masks](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [Workqueue](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/workqueue.txt)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-8.html)
