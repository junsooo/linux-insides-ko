리눅스 커널의 타이머와 시간관리 Part 1.
================================================================================

소개글
--------------------------------------------------------------------------------

이 글은  [linux-insides](https://0xax.gitbooks.io/linux-insides/content/) 책에서 새로운 장을 여는 글입니다. 이전의  [part](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-4.html) 는 [system call](https://en.wikipedia.org/wiki/System_call) 의 개념을 설명했으며, 이제 새로운 장을 시작할 차례입니다. 제목에서도 알 수 있듯이 , 이 장에서  리눅스 커널에서 `타이머` 와 `시간관리` 에 대해 다룰 것입니다. 현재 장에 대한 주제 선택은 우연이 아닙니다. 타이머 (및 일반적으로 시간관리)는 리눅스 커널에서 매우 중요하며 널리 사용됩니다. 리눅스 커널은  [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) 구현에서 다른 시간 초과와 같은 다양한 작업에 타이머를 사용합니다, 커널은 현재 시간, 비동기 함수 스케줄링, 다음 이벤트 인터럽트 스케줄링등 다양한 작업에 타이멀르 사용합니다.

그래서 우리는 여기서 다른 시간관리와 관련된 구현을 배울것 입니다. 우리는 다른유형의 타이머를 살펴보고 리눅스 커널의 하위시스템이 사용하는 방식이 어떻게 다른지 살펴 보겠습니다. 언제나 그렇듯이, 리눅스 커널의 가장 앞 부분부터 시작하여 리눅스커널의 초기화 과정을 거치게 됩니다.  우리는 이미 [chapter](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) 에서 리눅스 커널의 초기화 과정을 설명하였습니다, 기억이 나겠지만 우리는 chapter에서 몇가지를 넘겼습니다. 그 중 하나는 타이머 초기화 입니다.

시작해봅시다.

비표준 pc 하드웨어 시계 초기화 
--------------------------------------------------------------------------------

리눅스 커널이 압축 해제 된 후 (더 자세한 내용은 [Kernel decompression](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html) 의 압축 해제 부분을 참조) 아키텍쳐 비 특정 코드가 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일에서 작동하기 시작합니다. [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt),을 초기화 해주고 [cgroups](https://en.wikipedia.org/wiki/Cgroups) 을 초기화 및 [canary](https://en.wikipedia.org/wiki/Buffer_overflow_protection) 값을 설정후  `setup_arch` 함수의 호출을 볼 수 있습니다.

기억하겠지만, 이 함수 (defined in the [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L842))는 아키텍쳐 관련 항목을 준비/초기화 합니다. (예: [bss](https://en.wikipedia.org/wiki/.bss)섹션을 위한 공간을 예약하고 [initrd](https://en.wikipedia.org/wiki/Initrd),를 위한 장소를 예약하고 커널 명령 줄의 구문을 분석외에 많은 것들을 합니다.). 이 외에도 시간 관리 관련 기능들이 있습니다.

첫번째는:

```C
x86_init.timers.wallclock_init();
```

우리는 리눅스 커널의 초기화를 설명하는 장에서 `x86_init`구조를 보았습니다. 이 구조에는 [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms), [Intel CE4100](http://www.wpgholdings.com/epaper/US/newsRelease_20091215/255874.pdf),등과 같은 다양한 플랫폼의 기본 설정 기능에 대한 포인터가 포함되어 있습니다. `x86_init`구조는 [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c#L36),에 정의 되어있으며 기본적으로 표준 PC하드웨어를 결정합니다.

보시다 시피 `x86_init`구조에는 표준 자원 예약, 플랫폼 특정 메모리 설정, 인터럽트 핸들러 초기화 등과 같은 플랫폼 특정 설정을 위한 기능 세트를 제공하는 `x86_init_ops`유형이 있습니다. 이 구조는 다음과 같습니다:

```C
struct x86_init_ops {
	struct x86_init_resources       resources;
    struct x86_init_mpparse         mpparse;
    struct x86_init_irqs            irqs;
    struct x86_init_oem             oem;
    struct x86_init_paging          paging;
    struct x86_init_timers          timers;
    struct x86_init_iommu           iommu;
    struct x86_init_pci             pci;
};
```

유형이 `timers`이 있는 필드를 적어 `x86_init_timers` 저장합니다. 이 필드는 시간 관리 및 타이머와 관련이 있음을 그 이름으로 이해할 수 있습니다. `x86_init_timers` 에는 [void](https://en.wikipedia.org/wiki/Void_type)에 대한 포인터를 반환하는 모든 함수 4개의 필드가 있습니다.:

* `setup_percpu_clockev` - 부팅 cpu에 cpu당 이벤트 장치 설정;
* `tsc_pre_init` - [TSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter) 이전에 호출 된 플랫폼 기능;
* `timer_init` - 플랫폼 타이머 초기화;
* `wallclock_init` - 벽시계 장치 초기화.

우리가 이미 알고 있듯이, 이 경우에는  `wallclock_init`에는 벽시계장치의 초기화를 실행합니다. `x86_init` 구조를 보면, `wallclock_init` 가 `x86_init_noop`을 가르키는 것으로 볼수 있습니다.:

```C
struct x86_init_ops x86_init __initdata = {
	...
	...
	...
	.timers = {
		.wallclock_init		    = x86_init_noop,
	},
	...
	...
	...
}
```

여기서 `x86_init_noop` 은 아무것도 수행하지 않는 함수입니다.:

```C
void __cpuinit x86_init_noop(void) { }
```

표준 PC 하드웨어용, the `wallclock_init` 함수는 [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms) 플랫폼에서 사용됩니다. `x86_init.timers.wallclock_init` 의 초기화는 [arch/x86/platform/intel-mid/intel-mid.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/platform/intel-mid/intel-mid.c) 함수의 `x86_intel_mid_early_setup` 소스 코드 파일에 있습니다.:

```C
void __init x86_intel_mid_early_setup(void)
{
	...
	...
	...
	x86_init.timers.wallclock_init = intel_mid_rtc_init;
	...
	...
	...
}
```

`intel_mid_rtc_init`함수의 구현은 [arch/x86/platform/intel-mid/intel_mid_vrtc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/platform/intel-mid/intel_mid_vrtc.c) 소스코드 파일에 있으며 매우 단순해 보입니다. 우선, 이 함수는 이러한 장치를 [Simple Firmware Interface](https://en.wikipedia.org/wiki/Simple_Firmware_Interface) M-Real-Time-Clock 테이블을  분석하여 `sfi_mrtc_array` 배열로 가져오고  `set_time` 및 `get_time` 함수를 초기화 합니다.:

```C
void __init intel_mid_rtc_init(void)
{
	unsigned long vrtc_paddr;

	sfi_table_parse(SFI_SIG_MRTC, NULL, NULL, sfi_parse_mrtc);

	vrtc_paddr = sfi_mrtc_array[0].phys_addr;
	if (!sfi_mrtc_num || !vrtc_paddr)
		return;

	vrtc_virt_base = (void __iomem *)set_fixmap_offset_nocache(FIX_LNW_VRTC,
								vrtc_paddr);

    x86_platform.get_wallclock = vrtc_get_time;
	x86_platform.set_wallclock = vrtc_set_mmss;
}
```

그 후에는 `Intel MID` 기반 장치가 하드웨어 시계에서 시간을 얻을 수 있습니다. 이미 작성한 것처럼, 표준 PC[x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처는 `x86_init_noop` 을 지원하지 않으며 이 함수를 호출하는 동안 아무것도 하지 않습니다. 방금 [real time clock](https://en.wikipedia.org/wiki/Real-time_clock) 이[Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms) 아키테처에 대해 초기화 되는걸 보았습니다, 이제 일반적인 `x86_64` 아키텍처로 돌아가서 시간 관리 관련 내용을 살펴보겠습니다.

지피 알아보기
--------------------------------------------------------------------------------

`setup_arch` 함수 (which is located, as you remember, in the  [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L842) 소스 코드 파일에 있습니다.)로 돌아가면 시간 관리 관련 함수의 다음 호출이 표시됩니다.:

```C
register_refined_jiffies(CLOCK_TICK_RATE);
```

이 함수의 구현을 보기 전에  [jiffy](https://en.wikipedia.org/wiki/Jiffy_%28time%29). 에 대해 알아야 합니다.  wikipedia에서 관련 내용을 읽을수 있습니다.:

```
Jiffy is an informal term for any unspecified short period of time
```

이 정의는 리눅스 커널의 `jiffy` 와 매우 유사합니다. 시스템 부팅 이후 발생한 틱 수를 보유하는  `jiffies`를 가진 전역변수가 있습니다. 리눅스 커널은 이 변수를 0으로 설정합니다.:

```C
extern unsigned long volatile __jiffy_data jiffies;
```

초기화 과정에서. 이 전역변수는 타이머 인터럽트 동안 매번 증가합니다. 이 외에도 `jiffies` 변수 근처에서 비슷한 변수의 정의를 볼 수 있습니다.

```C
extern u64 jiffies_64;
```

실제로 이러한 변수 중 하나만 리눅스 커널에서 사용되며 프로세서 유형에 따라 다릅니다. [x86_64](https://en.wikipedia.org/wiki/X86-64) 의 경우 `u64` 를 사용하고  [x86](https://en.wikipedia.org/wiki/X86) 의 경우  `unsigned long` 입니다. 우리는 이것을 [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vmlinux.lds.S) 링커 스트립트를 보고 있습니다.:

```
#ifdef CONFIG_X86_32
...
jiffies = jiffies_64;
...
#else
...
jiffies_64 = jiffies;
...
#endif
```

`x86_32`의 경우 `jiffies`는  하위 `32` 비트의 `jiffies_64`의 변수입니다. 계획적으로, 다음과 같이 상상할수 있습니다.

```
                    jiffies_64
+-----------------------------------------------------+
|                       |                             |
|                       |                             |
|                       |       jiffies on `x86_32`   |
|                       |                             |
|                       |                             |
+-----------------------------------------------------+
63                     31                             0
```

이제 우리는 `jiffies` 에 대한 약간의 이론을 알게 되었고 함수의 리턴값을 알 수 있습니다. 함수에 대한 아키텍처 별 구현이 없습니다. -  `register_refined_jiffies`. 이 함수는 일반적인 커널 코드에 있습니다 - [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 소스 코드 파일에 있습니다. `register_refined_jiffies`의 요점은 순간 `clocksource`의 등록입니다. `register_refined_jiffies` 함수의 구현을 살펴 보기 전에 , 우리는 `clocksource`가 뭔지 알아야 합니다. 주석에서 읽을수 있습니다. :

```
The `clocksource` is hardware abstraction for a free-running counter.
```

당신에 대해 알지는 못하지만 `clocksource` 의 설명은 이해하기 좋은 소스코드가 아닙니다.  이게 무엇인지 이해하여 봅시다, 그러나 이 주제는 별도의 부분에서 훨씬 자세하게 설명되므로 따로 더 깊이 들어 가지는 않겠습니다. `clocksource` 의 요점은 타임키핑의 추상화 또는 매우 간단한 단어로 커널에 시간값을 제공합니다. 우리는 이미 시스템 부팅 이후 발생한 틱수를 나타내는 `jiffies` 인터페이스에 대해 알고 있습니다. 리눅스 커널에서 전역변수로 표시되며 각 타이머 인터럽트가 증가합니다. 리눅스 커널은 시간측정에 `jiffies` 를 사용할수 있습니다. 그렇다면 왜 우리는  `clocksource`와 같은 별도의 상황이 필요할까요? 실제로, 하드웨어 장치의 기능에 따라 다른 클럭 소스를 제공합니다. 시간의 간격측정에 대한 보다 정확한 기술의 가용성은 하드웨어에 따라 다릅니다.

예를 들어 `x86`에는 [Time Stamp  Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter) 라고 하는 64 비트 카운터가 내장 되어있으며 주파수는 프로세서의 주파수와 같습니다. 또 다른 예로는 [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)는 최소 `10 MHz` 주파수의 `64-bit` 카운터로 구성됩니다.두개의 다른 타이머와 둘 다  `x86`입니다.다른 아키텍처의 타이머를 추가하면 문제가 더 복잡해 집니다. 리눅스 커널은 이러한 문제를 해결하기 위해 `clocksource` 의 개념을 제공합니다.

`clocksource` 의 개념은 리눅스 커널에서  `clocksource` 의 구조로 표현 됩니다. 이 구조는[include/linux/clocksource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h) 헤다파일에 정의 되어있으며 시간을 설명하는 두개의 필드를 포함합니다. 예를 들어, 카운터의 이름인 `name`필드, 카운터의 다른 속성을 설명하는`flags` 필드, 함수의 포인터를 표현하는 `suspend` 및 `resume` 등을 포함합니다.

[kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 소스에 정의 된 `jiffies`에 대한 `clocksource`구조를 살펴 보겠습니다.
 코드파일:

```C
static struct clocksource clocksource_jiffies = {
	.name		= "jiffies",
	.rating		= 1,
	.read		= jiffies_read,
	.mask		= 0xffffffff,
	.mult		= NSEC_PER_JIFFY << JIFFIES_SHIFT,
	.shift		= JIFFIES_SHIFT,
	.max_cycles	= 10,
};
```

우리는 여기서 기본 이름의 정의인  `jiffies`. 를 볼 수 있습니다.  다음은`rating` 필드로, 지정된 하드웨어에 사용 가능한 시간 관리 소스 코드로 최상의 등록 시간 소스를 선택할수 있습니다.  `rating` 은 다음과 같은 값을 가집니다. :

* `1-99`    - 부팅 및 테스트 목적으로만 사용 가능;
* `100-199` - 실제 사용을 위한 기능이지만 바람직 하지 않음.
* `200-299` - 정확하고 사용가능한 클럭 소스.
* `300-399` - 빠르고 정확한 클럭 소스.
* `400-499` -이상적인 클럭 소스. 가능한 경우 반드시 사용해야합니다.;

예를 들어 [time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)의 등급은 `300`이지만 , [high precision event timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer) 의 등급은 `250`입니다. 다음 필드는 `read`입니다. - 클럭 소스의 사이클 값을 읽을 수 있는 기능에 대한 포인터로, 즉 `cycle_t` 유형의 `jiffies` 변수만 반환한다. :

```C
static cycle_t jiffies_read(struct clocksource *cs)
{
        return (cycle_t) jiffies;
}
```

즉 64-bit 의 양수형 타입:

```C
typedef u64 cycle_t;
```

다음 필드는  `mask` 값으로, `64 bit` 가 아닌 카운터에서 값을빼는 데 특수 오버플로 로직이 필요하지 않습니다.  이 경우의 마스크는 `0xffffffff` 이고  `32`비트 입니다. 이것은  `jiffy` 가  `42`초 후에 0으로 줄어드는 것을 의미합니다:

```python
>>> 0xffffffff
4294967295
# 42 nanoseconds
>>> 42 * pow(10, -9)
4.2000000000000006e-08
# 43 nanoseconds
>>> 43 * pow(10, -9)
4.3e-08
```

다음 두 필드인  `mult`와 `shift` 는 클럭 소스의 주기를 사이클 당 나노 초로 변환 하는데 사용됩니다.커널이  `clocksource.read` 함수를 호출하면 이함수는 `cycle_t` 데이터 형식으로 표시되는  `machine` 타입 단위의 값을 단환합니다.  이 리턴 값을  [nanoseconds](https://en.wikipedia.org/wiki/Nanosecond) 변환하려면 : `mult` 와 `shift` 두 필드가 필요합니다. `clocksource` 는  `clocksource_cyc2ns` 함수를 제공하여 다음과 같은 식으로 우리에게 도움이 됩니다:

```C
((u64) cycles * mult) >> shift;
```

보시다시피 `mult`  필드는 같습니다:

```C
NSEC_PER_JIFFY << JIFFIES_SHIFT

#define NSEC_PER_JIFFY  ((NSEC_PER_SEC+HZ/2)/HZ)
#define NSEC_PER_SEC    1000000000L
```

기본적으로, `shift`는

```C
#if HZ < 34
  #define JIFFIES_SHIFT   6
#elif HZ < 67
  #define JIFFIES_SHIFT   7
#else
  #define JIFFIES_SHIFT   8
#endif
```

`jiffies` 클록 소스는 `NSEC_PER_JIFFY` 승수 변환을 사용하여 나노 초 초과 사이클 비율을 지정합니다.  `JIFFIES_SHIFT` 및 `NSEC_PER_JIFFY` 의 값은 `HZ` 값에 따라 달라집니다.  `HZ` 는 시스템 타이머의 주파수를 나타냅니다. 이 매크로는 [include/asm-generic/param.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic/param.h) 에 정의 되어 있으며 `CONFIG_HZ` 커널 구성 옵션에 따라 다릅니다.  `HZ` 의 가치는 지원되는 아키텍처마다 다르지만 `x86` 의 경우 다음과 같이 정의 됩니다:

```C
#define HZ		CONFIG_HZ
```

여기서 `CONFIG_HZ` 는 다음 값 중 하나 일 수 있습니다:

![HZ](http://i63.tinypic.com/v8d6ph.png)

이것은 우리의 경우에 타이머 인터럽트 주파수가 `250 HZ` 이거나 초당 `250` 번 발생하거나 각`4ms`에 하나의 타이머 인터럽트가 발생한다는 것을 의미합니다.

 `clocksource_jiffies` 구조의 정의에서 볼 수 있는 마지막 필드는 잠재적으로 오버플로를 발생시키지 않고 안전하게 곱할 수 있는 `max_cycles` 값을 보유하는 최대 사이클 값입니다.

그럼,우리는  `clocksource_jiffies` 구조의 정의를 보았습니다. 또 한 `jiffies` 와 `clocksource`, 에 대해 조금 알고 있습니다. 본 파트의 시작 부분에 있어서, 우리는 다음과 같은 호출에 의해 중단됬습니다:

```C
register_refined_jiffies(CLOCK_TICK_RATE);
```

 다음 함수는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L842) 소스 코드 파일에 있습니다.

이미 글 쓴것 처럼 `register_refined_jiffies`함수의 주요 목적은  `refined_jiffies` 클럭 소스를 등록하는 것입니다.  우리는 이미 `clocksource_jiffies` 구조가 표준  `jiffies`클록 소스를 나타내는 것을 보았습니다. [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 에서 소스코드 파일, 또 다른 클럭 소스 정의를 찾을 수 있습니다:

```C
struct clocksource refined_jiffies;
```

`refined_jiffies` 와`clocksource_jiffies`: 사이에는 한가지 차이점이 있습니다. 표준 `jiffies` 기반 클록 소스는 모든 시스템에서 작동해야하는 최저 공통 분모 클록 소스입니다. 우리가 이미 알고 있듯이, `jiffies` 전역 변수는 각 타이머 인터럽트 동안 증가합니다. 이는 표준 `jiffies` 기반 클록 소스가 타이머 인터럽트 주파수와 동일한 해상도를 가짐을 의미합니다.이로부터 우리는 표준 `jiffies` 기반 클럭 소스가 부정확성을 겪을 수 있음을 이해할 수 있습니다.  `refined_jiffies` 는`jiffies`시포트의 기본으로 `CLOCK_TICK_RATE` 를 사용합니다.

이 함수의 구현을 살펴봅시다. 우선, `clocksource_jiffies`구조에 기반한 `refined_jiffies` 클록 소스를 볼 수 있습니다:

```C
int register_refined_jiffies(long cycles_per_second)
{
	u64 nsec_per_tick, shift_hz;
	long cycles_per_tick;

	refined_jiffies = clocksource_jiffies;
	refined_jiffies.name = "refined-jiffies";
	refined_jiffies.rating++;
	...
	...
	...
```

여기서 우리는`refined_jiffies`의 이름을  `refined-jiffies` 로 업데이트 하고 이 구조의 등급을 높이는 것을 볼 수 있습니다. 기억 할 수 있듯이`clocksource_jiffies` 의 등급은 - `1`, 입니다.  따라서`refined_jiffies` 의 클록 소스의 등급은 - `2`입니다. 이는 `refined_jiffies` 가 클록 소스 관리 코드에 가장 적합한 선택임을 의미합니다.

다음 단계에서는 한번의 틱당 사이클 수를 계산해야 합니다:

```C
cycles_per_tick = (cycles_per_second + HZ/2)/HZ;
```

우리는 `NSEC_PER_SEC` 매크로를 표준 `jiffies` 곱함수의 기본으로 사용했습니다. 여기서는 `register_refined_jiffies` 함수의 첫 번째 매개 변수 인`cycles_per_second` 를 사용하고 있습니다. 우리는 `CLOCK_TICK_RATE` 매크로를`register_refined_jiffies`함수에 전달했습니다. 이 매크로는[arch/x86/include/asm/timex.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/timex.h) 헤더 파일에 정의되어 있으며 다음으로 확장됩니다:

```C
#define CLOCK_TICK_RATE         PIT_TICK_RATE
```

여기서  `PIT_TICK_RATE` 매크로는  [Intel 8253](Programmable interval timer)의 주파수로 확장됩니다:

```C
#define PIT_TICK_RATE 1193182ul
```

이후 `hz << 8`을 저장할 `register_refined_jiffies`에 대한   `shift_hz`또는 시스템 타이머의 빈도를 계산한다. 정확성을 높이기 위해`cycles_per_second` 또는 프로그램 가능 간격 타이머의 빈도를 `8` 로 변경하여 추가 정확도를 확보합니다:

```C
shift_hz = (u64)cycles_per_second << 8;
shift_hz += cycles_per_tick/2;
do_div(shift_hz, cycles_per_tick);
```

다음 단계에서는 `shift_hz`에서와 마찬가지로  `NSEC_PER_SEC` 를 `8` 로 이동하여 1개 눈금당 초를 계산하고 이전과 동일한 계산을 수행한다:

```C
nsec_per_tick = (u64)NSEC_PER_SEC << 8;
nsec_per_tick += (u32)shift_hz/2;
do_div(nsec_per_tick, (u32)shift_hz);
```

```C
refined_jiffies.mult = ((u32)nsec_per_tick) << JIFFIES_SHIFT;
```

`register_refined_jiffies` 함수의 끝부분에서 우리는   [include/linux/clocksource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h)에 정의된 `__clocksource_register`함수에 새로운 클록소스 헤더에 를 정의한다 후 리턴한다:

```C
__clocksource_register(&refined_jiffies);
return 0;
```

클록 소스 관리 코드는 클록 소스 등록 및 선택을 위한 API를 제공합니다. 보다시피, 클록 초기화는 커널 초기화 중 또는 커널 모듈에서  `__clocksource_register`함수를 호출하여 등록됩니다. 등록하는 동안 클록 소스관리 코드는 `jiffies`에 대한 `clocksource` 구조를 초기화 할 때 이미 본 `clocksource.rating` 필드를 사용하여 시스템에서 사용 가능한 최상의 클록 소스를 선택합니다.

jiffies 사용하기
--------------------------------------------------------------------------------

이전 단락에서 두 개의`jiffies`기반 클록 소스의 초기화를 보았습니다:

* 표준`jiffies` 기반 클록 소스;
* 세련된  `jiffies` 기반 클록 소스;

여기에서 계산을 이해하지 못해도 걱정할 필요가 없습니다. 처음에는 무섭게 보입니다. 곧 우리는 단계적으로 이러한 것들을 배울 것입니다. 그래서, 우리는 `jiffies` 기반의 클록 소스의 초기화를 보았고 리눅스 커널은 커널이 작동하기 시작한 이 후 발생한 틱 수를 보유한느 전역 변수 `jiffies` 를 가지고 있다는 사실을 알고 있습니다. 이제 사용법을 봅시다. `jiffies` 를 사용하기 위해 이름이나 `get_jiffies_64`함수를 호출하여  `jiffies` 전역 변수를 사용할 수 있습니다. 이 함수는 [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 소스 코드 파일에 정의되어 있으며 `jiffies`의 `64-bit` 값만 리턴합니다. 

```C
u64 get_jiffies_64(void)
{
	unsigned long seq;
	u64 ret;

	do {
		seq = read_seqbegin(&jiffies_lock);
		ret = jiffies_64;
	} while (read_seqretry(&jiffies_lock, seq));
	return ret;
}
EXPORT_SYMBOL(get_jiffies_64);
```

`get_jiffies_64` 함수는 다음과 같이 `jiffies_read` 로 구현되지 않습니다.:

```C
static cycle_t jiffies_read(struct clocksource *cs)
{
	return (cycle_t) jiffies;
}
```

 `get_jiffies_64` 의 구현이 더 복잡하다는 것을 알 수 있습니다. `jiffies_64` 의 변수의 읽기는 [seqlocks](https://en.wikipedia.org/wiki/Seqlock)를 사용하여 구현됩니다. 실제로 이것은 전체 64 비트 값을 엄청 작은 단위를 읽을 수 없는 시스템에서 수행됩니다.

`jiffies` 또는 `jiffies_64` 변수에 액세스 할 수 있으면 이를 `human` 시간 단위로 변환 할 수 있습니다. 1초를 얻기 위해 다음 표현식을 사용할 수 있습니다:

```C
jiffies / HZ
```

그래서 우리가 이것을 알고 있다면 시간 단위를 얻을 수 있습니다. 예를 들면 다음과 같습니다:

```C
/* Thirty seconds from now */
jiffies + 30*HZ

/* Two minutes from now */
jiffies + 120*HZ

/* One millisecond from now */
jiffies + HZ / 1000
```

이게 답니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널에서 시간 및 시간 관리 관련 개념을 다루는 첫 번째 부분을 마치겠습니다. 우리는 먼저 두가지 개념과 초기화를 배웠습니다: `jiffies` 와 `clocksource`입니다. 다음 부분에서 우리는 이와 같은 흥미로운 주제를 계속해서 살펴볼 것이며 이 부분에서 이미 설명한 것처럼 리눅스 커널에서 이러한 시간 관리 개념과 다른 시간 관리 개념의 내부에 대해 이해하여 볼 것입니다.

질문이나 제안사항이 있으시면 언제든지 트위터 [0xAX](https://twitter.com/0xAX),에 핑을 보내거나 [email](anotherworldofworld@gmail.com) 을 보내거나  [issue](https://github.com/0xAX/linux-insides/issues/new)를 만들어 주세요.

**영어는 모국어가 아니며 불편을 끼쳐 드려 죄송합니다. 실수를 발견하면 PR을  [linux-insides](https://github.com/0xAX/linux-insides)로 보내주세요.**

링크
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol)
* [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)
* [cgroups](https://en.wikipedia.org/wiki/Cgroups)
* [bss](https://en.wikipedia.org/wiki/.bss)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms)
* [TSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [void](https://en.wikipedia.org/wiki/Void_type)
* [Simple Firmware Interface](https://en.wikipedia.org/wiki/Simple_Firmware_Interface)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [real time clock](https://en.wikipedia.org/wiki/Real-time_clock)
* [Jiffy](https://en.wikipedia.org/wiki/Jiffy_%28time%29)
* [high precision event timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)
* [nanoseconds](https://en.wikipedia.org/wiki/Nanosecond)
* [Intel 8253](https://en.wikipedia.org/wiki/Intel_8253)
* [seqlocks](https://en.wikipedia.org/wiki/Seqlock)
* [cloksource documentation](https://www.kernel.org/doc/Documentation/timers/timekeeping.txt)
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/SysCall/index.html)
