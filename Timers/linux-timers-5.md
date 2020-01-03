Linux 커널의 타이머 및 시간 관리 5 부.
================================================================================

`clockevents` 프레임 워크 소개
--------------------------------------------------------------------------------

이것은 리눅스 커널에서 타이머 및 시간 관리 관련 사항을 설명하는 [chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)의 다섯 번째 부분입니다. 이 부분의 제목에서 알 수 있듯이 `clockevents` 프레임 워크가 논의됩니다. 우리는 이미 이 장의 [두번째](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-2.html) 부분에서 하나의 프레임 워크를 보았습니다. `clocksource` 프레임 워크였습니다. 이 두 프레임 워크는 모두 Linux 커널에서 시간을 유지하는 추상화를 나타냅니다.

처음에는 메모리를 새로 고치고 그것이 `clocksource` 프레임 워크와 그 목적이 무엇인지 기억하려고 노력하십시오. `clocksource` 프레임 워크의 주요 목표는 [documentation](https://github.com/0xAX/linux/blob/0a07b238e5f488b459b6113a62e06b6aab017f71/Documentation/timers/timekeeping.txt)에 설명 된대로 `타임 라인`을 제공하는 것입니다. :

> 예를 들어 Linux 시스템에서 `date`명령을 실행하면 clcok source를 읽어 정확한 시간을 결정합니다.

Linux 커널은 다양한 클럭 소스를 지원합니다. 이들 중 일부는 [drivers/clocksource](https://github.com/torvalds/linux/tree/master/drivers/clocksource)에서 찾을 수 있습니다. 예를 들어 오래된 좋은 `1193182` Hz 주파수의  [Intel 8253](https://en.wikipedia.org/wiki/Intel_8253)-[프로그램 간격 타이머](https://en.wikipedia.org/wiki/Programmable_interval_timer), `3579545` Hz 주파수의 [ACPI PM](http://uefi.org/sites/default/files/resources/ACPI_5.pdf) 타이머 등. [drivers/clocksource](https://github.com/torvalds/linux/tree/master/drivers/clocksource) 디렉토리 외에도 각 아키텍처는 고유 한 아키텍처 별 클럭 소스를 제공 할 수 있습니다. 예를 들어 [x86](https://en.wikipedia.org/wiki/X86) 아키텍처는 [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)를 제공하거나 [powerpc](https://en.wikipedia.org/wiki/PowerPC)에서는 `timebase` 레지스터를 통해 프로세서 타이머에 액세스 할 수 있습니다.

각 클럭 소스는 단조 원자 카운터를 제공합니다. 이미 쓴 것처럼 Linux 커널은 다양한 클럭 소스 세트를 지원하며 각 클럭 소스에는 [frequency](https://en.wikipedia.org/wiki/Frequency)와 같은 자체 매개 변수가 있습니다. `clocksource` 프레임 워크의 주요 목표는 [API](https://en.wikipedia.org/wiki/Application_programming_interface)를 제공하여 시스템에서 사용 가능한 최상의 클럭 소스, 즉 가장 높은 주파수를 가진 클럭 소스를 선택하는 것입니다. `clocksource` 프레임 워크의 추가 목표는 인간 단위로 클럭 소스가 제공하는 원자 카운터를 나타내는 것입니다. 이 시점에서, 나노 초는 Linux 커널에서 주어진 클럭 소스의 시간 값 단위로 선호되는 선택입니다.

[include/linux/clocksource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h) 에 정의 된 `clocksource` 구조로 표시되는 `clocksource` 프레임 워크는 클럭 소스의 이름, 시스템의 특정 클럭 소스 등급 (주파수가 높은 클럭 소스는 시스템에서 가장 높은 등급을 가짐), 시스템에 등록 된 모든 클럭 소스의 `목록`과, `enable` 및 `disable` 필드는 클럭 소스를 활성화 및 비활성화하며, 클럭 소스의 원자 카운터 등을 반환해야하는 `read`함수에 대한 포인터를 포함합니다.

또한 `clocksource` 구조는 `mult`와 `shift`라는 특정 클록 소스에 의해 인간 단위로 제공되는 원자 카운터의 변환에 필요한 두 개의 필드를 제공합니다.

ex) [nanoseconds](https://en.wikipedia.org/wiki/Nanosecond).

```
ns ~= (clocksource * mult) >> shift
```

우리가 이미 알고 있듯이`clocksource` 구조 외에 `clocksource` 프레임 워크는 다른 주파수 스케일 팩터로 클록 소스를 등록하기 위한 API를 제공합니다.

```C
static inline int clocksource_register_hz(struct clocksource *cs, u32 hz)
static inline int clocksource_register_khz(struct clocksource *cs, u32 khz)
```

클럭 소스 등록 취소 :

```C
int clocksource_unregister(struct clocksource *cs)
```

등

`clocksource` 프레임 워크 외에도 Linux 커널은 `clockevents` 프레임 워크를 제공합니다. [documentation](https://github.com/0xAX/linux/blob/0a07b238e5f488b459b6113a62e06b6aab017f71/Documentation/timers/timekeeping.txt)에 설명 된대로 :

> 클럭 이벤트는 클럭 소스의 개념적 역전입니다

의 주요 목표는 시계 이벤트 장치 또는 다른 말로 이벤트를 등록 할 수있는 장치를 관리하는 것입니다. 즉, [interrupt](https://en.wikipedia.org/wiki/Interrupt) 미래의 정해진 시점에 발생합니다.

이제 우리는 리눅스 커널의 `clockevents` 프레임 워크에 대해 조금 알고 있으며 이제는 [API](https://en.wikipedia.org/wiki/Application_programming_interface)를 볼 차례입니다.

`clockevents` 프레임워크의 API
-------------------------------------------------------------------------------

클럭 이벤트 장치를 설명하는 주요 구조체는 `clock_event_device` 구조체입니다. 이 구조는 [include/linux/clockchips.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clockchips.h) 헤더 파일에 정의되어 있으며 거대한 필드 세트를 포함합니다. `clocksource` 구조뿐만 아니라 사람이 읽을 수 있는 시계 이벤트 장치 이름을 포함하는 `name` 필드가 있습니다 (예 : [local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) timer).

```C
static struct clock_event_device lapic_clockevent = {
    .name                   = "lapic",
    ...
    ...
    ...
}
```

`event_handler`, `set_next_event`, `next_event`의 주소는 [인터럽트 핸들러](https://en.wikipedia.org/wiki/Interrupt_handler) 인 특정 시계 이벤트 장치에 대한 기능이며, 다음 이벤트의 설정자 및 로컬 다음 이벤트를 위한 스토리지 설정입니다. `clock_event_device` 구조의 또 다른 필드는 `features` 필드입니다. 그 가치는 다음과 같은 일반적인 기능 중 하나 일 수 있습니다.

```C
#define CLOCK_EVT_FEAT_PERIODIC	0x000001
#define CLOCK_EVT_FEAT_ONESHOT		0x000002
```

여기서 `CLOCK_EVT_FEAT_PERIODIC`은 이벤트를 주기적으로 생성하도록 프로그래밍 될 수있는 장치를 나타냅니다. `CLOCK_EVT_FEAT_ONESHOT`는 이벤트를 한 번만 생성 할 수있는 장치를 나타냅니다. 이 두 기능 외에도 아키텍처 별 기능도 있습니다. 예를 들어 [x86_64](https://en.wikipedia.org/wiki/X86-64)는 두 가지 추가 기능을 지원합니다.

```C
#define CLOCK_EVT_FEAT_C3STOP		0x000008
```

첫 번째 `CLOCK_EVT_FEAT_C3STOP`은 [C3](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface#Device_states) 상태에서 시계 이벤트 장치가 중지됨을 의미합니다. 또한 `clock_event_device` 구조에는 `clocksource` 구조뿐만 아니라 `mult` 및 `shift` 필드가 있습니다. `clocksource` 구조는 다른 필드도 포함하지만 나중에 고려할 것입니다.

`clock_event_device` 구조의 일부를 고려한 후, `clockevents` 프레임 워크의 `API`를 살펴 볼 시간입니다. 시계 이벤트 장치를 사용하려면 먼저 clock_event_device 구조를 초기화하고 시계 이벤트 장치를 등록해야합니다. `clockevents` 프레임 워크는 클록 이벤트 장치의 등록을 위해 다음과 같은 API를 제공합니다.

```C
void clockevents_register_device(struct clock_event_device *dev)
{
   ...
   ...
   ...
}
```

이 함수는 [kernel/time/clockevents.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/clockevents.c) 소스 코드 파일에 정의되어 있습니다. `clockevents_register_device` 함수는 하나의 매개 변수 만 사용합니다.

* 클럭 이벤트 장치를 나타내는`clock_event_device` 구조의 주소.

따라서 클럭 이벤트 장치를 등록하려면 먼저 특정 클럭 이벤트 장치의 매개 변수로 `clock_event_device` 구조를 초기화해야합니다. 리눅스 커널 소스 코드에서 하나의 랜덤 클럭 이벤트 장치를 살펴봅시다. [drivers/clocksource](https://github.com/torvalds/linux/tree/master/drivers/clocksource) 디렉토리에서 찾거나 아키텍처 별 시계 이벤트 장치를 살펴보십시오. 예를 들어 [at91sam926x 용 PIT (Periodic Interval Timer)](http://www.atmel.com/Images/doc6062.pdf)를 예로 들어 보겠습니다. 구현은 [drivers/ locksource](https://github.com/torvalds/linux/tree/master/drivers/clocksource/timer-atmel-pit.c)에서 찾을 수 있습니다.

우선`clock_event_device` 구조의 초기화를 살펴 봅시다. 이것은 `at91sam926x_pit_common_init` 함수에서 발생합니다 :

```C
struct pit_data {
    ...
    ...
    struct clock_event_device       clkevt;
    ...
    ...
};

static void __init at91sam926x_pit_common_init(struct pit_data *data)
{
    ...
    ...
    ...
    data->clkevt.name = "pit";
    data->clkevt.features = CLOCK_EVT_FEAT_PERIODIC;
    data->clkevt.shift = 32;
    data->clkevt.mult = div_sc(pit_rate, NSEC_PER_SEC, data->clkevt.shift);
    data->clkevt.rating = 100;
    data->clkevt.cpumask = cpumask_of(0);

    data->clkevt.set_state_shutdown = pit_clkevt_shutdown;
    data->clkevt.set_state_periodic = pit_clkevt_set_periodic;
    data->clkevt.resume = at91sam926x_pit_resume;
    data->clkevt.suspend = at91sam926x_pit_suspend;
    ...
}
```

여기서 우리는 `at91sam926x_pit_common_init`가 하나의 매개 변수를 취한다는 것을 알 수 있습니다. `at91sam926x`의 주기 이벤트 관련 정보를 포함하는 `clock_event_device` 구조체를 포함하는 `pit_data` 구조체에 대한 포인터 [주기 간격 타이머](https://en.wikipedia.org/wiki/Programmable_interval_timer). 처음에는 타이머 장치의 이름과 기능을 채 웁니다. 우리의 경우 우리가 이미 알고 있듯이 주기적으로 이벤트를 생성하도록 프로그래밍 될 수 있는 주기적인 타이머를 처리합니다.

다음 두 필드 인 `shift`와 `mult`는 우리에게 친숙합니다. 타이머 카운터를 나노초로 변환하는 데 사용됩니다. 그런 다음 타이머 등급을 `100`으로 설정했습니다. 즉, 시스템에 등급이 높은 타이머가없는 경우이 타이머는 시간 표시에 사용됩니다. 다음 필드 인 `cpumask`는 시스템에서 장치가 작동 할 프로세서를 나타냅니다. 우리의 경우 장치는 첫 번째 프로세서에서 작동합니다. [include/linux/cpumask.h](https://github.com/torvalds/linux/tree/master/include/linux/cpumask.h) 헤더 파일에 정의 된 `cpumask_of` 매크로는 호출로 확장됩니다.

```C
#define cpumask_of(cpu) (get_cpu_mask(cpu))
```

여기서 `get_cpu_mask`는 주어진 `cpu` 번호를 포함하는 cpumask를 반환합니다. `cpumasks` 개념에 대한 자세한 내용은 [Linux 커널의 CPU 마스크](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html) 부분에서 읽을 수 있습니다. 마지막 4 줄의 코드에서 시계 이벤트 장치 일시 중지 / 재개, 장치 종료 및 시계 이벤트 장치 상태 업데이트에 대한 콜백을 설정했습니다.

`at91sam926x` 주기 타이머 초기화를 완료 한 후 다음 함수를 호출하여 등록 할 수 있습니다.

```C
clockevents_register_device(&data->clkevt);
```

이제 우리는 `clockevent_register_device` 함수의 구현을 고려할 수 있습니다. 위에서 이미 작성했듯이이 함수는 [kernel/time/clockevents.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/clockevents.c) 소스 코드 파일 및 초기 이벤트 장치 상태의 초기화에서 시작합니다.

```C
clockevent_set_state(dev, CLOCK_EVT_STATE_DETACHED);
```

실제로 이벤트 장치는 다음 상태 중 하나 일 수 있습니다.

```C
enum clock_event_state {
	CLOCK_EVT_STATE_DETACHED,
	CLOCK_EVT_STATE_SHUTDOWN,
	CLOCK_EVT_STATE_PERIODIC,
	CLOCK_EVT_STATE_ONESHOT,
	CLOCK_EVT_STATE_ONESHOT_STOPPED,
};
```

* `CLOCK_EVT_STATE_DETACHED` - 시계 이벤트 장치는 `clockevents` 프레임 워크에서 사용되지 않습니다. 실제로 모든 시계 이벤트 장치의 초기 상태입니다.
* `CLOCK_EVT_STATE_SHUTDOWN` - 시계 이벤트 장치의 전원이 꺼져 있습니다.
* `CLOCK_EVT_STATE_PERIODIC` - 클록 이벤트 장치는 주기적으로 이벤트를 생성하도록 프로그래밍 될 수 있습니다.
* `CLOCK_EVT_STATE_ONESHOT` - 클럭 이벤트 장치는 이벤트를 한 번만 생성하도록 프로그래밍 될 수 있습니다.
* `CLOCK_EVT_STATE_ONESHOT_STOPPED` - 시계 이벤트 장치가 이벤트를 한 번만 생성하도록 프로그래밍되었으며 이제 일시적으로 중지되었습니다.

`clock_event_set_state` 함수의 구현은 매우 쉽습니다.

```C
static inline void clockevent_set_state(struct clock_event_device *dev,
					enum clock_event_state state)
{
	dev->state_use_accessors = state;
}
```

보시다시피, 주어진`clock_event_device` 구조체의 `state_use_accessors` 필드를 주어진 값으로 채웁니다. 이 경우에는 `CLOCK_EVT_STATE_DETACHED` 입니다. 실제로 모든 시계 이벤트 장치는 등록 중에 이 초기 상태를 갖습니다. `clock_event_device` 구조의 `state_use_accessors` 필드는 클럭 이벤트 장치의 현재 상태를 제공합니다.

주어진 `clock_event_device` 구조의 초기 상태를 설정 한 후 주어진 시계 이벤트 장치의 `cpumask`가 0이 아닌지 확인합니다.

```C
if (!dev->cpumask) {
	WARN_ON(num_possible_cpus() > 1);
	dev->cpumask = cpumask_of(smp_processor_id());
}
```

`at91sam926x` 주기 타이머의 `cpumask`를 첫 번째 프로세서로 설정했음을 기억하십시오. 만약 `cpumask` 필드가 0이라면, 우리는 시스템에서 가능한 프로세서의 수를 확인하고 경고 메시지가 on보다 작 으면 인쇄합니다. 또한 주어진 클럭 이벤트 장치의 `cpumask`를 현재 프로세서로 설정합니다. `smp_processor_id` 매크로가 어떻게 구현되는지에 관심이 있다면, 리눅스 커널 초기화 과정 장의 네 번째 [part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)에서 더 자세히 읽을 수 있습니다.

이 검사 후 매크로 다음 호출에 의해 시계 이벤트 장치 등록의 실제 코드를 잠급니다.

```C
raw_spin_lock_irqsave(&clockevents_lock, flags);
...
...
...
raw_spin_unlock_irqrestore(&clockevents_lock, flags);
```

또한 `raw_spin_lock_irqsave` 및 `raw_spin_unlock_irqrestore` 매크로는 로컬 인터럽트를 비활성화하지만 다른 프로세서의 인터럽트는 여전히 발생할 수 있습니다. 클록 이벤트 디바이스 목록에 새 클록 이벤트 디바이스를 추가하고 다른 클록 이벤트 디바이스에서 인터럽트가 발생하는 경우 잠재적 [교착 상태](https://en.wikipedia.org/wiki/Deadlock)를 방지하기 위해 이를 수행해야합니다.

`raw_spin_lock_irqsave`와 `raw_spin_unlock_irqrestore` 매크로 사이에 다음과 같은 시계 이벤트 장치 등록 코드를 볼 수 있습니다.

```C
list_add(&dev->list, &clockevent_devices);
tick_check_new_device(dev);
clockevents_notify_released();
```

우선, 주어진 clock 이벤트 장치를 `clockevent_devices`로 표시되는 시계 이벤트 장치 목록에 추가합니다.

```C
static LIST_HEAD(clockevent_devices);
```

다음 단계에서 우리는 [kernel/time/tick-common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tick-common.c) 소스 코드 파일에 정의 된`tick_check_new_device` 함수를 호출합니다. 소스코드 파일 및 검사는 새로운 등록 된 시계 이벤트 장치의 사용 여부를 확인합니다. `tick_check_new_device` 함수는 주어진 `clock_event_device`가 `tick_device` 구조로 표시되는 현재 등록 된 틱 장치를 가져 와서 등급과 기능을 비교합니다. 실제로 `CLOCK_EVT_STATE_ONESHOT`이 선호됩니다 :

```C
static bool tick_check_preferred(struct clock_event_device *curdev,
				 struct clock_event_device *newdev)
{
	if (!(newdev->features & CLOCK_EVT_FEAT_ONESHOT)) {
		if (curdev && (curdev->features & CLOCK_EVT_FEAT_ONESHOT))
			return false;
		if (tick_oneshot_mode_active())
			return false;
	}

	return !curdev ||
		newdev->rating > curdev->rating ||
	       !cpumask_equal(curdev->cpumask, newdev->cpumask);
}
```

새로 등록 된 시계 이벤트 장치가 이전 틱 장치보다 선호되는 경우 이전 및 새 등록 장치를 교환하고 새 장치를 설치합니다.

```C
clockevents_exchange_device(curdev, newdev);
tick_setup_device(td, newdev, cpu, cpumask_of(cpu));
```

`clockevents_exchange_device` 기능이 해제되거나 다른 말로하면 `clockevent_devices` 목록에서 기존 시계 이벤트 장치가 삭제되었습니다. 다음 함수 인 `tick_setup_device`는 이름에서 알 수 있듯이 새로운 틱 장치를 설정합니다. 이 기능은 새로 등록 된 시계 이벤트 장치의 모드를 확인하고 틱 장치 모드에 따라 `tick_setup_periodic` 함수 또는 `tick_setup_oneshot`을 호출합니다.

```C
if (td->mode == TICKDEV_MODE_PERIODIC)
	tick_setup_periodic(newdev, 0);
else
	tick_setup_oneshot(newdev, handler, next_event);
```

이 함수는 모두 시계 이벤트 장치의 상태를 변경하기 위해 `clockevents_switch_state`를 호출하고 다음 이벤트의 최대 및 최소 차이 현재 시간과 시간 사이의 델타를 기반으로 시계 이벤트 장치의 다음 이벤트를 설정하기 위해 `clockevents_program_event` 함수를 호출합니다. `tick_setup_periodic` :

```C
clockevents_switch_state(dev, CLOCK_EVT_STATE_PERIODIC);
clockevents_program_event(dev, next, false))
```

그리고`tick_setup_oneshot_periodic` :

```C
clockevents_switch_state(newdev, CLOCK_EVT_STATE_ONESHOT);
clockevents_program_event(newdev, next_event, true);
```

`clockevents_switch_state` 함수는 시계 이벤트 장치가 주어진 상태에 있지 않은지 확인하고 동일한 소스 코드 파일에서 `__clockevents_switch_state` 함수를 호출합니다.

```C
if (clockevent_get_state(dev) != state) {
	if (__clockevents_switch_state(dev, state))
		return;
```

`__clockevents_switch_state` 함수는 주어진 상태에 따라 특정 콜백을 호출합니다.

```C
static int __clockevents_switch_state(struct clock_event_device *dev,
				      enum clock_event_state state)
{
	if (dev->features & CLOCK_EVT_FEAT_DUMMY)
		return 0;

	switch (state) {
	case CLOCK_EVT_STATE_DETACHED:
	case CLOCK_EVT_STATE_SHUTDOWN:
		if (dev->set_state_shutdown)
			return dev->set_state_shutdown(dev);
		return 0;

	case CLOCK_EVT_STATE_PERIODIC:
		if (!(dev->features & CLOCK_EVT_FEAT_PERIODIC))
			return -ENOSYS;
		if (dev->set_state_periodic)
			return dev->set_state_periodic(dev);
		return 0;
    ...
    ...
    ...
```

`at91sam926x` 주기 타이머의 경우 상태는`CLOCK_EVT_FEAT_PERIODIC`입니다.

```C
data->clkevt.features = CLOCK_EVT_FEAT_PERIODIC;
data->clkevt.set_state_periodic = pit_clkevt_set_periodic;
```

따라서`pit_clkevt_set_periodic` 콜백이 호출됩니다. [at91sam926x 용 PIT (Periodic Interval Timer)](http://www.atmel.com/Images/doc6062.pdf) 문서 를 읽으면 `주기 간격 타이머 모드 레지스터`가 있음을 알 수 있습니다. 주기적 간격 타이머를 제어 할 수 있습니다.

다음과 같습니다.

```
31                                                   25        24
+---------------------------------------------------------------+
|                                          |  PITIEN  |  PITEN  |
+---------------------------------------------------------------+
23                            19                               16
+---------------------------------------------------------------+
|                             |               PIV               |
+---------------------------------------------------------------+
15                                                              8
+---------------------------------------------------------------+
|                            PIV                                |
+---------------------------------------------------------------+
7                                                               0
+---------------------------------------------------------------+
|                            PIV                                |
+---------------------------------------------------------------+
```

`PIV`또는 `Periodic Interval Value` - 주기 간격 타이머의 기본 `20 비트` 카운터와 비교하여 값을 정의합니다. 비트가 `1`이면 `PITEN`또는 `주기 간격 타이머 사용`, 비트가 `1`이면 `PITIEN`또는 `주기 간격 타이머 인터럽트 사용`. 따라서 주기적 모드를 설정하려면 주기 간격 타이머 모드 레지스터에서 24 비트, 25 비트를 설정해야합니다. 그리고 우리는 `pit_clkevt_set_periodic` 기능에 그것을 하고 있습니다 :

```C
static int pit_clkevt_set_periodic(struct clock_event_device *dev)
{
        struct pit_data *data = clkevt_to_pit_data(dev);
        ...
        ...
        ...
        pit_write(data->base, AT91_PIT_MR,
                  (data->cycle - 1) | AT91_PIT_PITEN | AT91_PIT_PITIEN);

        return 0;
}
```

`AT91_PT_MR`, `AT91_PT_PITEN` 및 `AT91_PIT_PITIEN`은 다음과 같이 선언됩니다.
```C
#define AT91_PIT_MR             0x00
#define AT91_PIT_PITIEN       BIT(25)
#define AT91_PIT_PITEN        BIT(24)
```

새로운 시계 이벤트 장치의 설정이 끝나면 `clockevents_register_device` 기능으로 돌아갈 수 있습니다. 

`clockevents_register_device` 함수의 마지막 함수는 다음과 같습니다.
```C
clockevents_notify_released();
```

 함수는 해제 된 클럭 이벤트 장치를 포함하는 `clockevents_released` 목록을 확인합니다 ( `clockevents_exchange_device` 함수 호출 후 발생할 수 있음을 기억하십시오). 이 목록이 비어 있지 않으면 `clock_events_released` 목록에서 시계 이벤트 장치를 살펴보고 `clockevent_devices`에서 삭제합니다.

```C
static void clockevents_notify_released(void)
{
	struct clock_event_device *dev;

	while (!list_empty(&clockevents_released)) {
		dev = list_entry(clockevents_released.next,
				 struct clock_event_device, list);
		list_del(&dev->list);
		list_add(&dev->list, &clockevent_devices);
		tick_check_new_device(dev);
	}
}
```

그게 전부 입니다. 지금부터 새로운 시계 이벤트 장치를 등록했습니다. 따라서 `clockevents`프레임 워크의 사용법은 간단하고 명확합니다. 아키텍처는 시계 이벤트 코어에 시계 이벤트 장치를 등록했습니다. `clockevents` 코어 사용자는 시계 이벤트 장치를 사용할 수 있습니다. `clockevents` 프레임 워크는 등록되거나 등록되지 않은 시계 이벤트 장치와 같은 다양한 시계 관련 관리 이벤트에 대한 알림 메커니즘을 제공하며 프로세서는 [CPU hotplug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt) 등을 지원하는 시스템에서 오프라인 상태입니다.

우리는 `clockevents_register_device` 함수의 구현만을 보았습니다. 그러나 일반적으로 시계 이벤트 계층 [API](https://en.wikipedia.org/wiki/Application_programming_interface)는 작습니다. 클록 이벤트 장치 등록을위한 API 외에, 클록 이벤트 프레임 워크는 다음 이벤트 인터럽트, 클록 이벤트 장치 통지 서비스를 예약하고 클록 이벤트 장치의 일시 중단 및 재개를 지원하는 기능을 제공합니다.

`clockevents` API에 대해 더 알고 싶다면 다음 소스 코드와 헤더 파일을 연구 할 수 있습니다. [kernel/time/tick-common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/tick-common.c), [kernel/time/clockevents.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/clockevents.c) 및 [include/linux/clockchips.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clockchips.h).

결론
-------------------------------------------------------------------------------

이것은 리눅스 커널에서 타이머와 타이머 관리 관련 사항을 설명하는 [chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)의 다섯 번째 부분 입니다. 이전 부분에서는 `타이머` 개념에 대해 알게되었습니다. 이 부분에서 우리는 리눅스 커널에서 시간 관리 관련 내용을 계속 배웠고 또 다른 프레임 워크 인 `clockevents`에 대해 조금 보았습니다.

질문이나 제안 사항이 있으면 Twitter [0xAX](https://twitter.com/0xAX)에 핑(Ping)을 보내거나 [email](anotherworldofworld@gmail.com)을 보내거나 [issue](https://github.com/0xAX/linux-internals/issues/new)를 만드세요.

링크
-------------------------------------------------------------------------------

* [timekeeping documentation](https://github.com/0xAX/linux/blob/0a07b238e5f488b459b6113a62e06b6aab017f71/Documentation/timers/timekeeping.txt)
* [Intel 8253](https://en.wikipedia.org/wiki/Intel_8253)
* [programmable interval timer](https://en.wikipedia.org/wiki/Programmable_interval_timer)
* [ACPI pdf](http://uefi.org/sites/default/files/resources/ACPI_5.pdf)
* [x86](https://en.wikipedia.org/wiki/X86)
* [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)
* [powerpc](https://en.wikipedia.org/wiki/PowerPC)
* [frequency](https://en.wikipedia.org/wiki/Frequency)
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [nanoseconds](https://en.wikipedia.org/wiki/Nanosecond)
* [interrupt](https://en.wikipedia.org/wiki/Interrupt)
* [interrupt handler](https://en.wikipedia.org/wiki/Interrupt_handler)
* [local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [C3 state](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface#Device_states) 
* [Periodic Interval Timer (PIT) for at91sam926x](http://www.atmel.com/Images/doc6062.pdf)
* [CPU masks in the Linux kernel](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [deadlock](https://en.wikipedia.org/wiki/Deadlock)
* [CPU hotplug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
* [previous part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-3.html)
