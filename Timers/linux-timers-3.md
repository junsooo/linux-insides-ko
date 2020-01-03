Linux 커널의 타이머와 시간 관리. part 3.
================================================================================

tick broadcast framework 와 dyntick
--------------------------------------------------------------------------------


이것은 리눅스 커널에서 타이머와 시간 관리 관련 내용을 설명하는 [챕터](https://junsoolee.gitbook.io/linux-insides-ko/summary/timers)의 세 번째 파트이며 이전 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/timers/linux-timers-3)에서 `clocksource` 프레임 워크에서 멈췄습니다. 이 프레임 워크는 Linux 커널에서 제공하는 특수 카운터와 밀접하게 관련되어 있기 때문에 다루기 시작했습니다. 이 챕터의 첫 파트에서 보았던 카운터 중 하나는 `jiffies` 입니다. 첫 파트에서 썼듯이, 우리는 리눅스 커널 초기화 과정에서 시간 관리와 관련된 것을 단계별로 고려할 것입니다. 이전 단계는 다음을 호출하는 것이었습니다 :

```C
register_refined_jiffies(CLOCK_TICK_RATE);
```

이 함수는 [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 소스 코드 파일에 정의되어 있고 우리를 위해`refined_jiffies` 클럭 소스의 초기화를 실행합니다. 이 함수는 [https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c](arch/x86/kernel/setup.c) 소스 코드에 정의 된 `setup_arch` 함수에서 호출되며 아키텍처 별 (여기서는 [x86_64](https://en.wikipedia.org/wiki/X86-64)) 초기화를 수행합니다. `setup_arch`의 구현을 살펴보면 `register_refined_jiffies` 의 호출이 `setup_arch` 함수가 작업을 완료하기 전의 마지막 단계임을 알 수 있습니다.

`setup_arch` 실행이 끝난 후에 이미 설정된 `x86_64` 특성의 많은 것들이 있습니다. 예를 들어 이미 인터럽트를 처리 할 수 ​​있는 일부 초기 [interrupt](https://en.wikipedia.org/wiki/Interrupt) 핸들러, [initrd](https://en.wikipedia.org/wiki/Initrd)를 위해 예약 된 메모리 공간, 스캔된 [DMI](https://en.wikipedia.org/wiki/Desktop_Management_Interface), Linux 커널 로그 버퍼가 이미 설정되어 있습니다. 이는 [printk](https://en.wikipedia.org/wiki/Printk) 함수가 작동 할 수 있음을, [e820](https://en.wikipedia.org/wiki/E820)이 파싱되고 리눅스 커널은 이미 사용 가능한 메모리와 다른 많은 아키텍처 특성의 것들에 대해 알고있음을 의미합니다.(관심이 있다면,이 책의 두 번째 챕터에서 `setup_arch` 함수와 리눅스 커널 초기화 프로세스에 대해 더 많이 읽을 수 있습니다).

이제 `setup_arch`는 작업을 마치고 일반적인 리눅스 커널 코드로 돌아갈 수 있습니다. `setup_arch` 함수는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일에 정의 된 `start_kernel` 함수에서 호출되었습니다. 따라서 이 함수로 돌아갑니다. 여러분은 `start_kernel` 함수 안에서 `setup_arch` 함수 다음에 다른 많은 함수가 호출되는 것을 볼 수 있습니다. 그러나 우리의 챕터는 타이머와 시간 관리에 관련있는 것들에 대한 내용이기 때문에 우리는 이 주제와 관련이 없는 모든 코드는 건너 뜁니다. Linux 커널에서 시간 관리와 관련된 첫 번째 함수는 다음과 같습니다. :

```C
tick_init();
```

이 함수는 `start_kernel`에 있습니다. [kernel/time/tick-common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tick-common.c) 소스 코드 파일에 정의 된 `tick_init` 함수는 다음 두 가지를 수행합니다. :

* Initialization of `tick broadcast` framework related data structures;
* Initialization of `full` tickless mode related data structures.

우리는 이 책에서  `tick broadcast` 프레임 워크와 관련된 것을 보지 못했고, Linux 커널의 thickless 모드에 대해서는 전혀 모르는 상태입니다. 따라서 이 파트의 요점은 이러한 개념을 살펴보고 이들이 무엇인지 아는 것입니다.

Idle process
--------------------------------------------------------------------------------

우선, `tick_init` 함수의 구현을 살펴 봅시다. 이미 작성한 것처럼이 함수는 [kernel/time/tick-common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tick-common.c) 소스 코드 파일에 정의되어 있으며, 다음 두 함수의 호출로 구성됩니다. :

```C
void __init tick_init(void)
{
	tick_broadcast_init();
	tick_nohz_init();
}
```

소제목에서 알 수 있듯이 지금은 `tick_broadcast_init` 함수에만 관심을 갖습니다. 이 함수는 [kernel/time/tick-broadcast.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tick-broadcast.c) 소스 코드 파일에 정의되어 있으며, 데이터 구조체와 관련된 `tick broadcast` 프레임 워크의 초기화를 실행합니다. `tick_broadcast_init` 함수의 구현을 살펴보고 이 함수가 하는 일을 이해하기 전에 `tick broadcast` 프레임 워크에 대해 알아야합니다.

중앙 프로세서의 중요한 포인트는 프로그램을 실행하는 것입니다. 그러나 가끔 프로세서가 프로그램에서 사용되지 않을 때에는 특수한 상태에 있을 수 있습니다. 이 특수한 상태를 [idle](https://en.wikipedia.org/wiki/Idle_%28CPU%29)라고합니다. 프로세서에 실행할 것이 없으면 Linux 커널은 `idle` 작업을 시작합니다. 우리는 이미 [Linux 커널 초기화 과정](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-10.html)의 마지막 파트에서 이것에 대해 조금 살펴보았습니다. 리눅스 커널이 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일에서 `start_kernel` 함수의 모든 초기화 프로세스를 마치면 동일한 소스 코드 파일에서 `rest_init` 함수를 호출합니다. 이 함수의 중요한 포인트는 커널 `init` 스레드와 `kthreadd` 스레드를 시작한다는 것,`schedule` 함수를 호출하여 작업 스케줄링을 시작한다는 것, [kernel/sched/idle.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sched/idle.c) 소스 코드 파일에 정의 된 `cpu_idle_loop`를 호출함으로써 sleep 상태로 간다는 것입니다.

`cpu_idle_loop` 함수는 무한 루프를 나타내며 각 루프에서 스케줄을 다시 짤 필요가 있는지 확인합니다. 스케줄러가 실행할 항목을 찾은 후 `idle` 프로세스가 작업을 완료하면 `schedule_preempt_disabled` 함수를 호출하여 제어가 새로 실행 가능한 작업으로 이동합니다. :

```C
static void cpu_idle_loop(void)
{
	while (1) {
		while (!need_resched()) {
		...
		...
		...
	    */ the main idle function /*
		cpuidle_idle_call();
	}
	...
	...
	...
	schedule_preempt_disabled();
}
```

물론 지금 파트에서 `cpu_idle_loop` 함수의 전체 구현과 `idle` 상태의 세부 사항을 살펴보지는 않을 것입니다. 이는 주제와 관련이 없기 때문입니다. 하지만 한 흥미로운 부분이 있습니다. 우리는 프로세서가 한 번에 하나의 작업만 수행할 수 있다는 것을 알고 있습니다. 프로세서가 `cpu_idle_loop`에서 무한 루프를 실행하는 경우, Linux 커널은 어떻게 일정을 변경하고 `idle` 프로세스를 중지하기로 결정할까요? 답은 시스템 타이머 인터럽트입니다. 인터럽트가 발생하면 프로세서는 `idle` 스레드를 중지하고 제어를 인터럽트 처리기로 전송합니다. 시스템 타이머 인터럽트 핸들러가 처리되면 `need_resched`는 true를 반환하고, 리눅스 커널은`idle` 프로세스를 중단하고 현재 실행 가능한 작업으로 제어권을 넘깁니다. 그러나 시스템 타이머 인터럽트를 처리하는 것은 [전원 관리](https://en.wikipedia.org/wiki/Power_management)에 효과적이지 않습니다. 프로세서가 `idle` 상태인 경우 시스템 타이머 인터럽트를 보내는 데에 작은 문제가 있기 때문입니다.

기본적으로 리눅스 커널에서 사용 가능한 `CONFIG_HZ_PERIODIC` 커널 설정 옵션이 있으며, 이는 시스템 타이머의 각 인터럽트를 처리하도록 지시합니다. 이 문제를 해결하기 위해 Linux 커널은 스케줄링 클럭 인터럽트를 관리하는 두 가지 추가 방법을 제공합니다.

첫 번째는 idle 프로세서에서 스케줄링 클럭 틱을 생략하는 것입니다. Linux 커널에서 이 동작을 활성화하려면 `CONFIG_NO_HZ_IDLE` 커널 설정 옵션을 활성화해야합니다. 이 옵션을 사용하면 Linux 커널이 idle 프로세서로 타이머 인터럽트를 보내지 않아도됩니다. 이 경우 주기적인 타이머 인터럽트에서 온디맨드 인터럽트로 대체됩니다. 이 모드를 `dyntick-idle`모드라고 합니다. 그러나 커널이 시스템 타이머의 인터럽트를 처리하지 않으면, 시스템이 수행 할 작업이 없는지 어떻게 결정할 수 있을까요?

idle 작업이 실행되도록 선택 될 때마다 [kernel/time/tick-sched.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tich-sched.c)] 소스 코드 파일에 정의 된 `tick_nohz_idle_enter` 함수의 호출로 주기적 틱이 비활성화/활성화됩니다. Linux 커널에는 다음 인터럽트를 예약하는 데 사용되는 `클럭 이벤트 장치`라는 특별한 개념이 있습니다. 이 개념은 미래의 특정 시간에 인터럽트를 전달할 수 았으며 Linux 커널의 `clock_event_device` 구조체로 표시되는 장치에 대한 API를 제공합니다. 지금 `clock_event_device` 구조체의 구현으로 뛰어 들지는 않고, 이 챕터의 다음 파트에서 볼 것입니다. 하지만 지금 우리에게 하나의 흥미로운 부분이 있습니다.

두 번째 방법은 프로세서가 `idle`상태이거나, 실행 가능한 작업이 하나만 있거나, 다른 말로 프로세서가 바쁜 프로세서일때, 스케줄링 클럭 틱을 생략하는 것입니다. `CONFIG_NO_HZ_FULL` 커널 설정 옵션으로 이 기능을 활성화 할 수 있으며 타이머 인터럽트 수를 크게 줄일 수 있습니다.

idle 프로세서는 `cpu_idle_loop` 외에도 절전 상태 일 수 있습니다. 리눅스 커널은 특별한 `cpuidle` 프레임 워크를 제공합니다. 이 프레임 워크의 중요한 포인트는 idle 프로세서를 대기 상태로 만드는 것입니다. 이 상태들 세트의 이름은 `C-states`입니다. 로컬 타이머가 비활성화되어 있다면 프로세서가 어떻게 깨어날까요? 리눅스 커널은 이를 위해 `tick broadcast` 프레임 워크를 제공합니다. 이 프레임 워크의 핵심은 `C-states`의 영향을 받지 않는 타이머를 할당하는 것입니다. 이 타이머는 슬립상태의 프로세서를 깨울 것입니다.

이제 몇 가지 이론이 끝나면 함수 구현으로 돌아갈 수 있습니다. `tick_init` 함수는 다음 두 함수를 호출한다는 것을 떠올려봅시다. :


```C
void __init tick_init(void)
{
	tick_broadcast_init();
	tick_nohz_init();
}
```

첫 번째 함수를 알아 봅시다. [kernel/time/tick-broadcast.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tick-broadcast.c) 소스 코드 파일에 정의 된 첫 번째 `tick_broadcast_init` 함수는 `tick broadcast` '프레임 워크 관련 데이터 구조체의 초기화를 실행합니다. `tick_broadcast_init` 함수의 구현을 살펴 봅시다. :

```C
void __init tick_broadcast_init(void)
{
        zalloc_cpumask_var(&tick_broadcast_mask, GFP_NOWAIT);
        zalloc_cpumask_var(&tick_broadcast_on, GFP_NOWAIT);
        zalloc_cpumask_var(&tmpmask, GFP_NOWAIT);
#ifdef CONFIG_TICK_ONESHOT
         zalloc_cpumask_var(&tick_broadcast_oneshot_mask, GFP_NOWAIT);
         zalloc_cpumask_var(&tick_broadcast_pending_mask, GFP_NOWAIT);
         zalloc_cpumask_var(&tick_broadcast_force_mask, GFP_NOWAIT);
#endif
}
```

보다시피 `tick_broadcast_init` 함수는 `zalloc_cpumask_var` 함수의 도움으로 서로 다른 [cpumasks](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-2)를 할당합니다.[lib/cpumask.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/cpumask.c) 소스 코드 파일에 정의 된 `zalloc_cpumask_var` 함수는 다음 함수의 호출로 확장됩니다. :

```C
bool zalloc_cpumask_var(cpumask_var_t *mask, gfp_t flags)
{
        return alloc_cpumask_var(mask, flags | __GFP_ZERO);
}
```

궁극적으로 메모리 공간은 `kmalloc_node` 함수의 도움으로 특정 플래그와 함께 주어진 `cpumask`에 할당됩니다 :

```C
*mask = kmalloc_node(cpumask_size(), flags, node);
```

이제 `tick_broadcast_init` 함수에서 초기화 될 `cpumasks`를 살펴 봅시다. 보다시피 `tick_broadcast_init` 함수는 6 개의 `cpumasks`를 초기화 할 것이며, 마지막 3개의 `cpumasks`의 초기화는 `CONFIG_TICK_ONESHOT` 커널 설정 옵션에 의존합니다.

처음 세 개의 `cpumasks`는 다음과 같습니다. :

* `tick_broadcast_mask` - sleep 모드에 있는 프로세서 목록을 나타내는 비트 맵;
* `tick_broadcast_on` - 주기적인 브로드 캐스트 상태에 있는 프로세서 수를 저장하는 비트 맵;
* `tmpmask` - 임시 사용을 위한 비트 맵.

우리가 이미 알고 있듯이, 다음 3개의 `cpumasks`는 `CONFIG_TICK_ONESHOT` 커널 설정 옵션에 의존합니다. 실제로 각 clock 이벤트 장치는 다음 두 가지 모드 중 하나 일 것입니다. :

* `periodic` - 주기적인 이벤트를 발생시키는 clock 이벤트 장치;
* `oneshot`  - 한 번만 발생하는 이벤트를 발생시키는 clock 이벤트 장치.

리눅스 커널은 [include/linux/clockchips.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clockchips.h) 헤더 파일에서 이러한 클럭 이벤트 장치에 대한 두 개의 마스크를 정의합니다. :

```C
#define CLOCK_EVT_FEAT_PERIODIC        0x000001
#define CLOCK_EVT_FEAT_ONESHOT         0x000002
```

그리고 마지막 세 `cpumasks` 는:

* `tick_broadcast_oneshot_mask` - 알려져야하는 프로세서 수를 저장합니다.
* `tick_broadcast_pending_mask `- 방송 대기중인 프로세서의 수를 저장합니다.
* `tick_broadcast_force_mask` - 강제 브로드 캐스트 된 프로세서 수를 저장합니다.

우리는 `tick broadcast` 프레임워크에서 6 개의 `cpumasks`를 초기화했으며, 이제 이 프레임 워크의 구현을 진행할 수 있습니다.

`tick broadcast` 프레임워크
--------------------------------------------------------------------------------

하드웨어는 일부 clock 소스 장치를 제공 할 수 있습니다. 프로세서가 sleep 상태이고 로컬 타이머가 중지 된 경우 프로세서를 깨울 추가 clock 소스 장치가 있어야합니다. 리눅스 커널은 이 특별한 clock 소스 장치를 사용하여 지정된 시간에 인터럽트를 발생시킬 수 있습니다. 우리는 이미 리눅스 커널에서 `clock 이벤트` 장치라는 타이머를 알고 있습니다. `clock 이벤트` 장치 외에도 시스템의 각 프로세서에는 자체 지연 타이머가 있어 다음 지연 된 작업 시간에 인터럽트를 발생시키도록 프로그래밍 되어 있습니다. 또한이 타이머는 `jiffies` 등의 업데이트와 같은 주기적 작업을 수행하도록 프로그래밍 할 수 있습니다. 이 타이머는 Linux 커널에서 `tick_device` 구조체로 표시됩니다. 이 구조체는 [kernel/time/tick-sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tick-sched.h) 헤더 파일에 정의되어 있으며 다음과 같습니다. :

```C
struct tick_device {
        struct clock_event_device *evtdev;
        enum tick_device_mode mode;
};
```

`tick_device` 구조체에는 두 개의 필드가 포함되어 있습니다. 첫 번째 필드 인 `evtdev`는 [include/linux/clockchips.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clockchips.h) 헤더 파일에 정의 된 `clock_event_device` 구조체에 대한 포인터를 나타내며 clock 이벤트 장치의 디스크립터를 나타냅니다. `clock 이벤트` 장치를 사용하면 앞으로 일어날 이벤트를 등록 할 수 있습니다. 이미 쓴 것처럼, 이 파트에서는 `clock_event_device` 구조체와 관련 API를 고려하지 않고, 다음 파트에서 볼 것입니다.

`tick_device` 구조체의 두 번째 필드는 `tick_device`의 모드를 나타냅니다. 우리가 이미 알고 있듯이, 모드는 다음 중 하나 일 수 있습니다. :

```C
enum tick_device_mode {
        TICKDEV_MODE_PERIODIC,
        TICKDEV_MODE_ONESHOT,
};
```

시스템의 각 `clock 이벤트` 디바이스는 Linux 커널의 초기화 프로세스 중 `clockevents_register_device` 함수 또는 `clockevents_config_and_register`함수를 호출하여 자체적으로 등록됩니다. 새로운`clock events` 장치를 등록하는 동안 Linux 커널은 [kernel/time/tick-common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/tick-common.c) 소스 코드 파일에 정의 된 `tick_check_new_device` 함수를 호출하고, 주어진 `clock 이벤트` 장치가 Linux 커널에서 사용됩니다. 모든 확인 후 `tick_check_new_device` 함수는 다음을 호출합니다. :

```C
tick_install_broadcast_device(newdev);
```

이 함수는 주어진 `clock 이벤트` 장치가 브로드 캐스트 장치일 수 있는지 확인하고 주어진 장치가 브로드 캐스트 장치 일 수 있는 경우 이를 설치합니. `tick_install_broadcast_device` 함수의 구현을 살펴 봅시다. :

```C
void tick_install_broadcast_device(struct clock_event_device *dev)
{
	struct clock_event_device *cur = tick_broadcast_device.evtdev;

	if (!tick_check_broadcast_device(cur, dev))
		return;

	if (!try_module_get(dev->owner))
		return;

	clockevents_exchange_device(cur, dev);

	if (cur)
		cur->event_handler = clockevents_handle_noop;

	tick_broadcast_device.evtdev = dev;

	if (!cpumask_empty(tick_broadcast_mask))
		tick_broadcast_start_periodic(dev);

	if (dev->features & CLOCK_EVT_FEAT_ONESHOT)
		tick_clock_notify();
}
```

우선 `tick_broadcast_device`에서 현재 `clock event` 장치를 얻습니다. [kernel/time/tick-common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/tick-common.c) 소스 코드 파일에 `tick_broadcast_device` 가 정의되어 있습니다. :

```C
static struct tick_device tick_broadcast_device;
```

이는 프로세서의 이벤트를 추적하는 외부 클럭 장치를 나타냅니다. 현재 clock 장치를 얻은 후 첫 번째 단계는 주어진 clock 이벤트 장치가 브로드 캐스트 장치로 사용될 수 있는지 확인하는 `tick_check_broadcast_device` 함수의 호출입니다. `tick_check_broadcast_device` 함수의 중요한 포인트는 주어진 `clock events` 장치의 `features` 필드 값을 확인하는 것입니다. 이 필드의 이름에서 알 수 있듯이 `features` 필드에는 클럭 이벤트 장치의 특징들이 포함되어 있습니다. 사용가능한 값은 [include/linux/clockchips.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clockchips.h) 헤더 파일에 정의 되어있고, 주기적인 이벤트 등을 지원하는 clock 이벤트 장치를 나타내는 `CLOCK_EVT_FEAT_PERIODIC` 중 하나가 될 수 있습니다. 따라서, `tick_check_broadcast_device` 함수는 `CLOCK_EVT_FEAT_ONESHOT` 와 `CLOCK_EVT_FEAT_DUMMY` 와 기타 플래그에 대해 `features` 플래그를 확인하고, 주어진 클럭 이벤트 장치에 이러한 함수 중 하나가 있으면 `false`를 리턴합니다. 다른 방법으로 `tick_check_broadcast_device` 함수는 주어진 클럭 이벤트 장치와 현재 클럭 이벤트 장치의 `ratings`를 비교하여 더 좋은 것을 리턴합니다.

`tick_check_broadcast_device` 함수 이후, 우리는 클럭 이벤트의 모듈 소유자를 검사하는 `try_module_get` 함수의 호출을 볼 수 있습니다. 주어진 `클럭 이벤트` 장치가 올바르게 초기화되었는지 확인해야합니다. 다음 단계는 [kernel/time/clockevents.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/clockevents.c)에 정의 된 `clockevents_exchange_device` 함수의 호출입니다. 소스 코드 파일 및 이전 클럭 이벤트 장치를 해제하고 이전 함수 핸들러를 더미 핸들러로 대체합니다.

`tick_install_broadcast_device` 함수의 마지막 단계에서 우리는 `tick_broadcast_mask`가 비어 있지 않은지 확인하고, `tick_broadcast_start_periodic` 함수의 호출로 주기적인 모드에서 주어진 `클럭 이벤트` 장치를 시작합니다. :

```C
if (!cpumask_empty(tick_broadcast_mask))
	tick_broadcast_start_periodic(dev);

if (dev->features & CLOCK_EVT_FEAT_ONESHOT)
	tick_clock_notify();
```

`tick_broadcast_mask`는 이 `clock events` 장치를 등록하는 동안 `clock events` 장치를 검사하는 `tick_device_uses_broadcast` 함수으로 채워집니다. :

```C
int cpu = smp_processor_id();

int tick_device_uses_broadcast(struct clock_event_device *dev, int cpu)
{
	...
	...
	...
	if (!tick_device_is_functional(dev)) {
		...
		cpumask_set_cpu(cpu, tick_broadcast_mask);
		...
	}
	...
	...
	...
}
```

`smp_processor_id` 매크로에 대한 자세한 내용은 Linux 커널 초기화 프로세스 챕터의 네 번째 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-4)에서 읽을 수 있습니다.

`tick_broadcast_start_periodic` 함수는 주어진 `clock event` 장치를 점검하고 `tick_setup_periodic` 함수를 호출합니다. :

```
static void tick_broadcast_start_periodic(struct clock_event_device *bc)
{
	if (bc)
		tick_setup_periodic(bc, 1);
}
```

이 함수는 [kernel/time/tick-common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tick-common.c) 소스 코드 파일에 정의되어 있으며, 다음 함수를 호출하여 지정된 `클럭 이벤트` 장치의 브로드 캐스트 핸들러를 설정합니다. :

```C
tick_set_periodic_handler(dev, broadcast);
```

이 함수는 브로드 캐스트 상태(`on` 또는 `off`)를 나타내는 두 번째 매개 변수를 확인하고 브로드 캐스트 핸들러가 해당 값에 따라 설정되도록 합니다.

```C
void tick_set_periodic_handler(struct clock_event_device *dev, int broadcast)
{
	if (!broadcast)
		dev->event_handler = tick_handle_periodic;
	else
		dev->event_handler = tick_handle_periodic_broadcast;
}
```
`clock event` 장치가 인터럽트를 발생시키면 `dev->event_handler`가 호출됩니다. 예를 들어, [arch/x86/kernel/hpet.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/hpet.c) 소스 코드 파일에 있는 [고정밀 이벤트 타이머](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)의 인터럽트 핸들러를 살펴 보겠습니다.

```C
static irqreturn_t hpet_interrupt_handler(int irq, void *data)
{
	struct hpet_dev *dev = (struct hpet_dev *)data;
	struct clock_event_device *hevt = &dev->evt;

	if (!hevt->event_handler) {
		printk(KERN_INFO "Spurious HPET timer interrupt on HPET timer %d\n",
				dev->num);
		return IRQ_HANDLED;
	}

	hevt->event_handler(hevt);
	return IRQ_HANDLED;
}
```

`hpet_interrupt_handler`는 [irq](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 특정 데이터를 가져 와서 `클럭 이벤트` 장치의 이벤트 핸들러를 확인합니다. 우리는 방금 `tick_set_periodic_handler` 함수에서 설정했음을 떠올릴 수 있습니다. 따라서 고정밀 이벤트 타이머 인터럽트 핸들러의 끝에서 `tick_handler_periodic_broadcast` 함수가 호출됩니다.

`tick_handler_periodic_broadcast` 함수는 다음 함수를 호출합니다. :

```C
bc_local = tick_do_periodic_broadcast();
```

이 함수는 임시 `cpumask`에서 깨어나고 `tick_do_broadcast` 함수를 호출하도록 요청했던 프로세서의 수를 저장합니다. :

```
cpumask_and(tmpmask, cpu_online_mask, tick_broadcast_mask);
return tick_do_broadcast(tmpmask);
```

`tick_do_broadcast`는 주어진 클럭 이벤트의 `broadcast` 함수를 호출하여 프로세서 세트에 [IPI](https://en.wikipedia.org/wiki/Inter-processor_interrupt) 인터럽트를 보냅니다. 결국 우리는 주어진 `tick_device`의 이벤트 핸들러를 호출 할 수 있습니다. :

```C
if (bc_local)
	td->evtdev->event_handler(td->evtdev);
```

이것은 실제로 프로세서 로컬 타이머의 인터럽트 처리기를 나타냅니다. 이 후 프로세서가 깨어납니다. 이것이 리눅스 커널의 `tick broadcast` 프레임 워크에 관한 것입니다. 우리는 이 프레임 워크의 일부 측면, 예를 들어 `클럭 이벤트` 장치의 리프로그래밍과 oneshot 타이머 등의 브로드 캐스트와 같은 것들을 놓쳤습니다. Linux 커널은 매우 크기 때문에 우리는 모든 측면을 다루지는 않을 것입니다. 저는 여러분과 함께 뛰어드는 것이 흥미로울 것이라고 생각합니다.

우리는 `tick_init` 함수의 호출로 이 파트를 시작했습니다. 우리는 단순히 `tick_broadcast_init` 함수와 이와 관련된 이론만 살펴보지만, `tick_init` 함수는 또 다른 함수의 호출을 포함하며 이 함수는 `tick_nohz_init`입니다. 이 함수의 구현을 살펴봅시다.

데이터 구조체와 관련된 dyntick의 초기화
--------------------------------------------------------------------------------

우리는 이미 이 파트에서 `dyntick` 개념에 대한 정보를 보았으며, 이 개념으로 커널이 `idle` 상태에서 시스템 타이머 인터럽트를 비활성화 할 수 있다는 것을 알고 있습니다. `tick_nohz_init` 함수는 이 개념과 관련된 다른 데이터 구조체를 초기화합니다. 이 함수는 [kernel/time/tick-sched.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tich-sched.c) 소스 코드 파일에 정의되어 있으며 `idle` 상태를 위한 tick이 없는 모드의 상태와, 프로세서가 실행 가능한 작업이 하나만 있는 동안 시스템 타이머 인터럽트가 비활성화 된 상태를 나타내는 `tick_nohz_full_running` 변수의 값을 확인하는 것부터 시작합니다. :

```C
if (!tick_nohz_full_running) {
    if (tick_nohz_init_all() < 0)
    return;
}
```

이 모드가 실행되고 있지 않다면 동일한 소스 코드 파일에 정의 된 `tick_nohz_init_all` 함수를 호출하고 결과를 확인하세요. `tick_nohz_init_all` 함수는 `tick_nohz_full_mask`를 위한 공간을 할당하는 `alloc_cpumask_var`의 호출로 `tick_nohz_full_mask`를 할당하려고 시도합니다. `tick_nohz_full_mask`는 전체 `NO_HZ`를 활성화 한 프로세서 수를 저장합니다. `tick_nohz_full_mask`를 성공적으로 할당 한 후, `tick_nohz_full_mask`의 모든 비트를 설정하고 `tick_nohz_full_running`을 설정하고 결과를 `tick_nohz_init` 함수로 반환합니다.

```C
static int tick_nohz_init_all(void)
{
        int err = -1;
#ifdef CONFIG_NO_HZ_FULL_ALL
        if (!alloc_cpumask_var(&tick_nohz_full_mask, GFP_KERNEL)) {
                WARN(1, "NO_HZ: Can't allocate full dynticks cpumask\n");
                return err;
        }
        err = 0;
        cpumask_setall(tick_nohz_full_mask);
        tick_nohz_full_running = true;
#endif
        return err;
}
```

다음 단계에서 `housekeeping_mask`에 메모리 공간을 할당하려고 시도합니다. :

```C
if (!alloc_cpumask_var(&housekeeping_mask, GFP_KERNEL)) {
	WARN(1, "NO_HZ: Can't allocate not-full dynticks cpumask\n");
	cpumask_clear(tick_nohz_full_mask);
	tick_nohz_full_running = false;
	return;
}
```

이 `cpumask`는 `housekeeping`을 위한 프로세서의 수를 저장합니다. 즉, 우리는 `NO_HZ` 모드가 아닌 적어도 하나의 프로세서가 필요합니다. 왜냐하면 timekeeping과 같은 일들을 할 것이기 때문입니다. 이후에 우리는 아키텍처 별 `arch_irq_work_has_interrupt` 함수를 확인합니다. 이 함수는 특정 아키텍처에 대한 프로세서 간 인터럽트 전송 기능을 확인합니다. 프로세서의 시스템 타이머는 `NO_HZ`모드에서 비활성화되므로, 프로세서 간 인터럽트를 보내 오프라인 프로세서를 깨울 수있는 하나 이상의 온라인 프로세서가 있어야하므로 이를 확인해야합니다. 이 함수는 [x86_64](https://en.wikipedia.org/wiki/X86-64)에 대해 [arch/x86/include/asm/irq_work.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irq_work.h) 헤더 파일에 정의되어 있으며, 프로세서가 [CPUID](https://en.wikipedia.org/wiki/CPUID)에서 [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)를 가지고 있는지 확인합니다. :

```C
static inline bool arch_irq_work_has_interrupt(void)
{
    return cpu_has_apic;
}
```

프로세서에 `APIC`가 없는 경우 Linux 커널은 경고 메시지를 출력하고 `tick_nohz_full_mask` cpumask를 지우고, 시스템에서 가능한 모든 프로세서 수를 `housekeeping_mask`에 복사 한 다음 `tick_nohz_full_running` 변수의 값을 재설정합니다. :

```C
if (!arch_irq_work_has_interrupt()) {
	pr_warning("NO_HZ: Can't run full dynticks because arch doesn't "
		   "support irq work self-IPIs\n");
	cpumask_clear(tick_nohz_full_mask);
	cpumask_copy(housekeeping_mask, cpu_possible_mask);
	tick_nohz_full_running = false;
	return;
}
```

이 단계 후에, 우리는 `smp_processor_id`의 호출에 의해 현재 프로세서의 번호를 얻고 `tick_nohz_full_mask`에서 이 프로세서를 확인합니다. `tick_nohz_full_mask`에 주어진 프로세서가 포함되어 있으면 `tick_nohz_full_mask`에서 적절한 비트를 지웁니다. :

```C
cpu = smp_processor_id();

if (cpumask_test_cpu(cpu, tick_nohz_full_mask)) {
	pr_warning("NO_HZ: Clearing %d from nohz_full range for timekeeping\n", cpu);
	cpumask_clear_cpu(cpu, tick_nohz_full_mask);
}
```

이 프로세서는 timekeeping에 사용되기 때문입니다. 이 단계 후에 우리는 `tick_nohz_full_mask`가 아니라 `cpu_possible_mask`에 있는 모든 수의 프로세서를 넣습니다. :

```C
cpumask_andnot(housekeeping_mask,
	       cpu_possible_mask, tick_nohz_full_mask);
```

이 작업 후에 `housekeeping_mask`에는 timekeeping을 위한 프로세서를 제외한 모든 시스템 프로세서가 포함됩니다. `tick_nohz_init_all` 함수의 마지막 단계에서, 우리는 `tick_nohz_full_mask`에 정의 된 모든 프로세서를 살펴보고 각 프로세서에 대해 다음 함수를 호출합니다. :

```C
for_each_cpu(cpu, tick_nohz_full_mask)
	context_tracking_cpu_set(cpu);
```

[kernel/context_tracking.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/context_tracking.c) 소스 코드 파일에 정의 된 `context_tracking_cpu_set` 함수와 이 함수의 중요 포인트는`context_tracking.active` [percpu](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-1) 변수를 `true`로 설정한다는 것입니다. 특정 프로세서에 대해 `active` 필드가 `true`로 설정되면 이 프로세서의 Linux 커널 컨텍스트 추적 서브시스템에서 모든 [context switches](https://en.wikipedia.org/wiki/Context_switch)를 무시합니다.

이제 끝났습니다. 이것은 `tick_nohz_init` 함수의 끝입니다. 이 `NO_HZ` 이후에 관련된 데이터 구조체가 초기화됩니다. 우리는 `NO_HZ` 모드의 API를 보지 못했지만 곧 보게 될 것입니다.

결론
--------------------------------------------------------------------------------

이 챕터의 세 번째 파트는 Linux 커널의 타이머와 타이머 관리 관련 내용을 설명합니다. 이전 파트에서는 인터럽트와 하드웨어 특성에 독립적으로 다른 클럭 소스를 관리하기위한 프레임 워크를 나타내는 Linux 커널의 `clocksource`개념에 대해 알게 되었습니다. 우리는 이 부분에서 시간 관리 컨텍스트에서 Linux 커널 초기화 프로세스를 계속 살펴 보았고 우리에게 두 가지 새로운 개념, 즉 `tick broadcast` 프레임 워크와 `tick-less` 모드에 대해 알게되었습니다. 첫 번째 개념은 리눅스 커널이 sleep 상태의 프로세서를 처리하는 데 도움이 되고, 두 번째 개념은 커널이 idle 프로세서의 전원 관리를 개선하기 위해 작동 할 수있는 모드를 나타냅니다.

다음 파트에서 우리는 계속해서 리눅스 커널에서의 타이머 관리 관련 내용을 살펴보고 새로운 개념인 `타이머`를 보게 될 것입니다.

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견이나 핑을 남겨주시거나. [email](anotherworldofworld@gmail.com) 보내주시거나, [issue](https://github.com/0xAX/linux-insides/issues/new)를 만들어주세요.

**영어는 제 모국어가 아닙니다. 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수를 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한글 번역](https://github.com/junsooo/linux-insides-ko)으로 PR을 보내주세요.**

Links
-------------------------------------------------------------------------------

* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [interrupt](https://en.wikipedia.org/wiki/Interrupt)
* [DMI](https://en.wikipedia.org/wiki/Desktop_Management_Interface)
* [printk](https://en.wikipedia.org/wiki/Printk)
* [CPU idle](https://en.wikipedia.org/wiki/Idle_%28CPU%29)
* [power management](https://en.wikipedia.org/wiki/Power_management)
* [NO_HZ documentation](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/timers/NO_HZ.txt)
* [cpumasks](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [high precision event timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)
* [irq](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [IPI](https://en.wikipedia.org/wiki/Inter-processor_interrupt)
* [CPUID](https://en.wikipedia.org/wiki/CPUID)
* [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [percpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [context switches](https://en.wikipedia.org/wiki/Context_switch)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-2.html)
