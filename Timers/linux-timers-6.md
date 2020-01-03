Linux 커널의 타이머 및 시간 관리. Part 6.
================================================================================

x86_64 관련 클럭 소스
--------------------------------------------------------------------------------

이것은 리눅스 커널에서 타이머와 시간 관리 관련 내용을 설명하는 [챕터](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)의 여섯 번째 부분입니다. 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-5.html)에서 우리는 `clockevents` 프레임 워크를 보았고 이제는 Linux 커널에서 시간 관리 관련 내용을 계속해서 살펴볼 것입니다. 이 부분에서는 [x86](https://en.wikipedia.org/wiki/X86) 아키텍처 관련 클럭 소스의 구현에 대해 설명합니다 (이 챕터의 [두 번째 파트](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-2.html)에서 읽을 수 있는 `clocksource`개념에 대한 자세한 내용을 볼 수 있습니다.)

우선 `x86` 아키텍처에서 어떤 클럭 소스를 사용할 수 있는지 알아야합니다. [sysfs](https://en.wikipedia.org/wiki/Sysfs) 또는 `/sys/devices/system/clocksource/clocksource0/available_clocksource`의 내용에서 쉽게 알 수 있습니다. `/sys/devices/system/clocksource/clocksourceN`은 이를 달성하기 위해 두 가지 특수 파일을 제공합니다:

* `available_clocksource` - 시스템에서 사용 가능한 클럭 소스에 대한 정보를 제공합니다.;
* `current_clocksource`   - 시스템에서 현재 사용되는 클럭 소스에 대한 정보를 제공합니다.

자, 보자:

```
$ cat /sys/devices/system/clocksource/clocksource0/available_clocksource
tsc hpet acpi_pm
```

내 시스템에 3 개의 등록 된 클럭 소스가 있음을 알 수 있습니다:

* `tsc` - [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter);
* `hpet` - [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer);
* `acpi_pm` - [ACPI Power Management Timer](http://uefi.org/sites/default/files/resources/ACPI_5.pdf).

이제 최고의 클럭 소스 (시스템에서 가장 높은 등급을 가진 클럭 소스)를 제공하는 두 번째 파일을 살펴 보겠습니다:

```
$ cat /sys/devices/system/clocksource/clocksource0/current_clocksource
tsc
```

저의 경우 [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)입니다. 이 장의 [두 번째 부분](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-2.html)에서 알 수 있듯이 리눅스 커널에서 `clocksource` 프레임 워크의 내부를 설명하고 있듯이, 시스템에서 가장 좋은 클럭 소스는 최고 (최고) 등급을 가진 클럭 소스입니다. [주파수](https://en.wikipedia.org/wiki/Frequency)가 가장 높습니다.

[ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) 전원 관리 타이머의 주파수는 `3.579545 MHz`입니다. [고정밀 이벤트 타이머](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)의 주파수는 `10 MHz`이상입니다. [타임 스탬프 카운터](https://en.wikipedia.org/wiki/Time_Stamp_Counter)의 주파수는 프로세서에 따라 다릅니다. 예를 들어 구형 프로세서에서 `타임 스탬프 카운터`는 내부 프로세서 클럭 주기를 계산했습니다. 이는 프로세서의 주파수 스케일링이 변경 될 때 주파수가 변경되었음을 의미합니다. 최신 프로세서의 상황이 변경되었습니다. 최신 프로세서에는 프로세서의 모든 작동 상태에서 일정한 속도로 증가하는 `불변 타임 스탬프 카운터`가 있습니다. 실제로 우리는 `/proc/cpuinfo`의 출력에서 주파수를 얻을 수 있습니다. 예를 들어 시스템의 첫 번째 프로세서의 경우:

```
$ cat /proc/cpuinfo
...
model name	: Intel(R) Core(TM) i7-4790K CPU @ 4.00GHz
...
```

인텔 매뉴얼에 따르면 `타임 스탬프 카운터`의 주파수는 일정하고 프로세서의 최대 적격 주파수 또는 브랜드 문자열에 지정된 주파수 일 필요는 없지만 어쨌든 `ACPI PM` 타이머 또는 `고정밀 이벤트 타이머의 주파수` 보다 더 클 것입니다. 그리고 우리는 시스템에서 최고의 정격 또는 가장 높은 주파수를 가진 클럭 소스가 전류임을 알 수 있습니다.

이 3 개의 클록 소스 외에, `/sys/devices/system/clocksource/clocksource0/available_clocksource`의 출력에서 또 다른 2 개의 친숙한 클록 소스가 보이지 않습니다. 이 클럭 소스는 `jiffy`와 `refined_jiffies`입니다. 이 파일은 고해상도 클럭 소스 또는 다른 말로 [CLOCK_SOURCE_VALID_FOR_HRES](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/clocksource.h#L113) 플래그를 가진 클럭 소스 만 매핑하기 때문에 우리는 그것들을 볼 수 없습니다.

위에서 이미 썼 듯이이 부분에서이 세 가지 클럭 소스를 모두 고려할 것입니다. 초기화 순서대로 고려하거나:

* `hpet`;
* `acpi_pm`;
* `tsc`.

[dmesg](https://en.wikipedia.org/wiki/Dmesg) 유틸리티의 출력에서 순서가 정확히 이와 같은지 확인할 수 있습니다:

```
$ dmesg | grep clocksource
[    0.000000] clocksource: refined-jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1910969940391419 ns
[    0.000000] clocksource: hpet: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 133484882848 ns
[    0.094369] clocksource: jiffies: mask: 0xffffffff max_cycles: 0xffffffff, max_idle_ns: 1911260446275000 ns
[    0.186498] clocksource: Switched to clocksource hpet
[    0.196827] clocksource: acpi_pm: mask: 0xffffff max_cycles: 0xffffff, max_idle_ns: 2085701024 ns
[    1.413685] tsc: Refined TSC clocksource calibration: 3999.981 MHz
[    1.413688] clocksource: tsc: mask: 0xffffffffffffffff max_cycles: 0x73509721780, max_idle_ns: 881591102108 ns
[    2.413748] clocksource: Switched to clocksource tsc
```

첫 번째 클럭 소스는 [고정밀 이벤트 타이머](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)이므로 여기서 부터 시작하겠습니다.

고정밀 이벤트 타이머
--------------------------------------------------------------------------------


[x86](https://en.wikipedia.org/wiki/X86) 아키텍처에 대한 `고정밀 이벤트 타이머`의 구현은 [arch/x86/ kernel/hpet.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/hpet.c) 소스 코드 파일에 있습니다. 초기화는 `hpet_enable` 함수의 호출에서 시작됩니다. 이 함수는 Linux 커널 초기화 중에 호출됩니다. [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일에서 `start_kernel` 함수를 살펴보면, 모든 아키텍처 관련 항목이 초기화 된 후 초기 콘솔이 비활성화되고 시간 관리 서브 시스템이 이미 준비되어 있음을 알 수 있습니다. 다음 함수를 호출하면:

```C
if (late_time_init)
	late_time_init();
```

초기 지피 카운터가 이미 초기화 된 후 늦은 아키텍처 특정 타이머의 초기화를 수행합니다. `x86` 아키텍처에 대한`late_time_init` 함수의 정의는 [arch/x86/kernel/time.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel / time.c)소스 코드 파일에 있습니다. 꽤 쉬워 보입니다:

```C
static __init void x86_late_time_init(void)
{
	x86_init.timers.timer_init();
	tsc_init();
}
```

보시다시피, `x86` 관련 타이머의 초기화와 `타임 스탬프 카운터`의 초기화를 합니다. 다음 단락에서 보겠지만, 이제 `x86_init.timers.timer_init` 함수 호출을 고려해 보자. `timer_init`는 동일한 소스 코드 파일에서 `hpet_time_init` 함수를 가리 킵니다. [arch/x86/kernelx86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c)에서 `x86_init` 구조체의 정의를 보면 이를 확인할 수 있습니다:

```C
struct x86_init_ops x86_init __initdata = {
   ...
   ...
   ...
   .timers = {
		.setup_percpu_clockev	= setup_boot_APIC_clock,
		.timer_init		= hpet_time_init,
		.wallclock_init		= x86_init_noop,
   },
   ...
   ...
   ...
```

`High Precision Event Timer`를 활성화 할 수없는 경우 `hpet_time_init` 함수는 [programmable interval timer](https://en.wikipedia.org/wiki/Programmable_interval_timer)를 설정하고 활성화 된 타이머에 대한 기본 타이머 [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)를 설정합니다:

```C
void __init hpet_time_init(void)
{
	if (!hpet_enable())
		setup_pit_timer();
	setup_default_timer_irq();
}
```

먼저,`hpet_enable` 기능 검사에서 `is_hpet_capable` 함수를 호출하여 시스템에서 `High Precision Event Timer`를 활성화 할 수 있으며 가능하면 가상 주소 공간을 매핑합니다:

```C
int __init hpet_enable(void)
{
	if (!is_hpet_capable())
		return 0;

    hpet_set_mapping();
}
```

`is_hpet_capable` 함수는 커널 명령 행에 `hpet = disable`을 전달하지 않았으며 `hpet_address`가 [ACPI HPET](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) 테이블에서 수신되는지 확인합니다. `hpet_set_mapping` 함수는 타이머 레지스터의 가상 주소 공간을 매핑합니다:

```C
hpet_virt_address = ioremap_nocache(hpet_address, HPET_MMAP_SIZE);
```

[IA-PC HPET (High Precision Event Timers) Specification](http://www.intel.com/content/dam/www/public/us/en/documents/technical-specifications/software-developers-hpet-spec-1-0a.pdf)에서 읽을 수 있듯이:

> 타이머 레지스터 공간은 1024 바이트입니다

따라서 `HPET_MMAP_SIZE`도 `1024`바이트입니다:

```C
#define HPET_MMAP_SIZE		1024
```

`고정밀 이벤트 타이머`에 대한 가상 공간을 매핑 한 후, HPET_ID 레지스터를 읽고 타이머 수를 얻습니다:

```C
id = hpet_readl(HPET_ID);

last = (id & HPET_ID_NUMBER) >> HPET_ID_NUMBER_SHIFT;
```

`고정밀 이벤트 타이머`의 `일반 구성 레지스터`에 올바른 공간을 할당하려면 이 숫자를 가져와야합니다:

```C
cfg = hpet_readl(HPET_CFG);

hpet_boot_cfg = kmalloc((last + 2) * sizeof(*hpet_boot_cfg), GFP_KERNEL);
```

공간이 `고정밀 이벤트 타이머`의 구성 레지스터에 할당 된 후, 우리는 메인 카운터의 실행을 허용하고 모든 타이머에 대한 구성 레지스터에서 `HPET_CFG_ENABLE` 비트의 설정에 의해 타이머 인터럽트가 활성화되면 타이머 인터럽트를 허용합니다. 결국 우리는 `hpet_clocksource_register` 함수를 호출하여 새로운 클럭 소스를 등록합니다:

```C
if (hpet_clocksource_register())
	goto out_nohpet;
```

이미 익숙한 호출

```C
clocksource_register_hz(&clocksource_hpet, (u32)hpet_freq);
```

함수. `clocksource_hpet`이 `250`등급 (이전 `refined_jiffies` 클럭 소스의 등급을 기억했던 `2`)을 갖는 `clocksource` 구조체인 경우, 이름- `hpet` 및 `read_hpet` 콜백은 원자 카운터 판독 `고정밀 이벤트 타이머`에서 제공합니다:

```C
static struct clocksource clocksource_hpet = {
	.name		= "hpet",
	.rating		= 250,
	.read		= read_hpet,
	.mask		= HPET_MASK,
	.flags		= CLOCK_SOURCE_IS_CONTINUOUS,
	.resume		= hpet_resume_counter,
	.archdata	= { .vclock_mode = VCLOCK_HPET },
};
```

`clocksource_hpet`이 등록되면 [arch/x86/kernel/time.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/time.c) 소스 코드 파일에서 `hpet_time_init ()`함수로 돌아갈 수 있습니다. 마지막 단계는 다음을 호출한다는 것을 기억할 수 있습니다:

```C
setup_default_timer_irq();
```

`hpet_time_init ()`의 함수. `setup_default_timer_irq` 함수는 `레거시` IRQ의 존재를 확인하거나 다른 말로 [i8259](https://en.wikipedia.org/wiki/Intel_8259)에 대한 지원 및 설정 [IRQ0](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29#Master_PIC)에 따라 달라집니다.

그게 다 입니다. 이 순간부터 Linux 커널 `clock source` 프레임 워크에 등록 된 [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer) 클럭 소스는 `read_hpet`을 통해 일반 커널 코드에서 사용될 수 있습니다:

```C
static cycle_t read_hpet(struct clocksource *cs)
{
	return (cycle_t)hpet_readl(HPET_COUNTER);
}
```

`Main Counter Register`에서 원자 카운터를 읽고 반환하는 함수.

ACPI PM 타이머
--------------------------------------------------------------------------------

초 클럭 소스는 [ACPI 전원 관리 타이머](http://uefi.org/sites/default/files/resources/ACPI_5.pdf) 입니다. 이 클럭 소스의 구현은 [drivers/clocksource/acpi_pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/clocksource_acpi_pm.c) 소스 코드 파일에 있으며 `fs` [initcall](https://kernelnewbies.org/Documents/InitcallMechanism) 동안 `init_acpi_pm_clocksource` 함수의 호출에서 시작합니다.

`init_acpi_pm_clocksource`함수의 구현을 살펴보면, `pmtmr_ioport`변수의 값을 확인하는 것으로 시작한다는 것을 알 수 있습니다:

```C
static int __init init_acpi_pm_clocksource(void)
{
    ...
    ...
    ...
	if (!pmtmr_ioport)
		return -ENODEV;
    ...
    ...
    ...
```

이 `pmtmr_ioport` 변수는 `전원 관리 타이머 제어 레지스터 블록`의 확장 주소를 포함합니다. [arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/acpi/boot.c) 소스 코드 파일에 정의 된`acpi_parse_fadt` 함수에서 값을 가져옵니다. 이 함수는 `FADT` 또는 `고정 ACPI 설명 테이블` [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) 테이블을 구문 분석하고 `일반 주소 구조체` 형식에 표시된 `전원 관리 타이머 제어 레지스터 블록`의 확장 주소를 포함하는 `X_PM_TMR_BLK`필드의 값을 가져 오려고 시도합니다:

```C
static int __init acpi_parse_fadt(struct acpi_table_header *table)
{
#ifdef CONFIG_X86_PM_TIMER
        ...
        ...
        ...
		pmtmr_ioport = acpi_gbl_FADT.xpm_timer_block.address;
        ...
        ...
        ...
#endif
	return 0;
}
```

따라서 `CONFIG_X86_PM_TIMER` Linux 커널 구성 옵션이 비활성화되었거나 `acpi_parse_fadt` 기능에서 문제가 발생하면 `Power Management Timer` 레지스터에 액세스 할 수 없으며 `init_acpi_pm_clocksource`에서 돌아올 수 없습니다. 다른 방법으로, `pmtmr_ioport`변수의 값이 0이 아닌 경우이 타이머의 속도를 확인하고 다음을 호출하여 이 클럭 소스를 등록합니다:

```C
clocksource_register_hz(&clocksource_acpi_pm, PMTMR_TICKS_PER_SEC);
```

기능. `clocksource_register_hs`를 호출 한 후 `acpi_pm` 클럭 소스는 Linux 커널의 `clocksource` 프레임 워크에 등록됩니다:

```C
static struct clocksource clocksource_acpi_pm = {
	.name		= "acpi_pm",
	.rating		= 200,
	.read		= acpi_pm_read,
	.mask		= (cycle_t)ACPI_PM_MASK,
	.flags		= CLOCK_SOURCE_IS_CONTINUOUS,
};
```

`acpi_pm` 클럭 소스가 제공하는 원자 카운터를 읽기 위한 `200` 및 `acpi_pm_read` 콜백 등급. `acpi_pm_read` 함수는 `read_pmtmr` 함수를 실행합니다:

```C
static cycle_t acpi_pm_read(struct clocksource *cs)
{
	return (cycle_t)read_pmtmr();
}
```

전원 관리 타이머 레지스터 값을 읽는다. 이 레지스터의 구조체는 다음과 같습니다:

```
+-------------------------------+----------------------------------+
|                               |                                  |
|  upper eight bits of a        |      running count of the        |
| 32-bit power management timer |     power management timer       |
|                               |                                  |
+-------------------------------+----------------------------------+
31          E_TMR_VAL           24               TMR_VAL           0
```

이 레지스터의 주소는 `고정 된 ACPI 설명 테이블` [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) 테이블에 저장되어 있으며 이미 `pmtmr_ioport`에 있습니다. 따라서 `read_pmtmr`함수의 구현은 매우 쉽습니다:

```C
static inline u32 read_pmtmr(void)
{
	return inl(pmtmr_ioport) & ACPI_PM_MASK;
}
```

`전원 관리 타이머` 레지스터의 값을 읽고 `24`비트를 마스크합니다.

그게 다 입니다. 이제 이 파트의 마지막 클럭 소스 인 `Time Stamp Counter`로 이동합니다.

타임 스탬프 카운터
--------------------------------------------------------------------------------

이 파트의 세 번째이자 마지막 클럭 소스는 - [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter) 클럭 소스이며 그 구현은 [arch/x86/kernel/tsc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/tsc.c) 소스 코드 파일에 있습니다. 우리는 이미이 부분에서 `x86_late_time_init` 함수를 보았고 [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)의 초기화는 여기서 시작됩니다. 이 함수는 [arch/x86/kernel /tsc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/tsc.c) 소스 코드 파일에서 `tsc_init ()` 함수를 호출합니다.

`tsc_init` 함수의 시작 부분에서 check는 프로세서가 `Time Stamp Counter`를 지원하는지 확인하는 것을 볼 수 있습니다:

```C
void __init tsc_init(void)
{
	u64 lpj;
	int cpu;

	if (!cpu_has_tsc) {
		setup_clear_cpu_cap(X86_FEATURE_TSC_DEADLINE_TIMER);
		return;
	}
    ...
    ...
    ...
```

`cpu_has_tsc` 매크로는 `cpu_has` 매크로의 호출로 확장됩니다:

```C
#define cpu_has_tsc		boot_cpu_has(X86_FEATURE_TSC)

#define boot_cpu_has(bit)	cpu_has(&boot_cpu_data, bit)

#define cpu_has(c, bit)							\
	(__builtin_constant_p(bit) && REQUIRED_MASK_BIT_SET(bit) ? 1 :	\
	 test_cpu_cap(c, bit))
```

초기 리눅스 커널 초기화 동안 채워지는 `boot_cpu_data` 배열에서 주어진 비트 (이 경우 `X86_FEATURE_TSC_DEADLINE_TIMER`)를 검사합니다. 프로세서가 `타임 스탬프 카운터`를 지원하는 경우
[Model Specific Register](https://en.wikipedia.org/wiki/Model-specific_register)와 같은 다른 소스에서 주파수를 얻으려고하는 동일한 소스 코드 파일에서 `calibrate_tsc` 함수를 호출하여 `Time Stamp Counter`의 주파수를 얻습니다. [프로그램 간격 타이머](https://en.wikipedia.org/wiki/Programmable_interval_timer) 등을 통해 교정 그런 다음 시스템의 모든 프로세서에 대한 주파수 및 스케일 팩터를 초기화합니다:

```C
tsc_khz = x86_platform.calibrate_tsc();
cpu_khz = tsc_khz;

for_each_possible_cpu(cpu) {
	cyc2ns_init(cpu);
	set_cyc2ns_scale(cpu_khz, cpu);
}
```

첫 번째 부트 스트랩 프로세서만 `tsc_init`를 호출하기 때문입니다. 그런 다음 조건 `Time Stamp Counter`가 비활성화되어 있지 않은지 확인하십시오:

```
if (tsc_disabled > 0)
	return;
...
...
...
check_system_tsc_reliable();
```

부트 스트랩 프로세서에 `X86_FEATURE_TSC_RELIABLE` 기능이 있는 경우 `tsc_clocksource_reliable`을 설정하는 `check_system_tsc_reliable` 함수를 호출하십시오. 우리는 `tsc_init` 함수를 겪었지만 클럭 소스를 등록하지는 않았습니다. `Time Stamp Counter` 클럭 소스의 실제 등록은 다음에서 발생합니다:

```C
static int __init init_tsc_clocksource(void)
{
	if (!cpu_has_tsc || tsc_disabled > 0 || !tsc_khz)
		return 0;
    ...
    ...
    ...
    if (boot_cpu_has(X86_FEATURE_TSC_RELIABLE)) {
		clocksource_register_khz(&clocksource_tsc, tsc_khz);
		return 0;
	}
```

함수. 이 함수는 `device` [initcall](https://kernelnewbies.org/Documents/InitcallMechanism) 중에 호출되었습니다. 타임 스탬프 카운터 클럭 소스는 [고정밀 이벤트 타이머](https://en.wikipedia.org/wiki/High_Precision_Event_Timer) 클럭 소스 다음에 등록 되도록 해야 합니다.

이 3 개의 클럭 소스가 모두 `clocksource`프레임 워크에 등록되고 `Time Stamp Counter`클럭 소스가 다른 클럭 소스 중에서 가장 높은 등급을 갖기 때문에 활성으로 선택됩니다:

```C
static struct clocksource clocksource_tsc = {
	.name                   = "tsc",
	.rating                 = 300,
	.read                   = read_tsc,
	.mask                   = CLOCKSOURCE_MASK(64),
	.flags                  = CLOCK_SOURCE_IS_CONTINUOUS | CLOCK_SOURCE_MUST_VERIFY,
	.archdata               = { .vclock_mode = VCLOCK_TSC },
};
```

그게 다 입니다.

결론
--------------------------------------------------------------------------------

이것은 리눅스 커널에서 타이머와 타이머 관리 관련 내용을 설명하는 [chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)의 여섯 번째 부분의 끝입니다. 이전 부분에서는 `clockevents`프레임 워크에 대해 알게되었습니다. 이 부분에서 우리는 리눅스 커널에서 시간 관리 관련 내용을 계속 배우고 [x86](https://en.wikipedia.org/wiki/X86) 아키텍처에서 사용되는 약 3 가지의 다른 클럭 소스를 보았습니다. 다음 부분은 이 [chapter](https://0xax.gitbooks.io/linux-insides/content/Timers/index.html)의 마지막 부분이며 몇 가지 사용자 공간 관련 내용, 즉 시간 관련 [시스템 호출](https://en.wikipedia.org/wiki/System_call)은 Linux 커널에서 구현되었습니다.

궁금한 점이나 제안이 있으시면 트위터 [0xAX](https://twitter.com/0xAX)에 저를 핑(ping)하거나 [이메일](anotherworldofworld@gmail.com)로 보내거나 [이슈](https://github.com/0xAX/linux-insides/issues/new)를 만드세요.

** 영어는 제 모국어가 아니고 불편을 끼쳐 드려 죄송합니다. 실수를 발견하면 PR을 [linux-insides](https://github.com/0xAX/linux-insides)로 보내주십시오.**

링크
--------------------------------------------------------------------------------

* [x86](https://en.wikipedia.org/wiki/X86)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [High Precision Event Timer](https://en.wikipedia.org/wiki/High_Precision_Event_Timer)
* [ACPI Power Management Timer (PDF)](http://uefi.org/sites/default/files/resources/ACPI_5.pdf)
* [frequency](https://en.wikipedia.org/wiki/Frequency).
* [dmesg](https://en.wikipedia.org/wiki/Dmesg)
* [programmable interval timer](https://en.wikipedia.org/wiki/Programmable_interval_timer)
* [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [IA-PC HPET (High Precision Event Timers) Specification](http://www.intel.com/content/dam/www/public/us/en/documents/technical-specifications/software-developers-hpet-spec-1-0a.pdf)
* [IRQ0](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29#Master_PIC)
* [i8259](https://en.wikipedia.org/wiki/Intel_8259)
* [initcall](https://kernelnewbies.org/Documents/InitcallMechanism)
* [previous part](https://0xax.gitbooks.io/linux-insides/content/Timers/linux-timers-5.html)
