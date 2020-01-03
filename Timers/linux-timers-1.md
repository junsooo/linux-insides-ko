Linux 커널의 타이머 및 시간 관리. Part 1.

소개
--------------------------------------------------------------------------------

이 글은 [linux-insides](https://0xax.gitbooks.io/linux-insides/content/) 서적에서 새로운 장을 여는 또 다른 게시물입니다. 이전의 [부분](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-4.html)에서는 [시스템 호출](https://en.wikipedia.org/wiki/ System_call) 개념을 살펴 보았습니다. 이제 새 장을 시작할 차례입니다. 제목에서 알 수 있듯이이 장은 리눅스 커널의 `timers`와 `time management`에 대해 다룰 것 입니다. 현재 장에 대한 주제 선택은 우연이 아닙니다. 타이머(및 일반적으로 시간 관리)는 Linux 커널에서 매우 중요하며 널리 사용됩니다. 리눅스 커널은 다양한 작업, 예를 들어 [TCP](https://en.wikipedia.org/wiki/Transmission_Control_Protocol) 구현에서 다른 타임 아웃, 현재 시간을 알고있는 커널, 비동기 함수 스케줄링, 다음 이벤트 인터럽트 스케줄링 등의 타이머를 사용합니다.

그래서 우리는 이 부분에서 다른 시간 관리 관련된 것들의 구현을 배우기 시작할 것입니다. 우리는 다른 유형의 타이머와 다른 Linux 커널 서브 시스템이 이를 사용하는 방법을 볼 것입니다. 항상 그렇듯이 Linux 커널의 가장 빠른 부분부터 시작하여 Linux 커널의 초기화 프로세스를 진행합니다. 우리는 이미 리눅스 커널의 초기화 과정을 설명하는 특별한 [챕터](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)에서 이 작업을 수행했지만 몇 가지 부분을 놓쳤다는 것을 알고 있습니다. 그리고 그들 중 하나는 타이머 초기화입니다.

시작합시다.

비표준 PC 하드웨어 시계 초기화
--------------------------------------------------------------------------------

리눅스 커널이 압축 해제 된 후 (더 자세한 내용은 [커널 압축](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html) 부분에서 읽을 수 있습니다) 아키텍처 비 특정 코드가 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일에서 작동하기 시작합니다. [lock validator](https://www.kernel.org/doc/Documentation/locking/lockdep-design.txt)를 초기화하고 [cgroups](https://en.wikipedia.org/wiki/Cgroups)를 초기화하고 [canary](https://en.wikipedia.org/wiki/Buffer_overflow_protection) 값을 설정 한 후에는 `setup_arch` 함수의 호출을 볼 수 있습니다.

아시다시피 이 기능([arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L842)에 정의 되어있습니다.)은 아키텍처 관련 항목을 준비/초기화 합니다 (예 : [bss](https://en.wikipedia.org/wiki/.bss) 섹션을 위한 장소를 예약하고 [initrd](https://en.wikipedia.org/wiki/Initrd)를위한 장소를 예약합니다) 커널 명령 행 및 기타 여러 가지를 구문 분석합니다. 이 외에도 시간 관리 관련 기능이 있습니다.

처음은 이것 입니다:

```C
x86_init.timers.wallclock_init();
```

우리는 리눅스 커널의 초기화를 설명하는 장에서 이미 `x86_init` 구조체를 보았습니다. 이 구조체는 [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms), [Intel CE4100](http://www.wpgholdings.com/epaper/US/newsRelease_20091215/255874.pdf) 등과 같은 다른 플랫폼에 대한 기본 설정 기능에 대한 포인터를 포함합니다.`x86_init` 구조체는 [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c#L36)에 정의되어 있으며 기본적으로 표준 PC 하드웨어를 결정하는 것을 볼 수 있습니다.

보시다시피, `x86_init` 구조체에는 표준 자원 예약, 플랫폼 특정 메모리 설정, 인터럽트 핸들러 초기화 등과 같은 플랫폼 특정 설정을 위한 기능 세트를 제공하는 `x86_init_ops`유형이 있습니다. 이 구조체는 다음과 같습니다:

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

`x86_init_timers` 타입을 가진 `timers` 필드에 주목하십시오. 이 필드는 시간 관리 및 타이머와 관련이 있음을 그 이름으로 이해할 수 있습니다. `x86_init_timers`는 [void](https://en.wikipedia.org/wiki/Void_type)에서 포인터를 반환하는 모든 함수 인 4 개의 필드를 포함합니다:

* `setup_percpu_clockev` - 부팅 CPU에 CPU 당 이벤트 장치 설정;
* `tsc_pre_init` - [TSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter) init 이전에 호출 된 플랫폼 함수;
* `timer_init` - 플랫폼 타이머를 초기화;
* `wallclock_init` - 벽시계 장치 초기화.

따라서 이미 알고 있듯이 `wallwall_init`는 wallclock 장치의 초기화를 실행합니다. `x86_init` 구조체를 살펴보면 wallwall_init가 `x86_init_noop`을 가리키는 것을 볼 수 있습니다:

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

`x86_init_noop`은 아무 기능 없는 함수일뿐 입니다:

```C
void __cpuinit x86_init_noop(void) { }
```

표준 PC 하드웨어용. 실제로 wallwall_init 함수는 [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms) 플랫폼에서 사용됩니다. `x86_init.timers.wallclock_init`의 초기화는 `x86_intel_mid_early_setup` 함수의 [arch / x86 / platform / intel-mid / intel-mid.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/platform/intel-mid/intel-mid.c) 소스 코드 파일에 있습니다:

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

`intel_mid_rtc_init` 함수의 구현은 [arch/x86/platform/intel-mid/intel_mid_vrtc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/platform/intel-mid/intel_mid_vrtc.c) 소스 코드 파일에 있으며 매우 단순해 보입니다. 우선 이 함수는 [Simple Firmware Interface](https://en.wikipedia.org/wiki/Simple_Firmware_Interface) M-Real-Time-Clock 테이블을 구문 분석하여 그러한 장치를 `sfi_mrtc_array` 배열로 가져오고 `set_time` 및 `get_time` 함수를 초기화합니다:

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

그 후에는 `Intel MID`를 기반으로 한 장치가 하드웨어 시계에서 시간을 얻을 수 있습니다. 이미 쓴 것처럼 표준 PC [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처는 `x86_init_noop`을 지원하지 않으며 이 함수를 호출하는 동안 아무것도 하지않습니다. 방금 [Intel MID](https://en.wikipedia.org/wiki/Mobile_Internet_device#Intel_MID_platforms) 아키텍처에 대한 [실시간 시계](https://en.wikipedia.org/wiki/Real-time_clock)의 초기화를 보았습니다. 이제는 일반적인 `x86_64` 아키텍처로 돌아와서 시간 관리 관련 사항을 살펴 보겠습니다.

jiffies에 대한 숙지
--------------------------------------------------------------------------------

우리가 `setup_arch` 함수로 돌아 가면 (기억하듯이 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L842) 소스 코드 파일에 위치해 있습니다), 시간 관리 관련 함수의 다음 호출이 표시됩니다:

```C
register_refined_jiffies(CLOCK_TICK_RATE);
```


이 함수의 구현을 보기 전에 [jiffy](https://en.wikipedia.org/wiki/Jiffy_%28time%29)에 대해 알아야 합니다. wikipedia에서 읽을 수 있는 것처럼:

```
Jiffy는 지정되지 않은 짧은 기간 동안의 비공식 용어입니다.
```

이 정의는 Linux 커널의 `jiffy`와 매우 유사합니다. `jiffies`를 가진 전역 변수가 있는데, 이것은 시스템 부팅 이후 발생한 틱 수를 보유합니다. Linux 커널은 이 변수를 0으로 설정합니다:

```C
extern unsigned long volatile __jiffy_data jiffies;
```

초기화 과정에서. 이 전역 변수는 타이머 인터럽트 동안 매번 증가합니다. 이 외에도 `jiffies` 변수 근처에서 비슷한 변수의 정의를 볼 수 있습니다.

```C
extern u64 jiffies_64;
```

실제로 이러한 변수 중 하나만 Linux 커널에서 사용되며 프로세서 유형에 따라 다릅니다. [x86_64](https://en.wikipedia.org/wiki/X86-64)에서는 `u64`가 사용되며 [x86](https://en.wikipedia.org/wiki/X86)에서는 `signed long`이 됩니다. [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vmlinux.lds.S) 링커 스크립트를 보면 다음과 같습니다:

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

`x86_32`의 경우 `jiffies`는 `jiffies_64` 변수의 하위 32 비트가 됩니다. 대략적으로, 다음과 같이 상상할 수 있습니다.

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

이제 우리는 `jiffies`에 대한 약간의 이론을 알고 있으며 함수로 돌아갈 수 있습니다. 우리 함수에 대한 아키텍처 고유의 구현은 없습니다-`register_refined_jiffies`. 이 기능은 일반적인 커널 코드-[kernel / time / jiffies.c] 소스 코드 파일에 있습니다. `register_refined_jiffies`의 요점은 지피 `clocksource`의 등록입니다. `register_refined_jiffies`함수의 구현을 살펴보기 전에 `clocksource`가 무엇인지 알아야합니다. 주석에서 읽을 수 있듯이 :
```
`clocksource` 는 프리-런닝 카운터를 위한 하드웨어 추상화입니다.
```

확신하지 못하지만 저 설명으론 당신은 `clocksource`개념에 대해 잘 이해하지 못했습니다. 그것이 무엇인지 이해하려고 노력하겠지만,이 주제는 별도의 부분에서 훨씬 더 자세히 설명되므로 더 깊이 들어 가지 않을 것입니다. `클럭 소스`의 요점은 타임 키핑 추상화 또는 매우 간단한 단어로, 커널에 시간 값을 제공합니다. 우리는 이미 시스템 부팅 이후 발생한 틱 수를 나타내는 `jiffies` 인터페이스에 대해 알고 있습니다. Linux 커널에서 전역 변수로 표시되며 각 타이머 인터럽트가 증가합니다. 리눅스 커널은 시간 측정에 `jiffies`를 사용할 수 있습니다. 그렇다면 왜 우리가 `clocksource`와 같은 별도의 컨텍스트에서 필요합니까? 실제로 하드웨어 장치마다 기능에 따라 다른 클럭 소스를 제공합니다. 시간 간격 측정을 위한 보다 정확한 기술의 가용성은 하드웨어에 따라 다릅니다.

예를 들어, `x86`에는 [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)라는 64 비트 카운터가 내장되어 있으며 주파수는 프로세서 주파수와 동일 할 수 있습니다. 또는 예를 들어 [고정밀 이벤트 타이머](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)는 최소 `10MHz`주파수의 `64 비트`카운터로 구성됩니다. 두 개의 다른 타이머와 그 둘 다 `x86`입니다. 다른 아키텍처의 타이머를 추가하면 이 문제가 더 복잡해집니다. 리눅스 커널은 문제를 해결하기 위해 `clocksource`개념을 제공합니다.

clocksource 개념은 Linux 커널에서 `clocksource` 구조로 표현됩니다. 이 구조는 [include/linux/clocksource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h) 헤더 파일에 정의되어 있으며 설명하는 두 개의 필드를 포함합니다. 시간 카운터. 예를 들어, 카운터의 이름 인 `name` 필드, 카운터의 다른 속성을 설명하는 `flags` 필드, `suspend`및 `resume`함수에 대한 포인터 등을 포함합니다.

[kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 소스에 정의 된 지프에 대한 `clocksource` 구조를 살펴 보겠습니다. 코드 파일 :

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

여기서 기본 이름의 정의인 `jiffies`를 볼 수 있습니다. 다음은 `rating`필드로, 지정된 하드웨어에 사용 가능한 클록 소스 관리 코드로 최상의 등록 된 클록 소스를 선택할 수 있습니다. `rating`은 다음 값을 가질 수 있습니다:

* `1-99`    - 부팅 및 테스트 목적으로 만 사용 가능;
* `100-199` - 실제 사용을위한 기능이지만 바람직하지 않음.
* `200-299` - 정확하고 사용 가능한 클럭 소스.
* `300-399` - 빠르고 정확한 클럭 소스.
* `400-499` - 이상적인 클럭 소스. 가능한 경우 사용;

예를 들어 [타임 스탬프 카운터](https://en.wikipedia.org/wiki/Time_Stamp_Counter)의 등급은`300`이지만 [고정밀 이벤트 타이머](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)의 등급은`250`입니다. 다음 필드는 `읽기`입니다. 클럭 소스의 사이클 값을 읽을 수있는 함수의 포인터입니다. 즉,`cycle_t` 유형의 `jiffies` 변수 만 반환합니다:

```C
static cycle_t jiffies_read(struct clocksource *cs)
{
        return (cycle_t) jiffies;
}
```

그것은 단지 64 비트 부호없는 유형입니다:

```C
typedef u64 cycle_t;
```

다음 필드는 `마스크`값으로, 64 비트가 아닌 카운터에서 카운터 값을 빼는 데 특수 오버플로 논리가 필요하지 않습니다. 우리의 경우 마스크는 `0xffffffff`이고 `32` 비트입니다. 이것은 `jiffy`가 `42` 초 후에 0으로 줄어드는 것을 의미합니다:

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


다음 두 필드 인 `mult`및 `shift`는 클럭 소스의 주기를 사이클 당 나노 초로 변환하는 데 사용됩니다. 커널이 `clocksource.read` 함수를 호출하면, 이 함수는 우리가 지금 본 `cycle_t` 데이터 타입으로 표현 된 `machine`시간 단위의 값을 반환합니다. 이 리턴 값을 [nanoseconds](https://en.wikipedia.org/wiki/Nanosecond)로 변환하려면 `mult`와 `shift`라는 두 필드가 필요합니다. `clocksource`는 `clocksource_cyc2ns` 함수를 제공하여 다음과 같은 식으로 우리에게 도움이됩니다:

```C
((u64) cycles * mult) >> shift;
```

보시다시피 `mult`필드는 동일합니다:

```C
NSEC_PER_JIFFY << JIFFIES_SHIFT

#define NSEC_PER_JIFFY  ((NSEC_PER_SEC+HZ/2)/HZ)
#define NSEC_PER_SEC    1000000000L
```

기본적으로 `shift`는

```C
#if HZ < 34
  #define JIFFIES_SHIFT   6
#elif HZ < 67
  #define JIFFIES_SHIFT   7
#else
  #define JIFFIES_SHIFT   8
#endif
```

`jiffies` 클럭 소스는`NSEC_PER_JIFFY` 멀티 플라이어 변환을 사용하여 나노 초 오버 사이클 비율을 지정합니다. `JIFFIES_SHIFT` 및 `NSEC_PER_JIFFY`의 값은 `HZ`값에 따라 달라집니다. `HZ`는 시스템 타이머의 주파수를 나타냅니다. 이 매크로는 [include/asm-generic/param.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic/param.h)에 정의되어 있으며 `CONFIG_HZ`에 따라 다릅니다. 커널 구성 옵션. `HZ`의 값은 지원되는 아키텍처마다 다르지만 `x86`의 경우 다음과 같이 정의됩니다:


```C
#define HZ		CONFIG_HZ
```

여기서 `CONFIG_HZ`는 다음 값 중 하나 일 수 있습니다:

![HZ](http://i63.tinypic.com/v8d6ph.png)

이것은 우리의 경우에 타이머 인터럽트 주파수가 `250 HZ`이거나 초당 `250`번 발생하거나 각 `4ms`에 하나의 타이머 인터럽트가 발생한다는 것을 의미합니다.

`clocksource_jiffies` 구조체의 정의에서 볼 수있는 마지막 필드는 잠재적으로 오버플로를 발생시키지 않고 안전하게 곱할 수있는 최대 사이클 값을 보유하는 최대 사이클 값입니다.

자, 우리는 `clocksource_jiffies` 구조체의 정의를 보았습니다. 또한 우리는 `jiffies`와 `clocksource`에 대해 조금 알고 있습니다. 이제 우리 함수의 구현으로 돌아갈 시간입니다. 이 부분의 시작 부분에서 우리는 다음과 같은 호출을 중단했습니다.

```C
register_refined_jiffies(CLOCK_TICK_RATE);
```

[arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L842) 소스 코드 파일의 기능입니다.

이미 쓴 것처럼, `register_refined_jiffies` 함수의 주요 목적은 `refined_jiffies` 클럭 소스를 등록하는 것입니다. 우리는 이미 `clocksource_jiffies` 구조체가 표준 `jiffies` 클록 소스를 나타내는 것을 보았습니다. 이제 [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 소스 코드 파일을 보면 다른 클럭 소스 정의를 볼 수 있습니다:

```C
struct clocksource refined_jiffies;
```

`refined_jiffies`와 `clocksource_jiffies` 사이에는 한 가지 차이점이 있습니다. 표준 `jiffies` 기반 클록 소스는 모든 시스템에서 작동해야 하는 최저 공통 분모 클록 소스입니다. 우리가 이미 알고 있듯이, `jiffies` 전역 변수는 각 타이머 인터럽트 동안 증가합니다. 이는 표준 `jiffies`기반 클록 소스가 타이머 인터럽트 주파수와 동일한 해상도를 가짐을 의미합니다. 이로부터 우리는 표준 `jiffies` 기반 클럭 소스가 부정확성을 겪을 수 있음을 이해할 수 있습니다. `refined_jiffies`는 `jiffies` 시프트의 기본으로 `CLOCK_TICK_RATE`를 사용합니다.

이 함수의 구현을 보자. 우선, `clockclock_jiffies` 구조체에 기반한 `refined_jiffies` 클럭 소스를 볼 수 있습니다 :

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


여기서 우리는 `refined_jiffies`의 이름을 `refined-jiffies`로 업데이트하고 이 구조체의 등급을 높이는 것을 볼 수 있습니다. 기억하는 것처럼 `clocksource_jiffies`의 등급은 -1이며, 따라서 우리의 `refined_jiffies`의 클록 소스의 등급은 `2`입니다. 이는 `refined_jiffies`가 클럭 소스 관리 코드에 가장 적합한 선택임을 의미합니다.

다음 단계에서는 1 틱당 사이클 수를 계산해야합니다:

```C
cycles_per_tick = (cycles_per_second + HZ/2)/HZ;
```

우리는 `NSEC_PER_SEC` 매크로를 표준 `jiffies` 승수의 기본으로 사용했습니다. 여기서는 `register_refined_jiffies` 함수의 첫 번째 매개 변수 인 `cycles_per_second`를 사용하고 있습니다. 우리는 `CLOCK_TICK_RATE` 매크로를 `register_refined_jiffies` 함수에 전달했습니다. 이 매크로는 [arch/x86/include/asm/timex.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/timex.h) 헤더 파일에 정의되어 있습니다. 다음으로 확장됩니다:

```C
#define CLOCK_TICK_RATE         PIT_TICK_RATE
```

여기서 `PIT_TICK_RATE` 매크로는 [Intel 8253](프로그램 가능 간격 타이머)의 주파수로 확장됩니다:

```C
#define PIT_TICK_RATE 1193182ul
```

그런 다음 우리는 `hz << 8` 또는 다른 말로 시스템 타이머의 주파수를 저장할 `register_refined_jiffies`에 대해 `shift_hz`를 계산합니다. 정확성을 높이기 위해 `cycles_per_second` 또는 `8`의 프로그램 가능 간격 타이머의 주파수를 왼쪽으로 이동합니다:

```C
shift_hz = (u64)cycles_per_second << 8;
shift_hz += cycles_per_tick/2;
do_div(shift_hz, cycles_per_tick);
```

다음 단계에서 우리는 `shift_hz`와 마찬가지로 `NSEC_PER_SEC`를 `8`에 왼쪽으로 이동시켜 한 틱당 초 수를 계산하고 이전과 동일한 계산을 수행합니다:

```C
nsec_per_tick = (u64)NSEC_PER_SEC << 8;
nsec_per_tick += (u32)shift_hz/2;
do_div(nsec_per_tick, (u32)shift_hz);
```

```C
refined_jiffies.mult = ((u32)nsec_per_tick) << JIFFIES_SHIFT;
```

`register_refined_jiffies` 함수의 끝에서 우리는 [include/linux/clocksource.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d93eb973/include/linux/clocksource.h)에 정의 된 `__clocksource_register` 함수를 사용하여 새 클럭 소스를 등록합니다. 헤더 파일 및 반환 :

```C
__clocksource_register(&refined_jiffies);
return 0;
```

클럭 소스 관리 코드는 클럭 소스 등록 및 선택을 위한 API를 제공합니다. 보다시피, 클럭 초기화는 커널 초기화 중 또는 커널 모듈에서 `__clocksource_register` 함수를 호출하여 등록됩니다. 등록하는 동안 클럭 소스 관리 코드는 `jiffies`에 대한 `clocksource`구조체를 초기화 할 때 이미 보았던 `clocksource.rating` 필드를 사용하여 시스템에서 사용 가능한 최상의 클럭 소스를 선택합니다.

jiffies 사용
--------------------------------------------------------------------------------

이전 단락에서 두 개의 `jiffies` 기반 클록 소스의 초기화를 보았습니다:

* 표준 `jiffies` 기반 클록 소스;
* 재정의된 `jiffies` 기반 클럭 소스;

여기에서 계산을 이해하지 않아도 걱정하지 마십시오. 처음에는 무섭겠지만 곧 단계적으로 우리는 이러한 것들을 배울 것입니다. 그래서 우리는 `jiffies` 기반의 클럭 소스의 초기화를 보았고 Linux 커널은 커널이 작동하기 시작한 이후 발생한 틱 수를 보유하는 전역 변수 `jiffies`를 가지고 있다는 것을 알고 있습니다. 이제 사용법을 봅시다. `jiffies`를 사용하기 위해 이름이나 `get_jiffies_64` 함수를 호출하여 `jiffies` 전역 변수를 사용할 수 있습니다. 이 함수는 [kernel/time/jiffies.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/time/jiffies.c) 소스 코드 파일에 정의되어 있으며 `jifies`의 변수의 `64 비트` 전체를 반환합니다:

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

`get_jiffies_64` 함수는 다음과 같이 `jiffies_read`로 구현되지 않습니다:

```C
static cycle_t jiffies_read(struct clocksource *cs)
{
	return (cycle_t) jiffies;
}
```

`get_jiffies_64`의 구현이 더 복잡하다는 것을 알 수 있습니다. `jiffies_64` 변수의 읽기는 [seqlocks](https://en.wikipedia.org/wiki/Seqlock)를 사용하여 구현됩니다. 실제로 이것은 전체 64 비트 값을 원자적으로 읽을 수없는 시스템에서 수행됩니다.

`jiffies` 또는 `jiffies_64` 변수에 액세스 할 수 있으면 이를 휴먼 시간 단위로 변환 할 수 있습니다. 1 초를 얻기 위해 다음 표현식을 사용할 수 있습니다:

```C
jiffies / HZ
```

우리가 이것을 알고 있다면 시간 단위를 얻을 수 있습니다. 예를 들면 다음과 같습니다:

```C
/* Thirty seconds from now */
jiffies + 30*HZ

/* Two minutes from now */
jiffies + 120*HZ

/* One millisecond from now */
jiffies + HZ / 1000
```

그게 답니다.

결론
--------------------------------------------------------------------------------

이것으로 Linux 커널에서 시간 및 시간 관리 관련 개념을 다루는 첫 번째 부분을 마치겠습니다. 우리는 먼저 두 가지 개념과 초기화를 시작했습니다: `jiffies`와 `clocksource`. 다음 부분에서 우리는 이 흥미로운 주제를 계속해서 살펴볼 것이며, 이 부분에서 이미 쓴 것처럼, Linux 커널에서 이러한 시간 관리 개념과 다른 시간 관리 개념의 내부를 이해하려고 노력할 것입니다.

궁금한 점이나 제안이 있으시면 트위터 [0xAX](https://twitter.com/0xAX)에 저를 핑하거나 [이메일](anotherworldofworld@gmail.com)로 보내거나 [이슈](https://github.com/0xAX/linux-insides/issues/new)를 만드세요.

**영어는 제 모국어가 아니며 불편을 끼쳐 드려 죄송합니다. 실수를 발견하면 PR을 [linux-insides](https://github.com/0xAX/linux-insides)로 보내주십시오.**

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
