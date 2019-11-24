리눅스 커널의 타이머 및 시간 관리 제2부
================================================================================

클럭소스에 대한 소개
--------------------------------------------------------------------------------

이전 부분은[part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html) 현재 장의 첫 부분으로 리눅스 커널의 타이머 및 시간 관리 관련 내용을 설명합니다.[chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html) 우리는 이전 부분에서 두가지 개념에 대해 알게 되었습니다.

  * `jiffies`
  * `clocksource`

첫 번째는 [include/linux/jiffies.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/jiffies.h) 헤더 파일에 정의 된 전역 변수이고 각 타이머 인터럽트 동안 증가하는 카운터를 나타냅니다. 따라서 이 전역 변수에 액세스 할 수 있고 타이머 인터럽트 속도를 알고 있다면 `jiffies`를 사람들의 시간 단위로 변환 할 수 있습니다. 컴파일 시간 상수로 대표되는 타이머 인터럽트 속도를 리눅스 커널에서  `HZ` 라고 부르는 것을 이미 알고 있습니다. `HZ`의 값는 CONFIG_HZ의 커널 구성 옵션 값과 동일하며 [arch/x86/configs/x86_64_defconfig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/configs/x86_64_defconfig) 그 커널 구성 파일을 확인해보겠습니다:

```
CONFIG_HZ_1000=y
```

커널 구성 옵션은 설정이 되었습니다. 이것은 `CONFIG_HZ` 는 아키텍쳐의 기본값에 의해 `1000` [x86_64](https://en.wikipedia.org/wiki/X86-64)이 된다는 것을 의미합니다. 그래서 만약 우리가 `jiffies`의 값을 `HZ`의 값으로 나눈다면:

```
jiffies / HZ
```

우리는 리눅스 커널이 작동하기 시작했을때부터 경과한 시간(초)나 [uptime](https://en.wikipedia.org/wiki/Uptime), 시스템의 가동시간을 얻습니다. `HZ`는 1초 마다 일어나는 타이머 인터럽트를 표현하므로 우리는 나중에 일어날 일정 시간동안의 값을 설정할수 있습니다. 예를 들어:

```C
/* one minute from now */
unsigned long later = jiffies + 60*HZ;

/* five minutes from now */
unsigned long later = jiffies + 5*60*HZ;
```

이것은 리눅스에서 매우 기본적인 연습입니다. 예를 들어 만약 당신이 [arch/x86/kernel/smpboot.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/smpboot.c)소스코드 파일을 찾아본다면 당신은 `do_boot_cpu` 함수를 찾을 것입니다. 이 함수는 부트 스트랩 프로세스 이외의 모든 프로세서를 부팅합니다. 당신은 애플리케이션 프로세서의 응답을 10초동안 기다리는 스니펫을 찾을 수 있습니다.

```C
if (!boot_error) {
	timeout = jiffies + 10*HZ;
	while (time_before(jiffies, timeout)) {
		...
		...
		...
		udelay(100);
	}
	...
	...
	...
}
```

우리는 `jiffies + 10*HZ` 의 값을 `timeout` 함수에게 할당합니다. 아마 이해하셨겠지만 이것은 10초의 대기시간을 의미합니다. 이후에 우리는 `time_before` 매크로를 이용하여 `jiffies` 의 값과 대기시간을 비교하는 루프를 시작합니다.
또는 예를 들어 [Ensoniq Soundscape Elite](https://en.wikipedia.org/wiki/Ensoniq_Soundscape_Elite)사운드카드의 드라이버를 나타내는 [sound/isa/sscape.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/sound/isa/sscape) 소스코드를 살펴보면 On_Board 프로세서가 시작 승인 시퀸스에 반환될때 까지 지정된 대기시간의 초과를 기다리는 `obp_startup_ack` 함수가 표시됩니다.

```C
static int obp_startup_ack(struct soundscape *s, unsigned timeout)
{
	unsigned long end_time = jiffies + msecs_to_jiffies(timeout);

	do {
		...
		...
		...
		x = host_read_unsafe(s->io_base);
		...
		...
		...
		if (x == 0xfe || x == 0xff)
			return 1;
		msleep(10);
	} while (time_before(jiffies, end_time));

	return 0;
}
```
볼수 있듯이 `jiffies` 변수는 Linux kernel [code](http://lxr.free-electrons.com/ident?i=jiffies)에서 매우 넓게 사용됩니다. 제가 이미 적어놓은 것 처럼 우리는 이전 부분의 관련 개념인 `clocksource`에 아직 접하지 않았습니다. 우리는 이 개념과 `clocksource` 등록을 위한 API에 대한 짧은 설명을 봤을 뿐입니다.

`clocksource`에 대한 소개
--------------------------------------------------------------------------------

`clocksource` 의 개념은 리눅스 커널에서 클럭소스 관리를 위한 일반적인 API를 나타냅니다. 왜 우리는 이렇게 나뉘어진 틀 구조가 필요한 걸까요? 처음으로 돌아가보죠. `time` 개념은 리눅스 커널과 다른 운영체제 커널에서 매우 기초적인 개념입니다. 그리고 시간 관리는 이 개념에 있어서는 필수적인 것들 중에 한가지입니다. 예를 들어 리눅스 커널은 시스템이 시작했을때부터 경과한 시간을 업데이트해야합니다. 그리고 그것은 현재 프로세스가 모든 프로세스에 대해서 얼마나 오래 실행됬는지를 결정해야합니다. 리눅스 커널은 어디에서 시간에 대한 정보를 얻을까요? 가장 먼저 비휘발성 장치로 표시되는 Real Time Clock이나  [RTC](https://en.wikipedia.org/wiki/Real-time_clock)입니다. 우리는 리눅스 커널안에 있는[drivers/rtc](https://github.com/torvalds/linux/tree/master/drivers/rtc) 디렉토리에서 독립적인 아키텍쳐인 real time clock 드라이버를 찾을수 있습니다. 예를 들어 `CMOS/RTC` 는[x86](https://en.wikipedia.org/wiki/X86) architecture. The second is system timer - timer that excites [interrupts](https://en.wikipedia.org/wiki/Interrupt)의 아키텍쳐인 겨우 [arch/x86/kernel/rtc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/rtc.c) 입니다. 두번째는 시스템 타이머입니다. 주기적인 속도로 인터럽트를 자극하는 타이머이죠. 예를 들어 [IBM PC](https://en.wikipedia.org/wiki/IBM_Personal_Computer) 호환의 경우에는 - programmable intercal time입니다.

우리는 이미 시간을 지키기 위해 리눅스 커널에서 `jiffies`를 사용할 수 있다는 것을 알고 있습니다. `jiffies`는 `HZ`로 빈번하게 업데이트되는 읽기 전용 글로벌 변수로 간주 될 수 있습니다. 우리는 `HZ`가 합당한 범위가 `100`에서 `1000`Hz 인 컴파일-타임 커널 매개 변수라는 것을 알고 있습니다. 따라서 `1 ~ `10` 밀리 초로 시간 측정을 위한 인터페이스가 보장됩니다. 
표준 `jiffies` 외에도, 우리는 거의 '1193182' 헤르츠 인 `i8253 / i8254`[programmable interval timer](https://en.wikipedia.org/wiki/Programmable_interval_timer) 또한  틱 속도를 기반으로하는 이전 부분의 `refined_jiffies` 클록 소스를 확인했습니다. 따라서 'refined_jiffies'로 약 '1' 마이크로 초의 해상도를 얻을 수 있습니다. 이 때, 주어진 클럭 소스의 시간 값 단위로 [nanoseconds](https://en.wikipedia.org/wiki/Nanosecond)가 가장 선호되는 선택입니다. 

시간 간격의 측정을 위한 더 정확한 기술의 가용성은 하드웨어에 의존하는 것입니다. 우리는 `x86` 의존적인 타이머 하드웨어에 대해서 알게 되었습니다. 하지만 각각의 아키텍쳐는 각각의 하드웨어적인 타이머를 제공합니다. 일찍이 각 아키텍쳐는 이 목적을 위한 자체적인 구현을 할 수 있습니다. 이 문제에 대한 해결법은 다양한 클럭소스를 관리하고 타이머 인터럽트와는 독립된 공통 프레임워크 안에 있는 추상화 레이어와 연합된 API입니다. 이 평범한 코드 프레임워크는 `clocksource` 프레임워크가 되었습니다.

일반적인 시간 및 클럭 소스 관리 프레임 워크는 많은 시간 관리 코드를 독립적인 아키텍쳐의 부분으로 옮겼습니다. 의존적인 아키텍쳐 부분은 저수준의 하드웨어 클럭 소스 정의 및 관리를 줄여줍니다. 다른 아키텍쳐나 다른 하드웨어에서 시간 간격을 측정하려면 많은 재원이 필요하고 또한 이것은 매우 복잡합니다. 각 시계 관련 서비스의 구현은 개별 하드웨어 장치와 밀접한 관련이 있습니다. 그리고 이것은 다른 아키텍쳐에서도 비슷한 구현이 가능합니다.

이 프레임워크에서 각각의 클럭 소스는 단조롭게 증가하는 값의 시간에 대한 표현을 유지하는 것을 필요로 합니다. 우리가 리눅스 커널 코드에서 볼수 있듯이 나노초는 클럭 소스에서 요즘 가장 잘쓰이는 시간 값중에서 가장 마음에 드는 선택입니다. 클럭 소스 프레임워크의 중요한 점은 사용자가 시스템을 구성하고 다른 클록 소스를 선택, 액세스 및 스케일링 할 때 클럭 기능을 지원하는 다양한 하드웨어 장치 중에서 클럭소스를 선택할 수 있도록 하는 것입니다. 

클럭소스 구조
--------------------------------------------------------------------------------

`clocksource` 프레임워크의 기본적인 것은 `clocksource` 구조인데 그것은 [include/linux/clocksource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h) 헤더파일에 정의되어있습니다. 우리는 이전 부분[part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html).에서 `clocksource`  구조에 의해 제공된 일부 필드를 이미 보았습니다. 이 구조의 전체적인 정의를 살펴보고 모든 필드에 대해 설명해봅시다.

```C
struct clocksource {
	cycle_t (*read)(struct clocksource *cs);
	cycle_t mask;
	u32 mult;
	u32 shift;
	u64 max_idle_ns;
	u32 maxadj;
#ifdef CONFIG_ARCH_CLOCKSOURCE_DATA
	struct arch_clocksource_data archdata;
#endif
	u64 max_cycles;
	const char *name;
	struct list_head list;
	int rating;
	int (*enable)(struct clocksource *cs);
	void (*disable)(struct clocksource *cs);
	unsigned long flags;
	void (*suspend)(struct clocksource *cs);
	void (*resume)(struct clocksource *cs);
#ifdef CONFIG_CLOCKSOURCE_WATCHDOG
	struct list_head wd_list;
	cycle_t cs_last;
	cycle_t wd_last;
#endif
	struct module *owner;
} ____cacheline_aligned;
```


우리는 이미 이전 파트에서 `clocksource` 구조의 첫부분을 보았습니다. 그것은 클럭소스 프레임워크에서 선택한 최고의 카운터를 반환하는 `read` 함수에 대한 포인터입니다. 예를 들어 `jiffies_read` 함수를 이용해 `jiffies` 를 읽는 것과 같습니다.

```C
static struct clocksource clocksource_jiffies = {
	...
	.read		= jiffies_read,
	...
}
```

`jiffies_read` 가 반환됨:

```C
static cycle_t jiffies_read(struct clocksource *cs)
{
	return (cycle_t) jiffies;
}
```
혹은 `read_tsc` 함수:

```C
static struct clocksource clocksource_tsc = {
	...
    .read                   = read_tsc,
	...
};
```

타임 스탬프[time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter) 카운터를 읽기 위함이다.

다음 필드는 `64 bit` 카운터가 아닌 카운터에서 카운터 값 사이의 감산 시 특별한 오버플로 로직이 필요하지 않도록 하는 `mask`입니다. `mask` 필드 이후에 우리는 두가지 필드를 볼수 있습니다: `mult` 와 `shift`. 이것들은 수학적인 함수에 기반을 한 필드들입니다. 그리고 그것들은 각 클럭 소스에 특정한 시간 값을 변환하는 기능을 제공합니다. 즉 이 두 필드는 우리가 카운터의 추상적인 기계 시간 단위를 나노초로 변환하는 데 도움을 준다.

이 두 필드 후에 우리는 `64`bits `max_idle_ns` 필드가 클럭 소스가 허용하는 max_idle_time을 나노초 단위로 나타낸다는 것을 알 수 있다. `CONFIG_NO_HZ` 커널 구성 옵션이 활성화된 Linux 커널에 대해서는 이 필드가 필요하다. 이 커널 구성 옵션을 사용하면 리눅스 커널이 일반 타이머 확인 없이 실행될 수 있다(다른 부분에서 자세한 설명을 볼 수 있다). 동적 눈금이 커널을 단일 눈금보다 더 긴 시간 동안 재울 수 있다는 문제는 더 이상 수면 시간이 무제한일 수 있다는 것이다. `max_idle_ns` 필드는 이 절전 수면 제한을 나타낸다.

`max_idle_ns` 다음 필드는 `maxadj` 필드로 `mult`에 대한 최대 조정 값이다. 사이크을 나노초로 변환하는 주요공식이 여기있다:

```C
((u64) cycles * mult) >> shift;
```
이것은 `100%` 정확하진 않습니다. 대신에 숫자는 가능한한 1 나노초에 가까이 나타내고 `maxadj`는 이것을 수정하는 것을 도와주고 또한 클럭소스 API가 조정될때 오버플로가 발생할 수 있는 `mult` 값을 피할 수 있도록 해줍니다. 다음 4가지 필드는 함수에 대한 포인터입니다.

* `enable` - 클럭 소스를 활성화하는 선택적 기능;
* `disable` - 클럭 소스를 비활성화하는 선택적 기능;
* `suspend` - 클럭 소스에 대한 일시 중단 기능;
* `resume` - 클럭 소스에 대한 재개;

다음 필드는 `max_cycle`이며, 이름에서 알 수 있듯이 이 필드는 잠재적 오버플로 전의 최대 사이클 값을 나타낸다. 그리고 마지막 필드는 소유자가 클럭 소스의 소유자인 커널 [module](https://en.wikipedia.org/wiki/Loadable_kernel_module)에 대한 참조를 나타낸다. 이게 다야. 우리는 방금 `clocksource` 구조의 모든 표준 필드를 살펴보았다. 하지만 당신은 우리가  `clocksource` 구조의 몇몇 분야를 놓쳤다는 것을 알 수 있다. 누락된 필드는 다음 두 가지 유형으로 나눌 수 있다. 첫 번째 유형의 분야는 이미 우리에게 알려져 있다. 예를 들어, 그것들은 `clocksource`의 이름을 나타내는 `name` 필드, 리눅스 커널이 최고의  `clocksource` 등을 선택하도록 돕는 `rating` 필드 등이다. 다른 리눅스 커널 구성 옵션에 종속된 두 번째 유형. 이 필드를 봅시다.

첫 번째 필드는 `archdata`다. 이 필드에는 `arch_clocksource_data` 유형이 있으며 `CONFIG_ARCH_CLOCKSOURCE_DATA` 커널 구성 옵션에 따라 달라집니다. 이 분야는 현재 [x86](https://en.wikipedia.org/wiki/X86) 및 [IA64](https://en.wikipedia.org/wiki/IA-64) 아키텍처에 대해서만 실제적입니다. 그리고 다시, 우리가 필드의 이름에서 이해할 수 있듯이, 그것은 클럭소스에 대한 아키텍처별 데이터를 나타냅니다. 예를 들어 `vDSO` 클럭 모드를 나타낸다.

```C
struct arch_clocksource_data {
    int vclock_mode;
};
```
 
`x86` 아키텍쳐의 경우 `vDOS` 클럭 모드가 다음 중 하나가 될 수 있는 경우:

```C
#define VCLOCK_NONE 0
#define VCLOCK_TSC  1
#define VCLOCK_HPET 2
#define VCLOCK_PVCLOCK 3
```

마지막 3개 필드는 `wd_list`, `cs_last`, `wd_last`인데 `CONFIG_CLOCKSOURCE_WATCHDOG` 커널 구성 옵션에 따라 달라진다. 우선 `watchdog` 가 뭔지 이해하도록 노력하도록 하자. 간단히 말해서 `watchdog`는 컴퓨터 오작동을 감지하고 그것을 복구하는 데 사용되는 타이머다. 이 세 분야 모두 클럭소스 프레임워크에서 사용하는 감시 관련 데이터를 포함하고 있습니다. Linux 커널 소스 코드를 grep(?)할 경우, [arch/x86/KConfig](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Kconfig#L54) 커널 구성 파일에만 `CONFIG_CLOCKSOURCE_WATCHDOG` 커널 구성 옵션이 포함되어 있음을 확인할 수 있습니다. 그렇다면 `x86`과 `x86_64`는 왜 `watchdog`가 필요할까? 모든 x86 프로세서에는 특수 64비트 레지스터 - 타임스탬프 카운터가 있다는 것을 이미 알고 있을 것입니다. 이 레지스터는 재설정 후 [cycles](https://en.wikipedia.org/wiki/Clock_rate)를 포함합니다. 때때로 [time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)는 다른 클럭 소스에 대해 검증되어야 합니다. 이 파트에서 `watchdog` 타이머의 초기화를 볼 수 없을 것이며, 그 전에 타이머에 대해 더 자세히 알아봐야 합니다.
그게 다입니다. 이시간을 기해서 우리는 클럭소스 구조의 모든 필드에 대해서 알게 되었습니다. 이 지식은 클럭소스 프레임워크의 내부를 배울때 도움을 줄것입니다.

그게 다입니다. 이시간을 기해서 우리는 `clocksource` 구조의 모든 필드에 대해서 알게 되었습니다. 이 지식은 `clocksource` 프레임워크의 내부를 배울때 도움을 줄것입니다.

새 클럭소스 등록
--------------------------------------------------------------------------------

우리는 이전  [part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html)에서 클럭소스 프레임 워크의 오직 하나의 기능만을 보았습니다.
이 기능은 `__clocksource_register`입니다. 이 기능은 [include/linux/clocksource.h](https://github.com/torvalds/linux/tree/master/include/linux/clocksource.h)에 정의되어 있습니다. 헤더 파일 그리고 우리가 기능의 이름으로부터 이해할 수 있듯이, 이 기능의 주요 점은 새로운 클럭 소스를 등록하는 것입니다. `__clocksource_register` 기능의 구현을 살펴보면, `__clocksource_register_scale` 함수를 호출하기만 하면 그 결과를 다음과 같이 반환하는 것을 알 수 있을 것입니다:

```C
static inline int __clocksource_register(struct clocksource *cs)
{
	return __clocksource_register_scale(cs, 1, 0);
}
```
우리가 `__clocksource_register_scale`함수의 기능 구현을 보기 전에 우리는 `clocksource`가 새로운 클럭소스 등록을 위한 추가 API를 제공하는 것을 볼 수 있습니다.

```C
static inline int clocksource_register_hz(struct clocksource *cs, u32 hz)
{
        return __clocksource_register_scale(cs, 1, hz);
}

static inline int clocksource_register_khz(struct clocksource *cs, u32 khz)
{
        return __clocksource_register_scale(cs, 1000, khz);
}
```
그리고 모든 함수들은 같은 일을 합니다. 그들은 `__clocksource_register_scale` 함수의 값을 돌려줍니다. 하지만 다른 매개변수 세트를 가지고 있습니다. `__clocksource_register_scale` 함수는 [kernel/time/clocksource.c](https://github.com/torvalds/linux/tree/master/kernel/time/clocksource.c) 소스코드 파일에 정의 되어 있습니다. 두가지 함수의 차이점을 이해하려면  `__clocksource_register_khz` 함수의 매개변수를 살펴보자. 우리가 볼 수 있듯이 이 함수는 세개의 매개변수를 갖습니다.

* `cs` - 클럭소스가 설치되게 해준다.;
* `scale` - 클럭 소스의 축척 계수 즉, 주파수에 대해 이 매개변수의 값을 곱하면 클럭 소스의 `hz`를 얻을 수 있다;
* `freq` - 클럭소스 주파수를 눈금으로 나눕니다.;

이제 `__clocksource_register_scale` 함수의 구현에 대해 살펴보자.

```C
int __clocksource_register_scale(struct clocksource *cs, u32 scale, u32 freq)
{
        __clocksource_update_freq_scale(cs, scale, freq);
        mutex_lock(&clocksource_mutex);
        clocksource_enqueue(cs);
        clocksource_enqueue_watchdog(cs);
        clocksource_select();
        mutex_unlock(&clocksource_mutex);
        return 0;
}
```

우선 `__clock_register_scale` 함수는 동일한 소스 코드 파일에 정의되어 있는 `__clock_update_frq_scale` 함수의 호출에서 시작하여 주어진 클럭 소스를 새로운 주파수로 업데이트하는 것을 알 수 있다. 이 기능의 구현에 대해 살펴보자. 첫 번째 단계에서는 주어진 주파수를 확인하고 `zero`으로 통과되지 않은 경우 주어진 클럭 소스에 대한 멀티 및 시프트 파라미터를 계산해야 합니다. 주파수의 값을 확인해야 하는 이유는? 사실 `zero`이 될 수 있습니다. `__clocksource_register` 기능의 구현을 주의깊게 살펴본다면, 우리가 주파수를 0으로 넘겼다는 것을 알아차렸을지도 모릅니다. 자체 정의 된 `mult` 및 `shift` 매개 변수가있는 일부 클럭 소스에 대해서만 이를 수행합니다. previous [part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html)을 살펴보면 `jiffies` 에 대한 `mult`와 `shift` 계산을 볼 수 있습니다. `mult` 와 `shift` 계산을 봐보죠:

```C
void __clocksource_update_freq_scale(struct clocksource *cs, u32 scale, u32 freq)
{
        u64 sec;

		if (freq) {
             sec = cs->mask;
             do_div(sec, freq);
             do_div(sec, scale);

             if (!sec)
                   sec = 1;
             else if (sec > 600 && cs->mask > UINT_MAX)
                   sec = 600;
 
             clocks_calc_mult_shift(&cs->mult, &cs->shift, freq,
                                    NSEC_PER_SEC / scale, sec * scale);
	    }
	    ...
        ...
        ...
}
```

여기서는 클럭 소스 카운터가 오버플로우되기 전에 실행할 수 있는 최대 시간(초)의 계산을 볼 수 있습니다. 우선, 우리는 클럭 소스 마스크의 값으로 `sec` 변수를 채웁니다. 클럭 소스의 마스크는 주어진 클럭 소스에 유효한 최대 비트 양을 나타냅니다. 이 일이 끝나면, 우리는 두 개로 나뉜 연산을 볼수 있습니다. 처음에 우리는 클럭 소스 주파수와 스케일 팩터로 `sec` 변수를 나눕니다.  `freq` 매개변수는 1초 안에 얼마나 많은 타이머 인터럽트가 발생할지를 보여줍니다. 그래서 우리는 타이머의 주파수에서 카운터의 최대 수(예: `jiffy`)를 나타내는 `mask` 값을 나누며, 특정 클럭 소스의 최대 수(초)를 얻을 것입니다. 2분할 작동은 1헤르츠 또는 1킬로헤르츠(10^3 Hz)의 스케일 인자에 따라 특정 클럭 소스에 대해 최대 초를 제공합니다. 
최대 시간(초)을 얻은 후 이 값을 확인하여 다음 단계에서 결과에 따라 `1` 또는 `600`으로 설정합니다. 이 값은 클럭 소스의 최대 절전 시간(초)이다. 다음 단계에서 우리는 `clocks_calc_mult_shift`의 호출을 볼 수 있습니다. 이 함수의 핵심은 주어진 클럭 소스에 대한 `mult` 및  값 계산입니다. `__clocksource_update_frq_scale` 함수의 끝에서 우리는 주어진 클럭 소스의 계산된 멀티 값만 조정 후 오버플로를 일으키지 않는지 확인하고, 주어진 클럭 소스의 `max_idle_ns` 및 `max_cycle` 값을 클럭 소스 카운터로 변환할 수 있는 최대 나노초로 업데이트하고 결과를 커널 버퍼로 인쇄합니다:

```C
pr_info("%s: mask: 0x%llx max_cycles: 0x%llx, max_idle_ns: %lld ns\n",
	cs->name, cs->mask, cs->max_cycles, cs->max_idle_ns);
```

that we can see in the [dmesg](https://en.wikipedia.org/wiki/Dmesg) output:

```
$ dmesg | grep "clocksource:"
[    0.000000] clocksource: refined-jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1910969940391419 ns
[    0.000000] clocksource: hpet: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 133484882848 ns
[    0.094084] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1911260446275000 ns
[    0.205302] clocksource: acpi_pm: mask: 0xffffff max_cycles: 0xffffff, max_idle_ns: 2085701024 ns
[    1.452979] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x7350b459580, max_idle_ns: 881591204237 ns
```

`__clocksource_update_frq_scale` 함수가 작업을 마치면 새로운 클럭 소스를 등록할 `__clocksource_register_scale` 함수로 돌아갈 수 있다. 우리는 다음 세 가지 기능의 호출을 볼 수 있다.

```C
mutex_lock(&clocksource_mutex);
clocksource_enqueue(cs);
clocksource_enqueue_watchdog(cs);
clocksource_select();
mutex_unlock(&clocksource_mutex);
```

첫 번째가 호출되기 전에 `clocksource_mutex` [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)를 잠근다는 점에 유의하십시오. `clocksource_mutex` 뮤텍스의 포인트는 현재 선택된 클럭소스 및 등록된 클럭소스가 포함된 목록을 나타내는 clocksource_list 변수를 나타내는 `curr_clocksource` 변수를 보호하는 것이다. 자, 이제 이 세 가지 기능을 살펴봅시다.
첫 번째 `clocksource_enqueue` 함수 및 다른 두 개는 동일한 소스 코드 파일[file](https://github.com/torvalds/linux/tree/master/kernel/time/clocksource.c)에 정의되어 있다. 우리는 이미 등록된 모든 클럭 소스와 다른 말로 하면 `clocksource_list`의 모든 요소를 살펴보고 주어진 `clocksource`에 가장 적합한 장소를 찾으려고 노력한다.

```C
static void clocksource_enqueue(struct clocksource *cs)
{
	struct list_head *entry = &clocksource_list;
	struct clocksource *tmp;

	list_for_each_entry(tmp, &clocksource_list, list)
		if (tmp->rating >= cs->rating)
			entry = &tmp->list;
	list_add(&cs->list, entry);
}
```

결국 우리는 새로운 클럭 소스를 `clocksource_list`에 삽입할 뿐입니다. 두 번째 기능인 `clocksource_enqueue_watchdog`는 이전 기능과 거의 동일하지만 `wd_list`에 새 클럭 소스를 삽입하는 것은 클럭 소스의 플래그에 따라 달라지며 새로운  [watchdog](https://en.wikipedia.org/wiki/Watchdog_timer) 타이머를 시작한다. 내가 이미 썼듯이, 우리는 이 부분에서 `watchdog`와 관련된 것들을 고려하지 않고 다음 부분에서 할 것입니다.

마지막 함수는 `cocksource_select`입니다. 기능 이름에서 알 수 있듯이, 이 기능의 주요 지점 - 등록된 클럭 소스에서 최고의 `clocksource`를 선택하십시오. 이 기능은 기능 도우미의 호출로만 구성됩니다. 


```C
static void clocksource_select(void)
{
	return __clocksource_select(false);
}
```
`__clocksource_select` 함수는 하나의 파라미터(우리의 경우 `false`)를 취한다는 점에 유의하세요. 이 [bool](https://en.wikipedia.org/wiki/Boolean_data_type) 매개변수는 `cocksource_list`를 횡단하는 방법을 보여줍니다. 우리의 경우, 우리는 locksource_list의 모든 항목을 검토한다는 것을 의미하는 `false`를 통과합니다. 우리는 이미 `clocksource_enqueue` 함수의 호출 이후에 `clocksource_list`에서 최고의 등급을 가진 클럭소스가 첫 번째가 될 것이라는 것을 알고 . 그래서 우리는 이 목록에서 그것을 쉽게 얻을 수 있습니다. 최고 등급의 시계 소스를 찾은 후 다음으로 전환합니다:

```C
if (curr_clocksource != best && !timekeeping_notify(best)) {
	pr_info("Switched to clocksource %s\n", best->name);
	curr_clocksource = best;
}
```

이 작업의 결과는 `dmesg` 출력에서 확인할 수 있습니다.

```
$ dmesg | grep Switched
[    0.199688] clocksource: Switched to clocksource hpet
[    2.452966] clocksource: Switched to clocksource tsc
```

`dmesg` 출력(우리의 경우 `hpet`과 `tsc`)에서 두 개의 클럭 소스를 볼 수 있다는 점에 주의하십시오. 사실 특정 하드웨어에 많은 다른 시계 소스가 있을 수 있습니다. 그래서 리눅스 커널은 등록된 모든 클럭 소스에 대해 알고 있으며, 새로운 클럭 소스의 등록 후 매번 더 좋은 등급을 가진 클럭 소스로 전환합니다.

[kernel/time/clocksource.c](https://github.com/torvalds/linux/tree/master/kernel/time/clocksource.c) 소스 코드 파일의 하단을 살펴보면 [sysfs](https://en.wikipedia.org/wiki/Sysfs) 인터페이스가 있음을 알 수 있습니다. 기본 초기화는 장치 초기화 통화 중에 호출되는 `init_clocksource_sysfs` 기능에서 발생합니다. `init_clocksource_sysfs` 기능의 구현에 대해 알아봅시다.

```C
static struct bus_type clocksource_subsys = {
	.name = "clocksource",
	.dev_name = "clocksource",
};

static int __init init_clocksource_sysfs(void)
{
	int error = subsys_system_register(&clocksource_subsys, NULL);

	if (!error)
		error = device_register(&device_clocksource);
	if (!error)
		error = device_create_file(
				&device_clocksource,
				&dev_attr_current_clocksource);
	if (!error)
		error = device_create_file(&device_clocksource,
					   &dev_attr_unbind_clocksource);
	if (!error)
		error = device_create_file(
				&device_clocksource,
				&dev_attr_available_clocksource);
	return error;
}
device_initcall(init_clocksource_sysfs);
```

우선 `subys_system_register` 함수의 호출로 클럭소스 하위시스템을 등록한다는 것을 알 수 있습니다. 즉, 이 기능의 호출 후, 다음과 같은 디렉토리를 갖게 됩니다.

```
$ pwd
/sys/devices/system/clocksource
```

이 단계 이후, 우리는 다음과 같은 구조로 대표되는 `device_clocksource` 장치의 등록을 볼 수 있습니다.:

```C
static struct device device_clocksource = {
	.id	= 0,
	.bus	= &clocksource_subsys,
};
```

세 개의 파일 생성:

* `dev_attr_current_clocksource`;
* `dev_attr_unbind_clocksource`;
* `dev_attr_available_clocksource`.

이러한 파일은 시스템의 현재 클럭 소스, 시스템에서 사용 가능한 클럭 소스, 클럭 소스의 바인딩을 해제할 수 있는 인터페이스에 대한 정보를 제공합니다.

`init_clocksource_sysfs` 기능이 실행된 후 다음에서 사용 가능한 클럭 소스에 대한 몇 가지 정보를 찾을 수 있을 것입니다:

```
$ cat /sys/devices/system/clocksource/clocksource0/available_clocksource 
tsc hpet acpi_pm 
```

혹은 예를 들어 시스템의 현재 클럭 소스에 대한 정보:

```
$ cat /sys/devices/system/clocksource/clocksource0/current_clocksource 
tsc
```

앞부분에서는 `jiffies` 시계소스의 등록을 위한  API를 보았지만, `clocksource` 프레임워크에 관한 세부사항에 대해서는 깊이 파고들지 않았습니다. 이 파트에서 우리는 그것을 했고 새로운 클럭소스 등록과 시스템에서 가장 좋은 등급 값을 가진 클럭소스의 선택을 구현하는 것을 보았습니다. 물론, 클럭소스 프레임워크가 제공하는 API만 있는 것은 아닙니다. `clocksource_list` 등에서 주어진 클럭소스를 제거하기 위해 `clocksource_unregister`와 같은 몇 가지 추가 기능이 있습니다. 그러나 나는 이 기능들을 지금 우리에게 중요하지 않기 때문에 이 부분에 대해서는 설명하지 않을 것입니다. 어쨌든 당신이 그것에 흥미가 있다면, 당신은 그것을 [kernel/time/clocksource.c](https://github.com/torvalds/linux/tree/master/kernel/time/clocksource.c).에서 찾을 수 있습니다.

이것이 전부입니다~

결론
--------------------------------------------------------------------------------

리눅스 커널의 타이머와 타이머 관리 관련 내용을 기술한 챕터의 2부 끝부분입니다. 앞부분에서 `jiffies`와 `clocksource`라는 두 가지 개념을 알게 되었습니다. 이 파트에서 우리는 `jiffies` 사용의 몇 가지 예를 보았고 `clocksource` 개념에 대해 더 자세히 알았습니다.

질문이나 제안 사항이 있으면 언제든지 twitter [0xAX](https://twitter.com/0xAX)로 전화를 걸거나 [email](anotherworldofworld@gmail.com)을 보내거나 저에게 [issue](https://github.com/0xAX/linux-insides/issues/new) 걸어주세요. 

**영어는 내 모국어가 아니라는 것을 명심해주세요. 그리고 어떤 불편함 있었다면 죄송합니다. 만약 당신이 실수를 발견한다면, 나에게[linux-insides](https://github.com/0xAX/linux-insides)로 PR을 보내주십시오.**

Links
-------------------------------------------------------------------------------

* [x86](https://en.wikipedia.org/wiki/X86)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [uptime](https://en.wikipedia.org/wiki/Uptime)
* [Ensoniq Soundscape Elite](https://en.wikipedia.org/wiki/Ensoniq_Soundscape_Elite)
* [RTC](https://en.wikipedia.org/wiki/Real-time_clock)
* [interrupts](https://en.wikipedia.org/wiki/Interrupt)
* [IBM PC](https://en.wikipedia.org/wiki/IBM_Personal_Computer)
* [programmable interval timer](https://en.wikipedia.org/wiki/Programmable_interval_timer)
* [Hz](https://en.wikipedia.org/wiki/Hertz)
* [nanoseconds](https://en.wikipedia.org/wiki/Nanosecond)
* [dmesg](https://en.wikipedia.org/wiki/Dmesg)
* [time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [loadable kernel module](https://en.wikipedia.org/wiki/Loadable_kernel_module)
* [IA64](https://en.wikipedia.org/wiki/IA-64)
* [watchdog](https://en.wikipedia.org/wiki/Watchdog_timer)
* [clock rate](https://en.wikipedia.org/wiki/Clock_rate)
* [mutex](https://en.wikipedia.org/wiki/Mutual_exclusion)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [previous part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-1.html)
