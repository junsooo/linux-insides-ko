initcall 메커니즘
================================================================================

소개
--------------------------------------------------------------------------------

제목에서 알 수 있듯이 이 부분은 리눅스 커널에서 흥미롭고 중요한 개념인 `initcall`을 다룰 것입니다. 우리는 이미 다음과 같은 정의를 보았습니다.

```C
early_param("debug", debug_kernel);
```

or

```C
arch_initcall(init_pit_clocksource);
```

리눅스 커널의 일부에서. 이 메커니즘이 Linux 커널에서 어떻게 구현되는지 살펴보기 전에 실제로 그 메커니즘과 Linux 커널이 이를 사용하는 방법을 알아야합니다. 이와 같은 정의는 [콜백](https://en.wikipedia.org/wiki/Callback_%28computer_programming%29) 함수를 나타냅니다.이 함수는 바로 Linux 커널 초기화 중에 호출됩니다. 실제로 `initcall` 메커니즘의 핵심은 내장 모듈과 서브 시스템 초기화의 올바른 순서를 결정하는 것입니다. 예를 들어 다음 기능을 살펴 보겠습니다.

```C
static int __init nmi_warning_debugfs(void)
{
    debugfs_create_u64("nmi_longest_ns", 0644,
                       arch_debugfs_dir, &nmi_longest_ns);
    return 0;
}
```

[arch/x86/kernel/nmi.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/nmi.c) 소스 코드 파일에서 보시다시피 `arch_debugfs_dir` 디렉토리에 `nmi_longest_ns` [debugfs](https://en.wikipedia.org/wiki/Debugfs) 파일 만 생성됩니다. 실제로 이 `debugfs` 파일은 `arch_debugfs_dir`이 생성 된 후에만 생성 될 수 있습니다. 이 디렉토리는 Linux 커널의 아키텍처 별 초기화 중에 생성됩니다. 실제로 이 디렉토리는 [arch/x86/kernel/kdebugfs.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/kdebugfsc)의 `arch_kdebugfs_init` 함수에 생성됩니다. 소스 코드 파일 `arch_kdebugfs_init` 함수도 `initcall`로 표시됩니다:

```C
arch_initcall(arch_kdebugfs_init);
```

리눅스 커널은 `fs` 관련 `initcalls` 전에 모든 아키텍처 특정 `initcalls`를 호출합니다. 따라서 `nmi_longest_ns` 파일은 `arch_kdebugfs_dir` 디렉토리가 생성 된 후에만 생성됩니다. 실제로 리눅스 커널은 8 가지 레벨의 `initcalls`을 제공합니다:

* `early`;
* `core`;
* `postcore`;
* `arch`;
* `subsys`;
* `fs`;
* `device`;
* `late`.

모든 이름은 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일에 정의 된 `initcall_level_names` 배열로 표시됩니다:

```C
static char *initcall_level_names[] __initdata = {
	"early",
	"core",
	"postcore",
	"arch",
	"subsys",
	"fs",
	"device",
	"late",
};
```

이 식별자에 의해 `initcall`로 표시된 모든 함수는 동일한 순서로 호출되거나 처음에는 `초기 initcalls`, 두 번째는 `core initcalls`등에서 호출됩니다.이 순간부터 우리는 `initcall`에 대해 조금 알고 있습니다. `메커니즘, 그래서 우리는 이 메커니즘이 어떻게 구현되는지보기 위해 리눅스 커널의 소스 코드로 뛰어 들기 시작할 수있다.

리눅스 커널에서 initcall 메커니즘 구현
--------------------------------------------------------------------------------

리눅스 커널은 [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) 헤더 파일에서 매크로 집합을 제공하여 주어진 함수를 `initcall`로 표시합니다. 이 매크로는 모두 매우 간단합니다.

```C
#define early_initcall(fn)		__define_initcall(fn, early)
#define core_initcall(fn)		__define_initcall(fn, 1)
#define postcore_initcall(fn)		__define_initcall(fn, 2)
#define arch_initcall(fn)		__define_initcall(fn, 3)
#define subsys_initcall(fn)		__define_initcall(fn, 4)
#define fs_initcall(fn)			__define_initcall(fn, 5)
#define device_initcall(fn)		__define_initcall(fn, 6)
#define late_initcall(fn)		__define_initcall(fn, 7)
```

우리가 볼 수 있듯이 이러한 매크로는 동일한 헤더 파일에서 `__define_initcall` 매크로의 호출로 확장됩니다. 또한, `__define_initcall` 매크로는 두 개의 인자를 사용합니다:

* `fn` - 특정 레벨의 `initcalls` 호출 중에 호출되는 콜백 함수;
* `id` - 동일한 두 개의 `initcalls`가 동일한 핸들러를 가리키는 경우 오류를 방지하기 위해 `initcall`을 식별하는 식별자.

`__define_initcall` 매크로의 구현은 다음과 같습니다:

```C
#define __define_initcall(fn, id) \
	static initcall_t __initcall_##fn##id __used \
	__attribute__((__section__(".initcall" #id ".init"))) = fn; \
	LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

`__define_initcall` 매크로를 이해하기 위해서는 먼저 `initcall_t`타입을 살펴 봅시다. 이 타입은 같은 [header]() 파일에 정의되어 있으며 `initcall`의 결과인 [integer](https://en.wikipedia.org/wiki/Integer)에 대한 포인터를 반환하는 함수에 대한 포인터를 나타냅니다:

```C
typedef int (*initcall_t)(void);
```

이제 `_-define_initcall` 매크로로 돌아 갑시다. [##](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)은 두 개의 심볼을 연결하는 기능을 제공합니다. 우리의 경우, `__define_initcall` 매크로의 첫 번째 줄은 `.initcall id .init` [ELF section](http://www.skyfree.org/linux/references/ELF_Format.pdf)에 있고 다음의 [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) 속성으로 표시되는 주어진 함수의 정의를 생성합니다: `__initcall_function_name_id` 및 `__used`. 커널 [linker](https://en.wikipedia.org/wiki/Linker_%28computing%29) 스크립트의 데이터를 나타내는 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic/vmlinux.lds.h) 헤더 파일을 살펴보면 모든 `initcalls` 섹션이 `.data` 섹션에 배치됩니다:

```C
#define INIT_CALLS					\
		VMLINUX_SYMBOL(__initcall_start) = .;	\
		*(.initcallearly.init)					\
		INIT_CALLS_LEVEL(0)					    \
		INIT_CALLS_LEVEL(1)					    \
		INIT_CALLS_LEVEL(2)					    \
		INIT_CALLS_LEVEL(3)					    \
		INIT_CALLS_LEVEL(4)					    \
		INIT_CALLS_LEVEL(5)					    \
		INIT_CALLS_LEVEL(rootfs)				\
		INIT_CALLS_LEVEL(6)					    \
		INIT_CALLS_LEVEL(7)					    \
		VMLINUX_SYMBOL(__initcall_end) = .;

#define INIT_DATA_SECTION(initsetup_align)	\
	.init.data : AT(ADDR(.init.data) - LOAD_OFFSET) {	   \
        ...                                                \
        INIT_CALLS						                   \
        ...                                                \
	}

```

두 번째 속성 인 `__used`는 [include/linux/compiler-gcc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/compiler-gcc.h) 헤더에 정의되어 있습니다. 파일과 다음 `gcc` 속성의 정의로 확장됩니다:

```C
#define __used   __attribute__((__used__))
```

변수 정의되었지만 사용되지 않은 경고를 방지합니다. `__define_initcall` 매크로의 마지막 줄은 다음과 같습니다:

```C
LTO_REFERENCE_INITCALL(__initcall_##fn##id)
```

`CONFIG_LTO` 커널 설정 옵션에 의존하고 컴파일러를 위한 스텁을 제공합니다. [링크 시간 최적화](https://gcc.gnu.org/wiki/LinkTimeOptimization):

```
#ifdef CONFIG_LTO
#define LTO_REFERENCE_INITCALL(x) \
        static __used __exit void *reference_##x(void)  \
        {                                               \
                return &x;                              \
        }
#else
#define LTO_REFERENCE_INITCALL(x)
#endif
```

모듈에 변수에 대한 참조가 없을 때 문제를 방지하기 위해 프로그램의 끝으로 이동합니다. 이것이 `__define_initcall` 매크로에 관한 것입니다. 따라서 모든 `* _initcall` 매크로는 Linux 커널 컴파일 중에 확장되며 모든 `initcalls`는 해당 섹션에 배치되며 모든 `.data` 섹션에서 사용할 수 있으며 Linux 커널은 초기화 과정에서 호출 할 특정 `initcall`을 찾을 수 있는 곳을 알고 있어야합니다.

리눅스 커널이 `initcalls`를 호출 할 수 있기 때문에 리눅스 커널이 이를 어떻게 수행하는지 살펴 보자. 이 프로세스는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일의 `do_basic_setup` 기능에서 시작합니다:

```C
static void __init do_basic_setup(void)
{
    ...
    ...
    ...
   	do_initcalls();
    ...
    ...
    ...
}
```

메모리 관리자 관련 초기화,`CPU` 서브 시스템 및 기타 이미 완료된 주요 초기화 단계 직후에 Linux 커널 초기화 중에 호출됩니다. `do_initcalls` 함수는 `initcall` 레벨의 배열을 거치고 각 레벨에 대해 `do_initcall_level` 함수를 호출합니다:

```C
static void __init do_initcalls(void)
{
	int level;

	for (level = 0; level < ARRAY_SIZE(initcall_levels) - 1; level++)
		do_initcall_level(level);
}
```

`initcall_levels` 배열은 동일한 소스 코드 [file](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에 정의되며`__define_initcall` 매크로에 정의 된 섹션에 대한 포인터를 포함합니다.

```C
static initcall_t *initcall_levels[] __initdata = {
	__initcall0_start,
	__initcall1_start,
	__initcall2_start,
	__initcall3_start,
	__initcall4_start,
	__initcall5_start,
	__initcall6_start,
	__initcall7_start,
	__initcall_end,
};
```

관심이 있다면 리눅스 커널 컴파일 후에 생성 된 `arch/x86/kernel/vmlinux.lds` 링커 스크립트에서 다음 섹션을 찾을 수 있습니다:

```
.init.data : AT(ADDR(.init.data) - 0xffffffff80000000) {
    ...
    ...
    ...
    ...
    __initcall_start = .;
    *(.initcallearly.init)
    __initcall0_start = .;
    *(.initcall0.init)
    *(.initcall0s.init)
    __initcall1_start = .;
    ...
    ...
}
```

이것에 익숙하지 않다면이 책의 특별 [부분](https://0xax.gitbooks.io/linux-insides/content/Misc/linux-misc-3.html)에서 [링커](https://en.wikipedia.org/wiki/Linker_%28computing%29)에 대해 더 많이 알 수 있습니다.

방금 살펴본 바와 같이 `do_initcall_level` 함수는 하나의 매개변수인 `initcall` 레벨을 취하고 다음 두 가지를 수행한다. 우선이 함수는 매개 변수를 포함 할 수있는 일반적인 커널 [command line](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)의 사본인 `initcall_command_line`을 분석한다. 에서 [커널/params.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/params.c) 소스 코드 파일에서 'parse_args` 기능 모듈은 각각의 레벨에 대한 `do_on_initcall` 함수를 호출:

```C
for (fn = initcall_levels[level]; fn < initcall_levels[level+1]; fn++)
		do_one_initcall(*fn);
```

`do_on_initcall`은 우리에게 중요한 역할을 합니다. 보시다시피,이 함수는 `initcall` 콜백 함수를 나타내는 하나의 매개 변수를 취하며 주어진 콜백의 호출을 수행합니다:

```C
int __init_or_module do_one_initcall(initcall_t fn)
{
	int count = preempt_count();
	int ret;
	char msgbuf[64];

	if (initcall_blacklisted(fn))
		return -EPERM;

	if (initcall_debug)
		ret = do_one_initcall_debug(fn);
	else
		ret = fn();

	msgbuf[0] = 0;

	if (preempt_count() != count) {
		sprintf(msgbuf, "preemption imbalance ");
		preempt_count_set(count);
	}
	if (irqs_disabled()) {
		strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
		local_irq_enable();
	}
	WARN(msgbuf[0], "initcall %pF returned with %s\n", fn, msgbuf);

	return ret;
}
```

`do_on_initcall` 함수의 기능을 이해하려고 노력하자. 우선 [선점](https://en.wikipedia.org/wiki/Preemption_%28computing%29) 카운터를 늘려 나중에 불균형이 없는지 확인할 수 있습니다. 이 단계 후에 우리는 `initcall_backlist` 함수의 호출을 볼 수 있습니다. 블랙리스트에있는 `initcalls`를 저장하는 `blacklisted_initcalls`목록을 살펴보고 주어진 `initcall`이 이 목록에 있으면 해제합니다:

```C
list_for_each_entry(entry, &blacklisted_initcalls, next) {
	if (!strcmp(fn_name, entry->buf)) {
		pr_debug("initcall %s blacklisted\n", fn_name);
		kfree(fn_name);
		return true;
	}
}
```

블랙리스트에있는 `initcalls`는 `blacklisted_initcalls` 목록에 저장되며 이 목록은 Linux 커널 명령 행에서 초기 Linux 커널 초기화 중에 채워집니다.

블랙리스트에있는 `initcalls`가 처리 된 후 코드의 다음 부분은 `initcall`을 직접 호출합니다:

```C
if (initcall_debug)
	ret = do_one_initcall_debug(fn);
else
	ret = fn();
```

`initcall_debug` 변수의 값에 따라 `do_one_initcall_debug` 함수는 `initcall`을 호출하거나이 함수는 `fn()`을 통해 직접 수행합니다. `initcall_debug` 변수는 [same](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일에 정의되어 있습니다:

```C
bool initcall_debug;
```

커널[log buffer](https://en.wikipedia.org/wiki/Dmesg)에 일부 정보를 인쇄하는 기능을 제공합니다. 변수의 값은 커널 명령에서 `initcall_debug` 매개 변수를 통해 설정할 수 있습니다. Linux 커널 명령 행의 [documentation](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)에서 읽을 수있는 것 처럼:

```
initcall_debug	[KNL] Trace initcalls as they are executed.  Useful
                      for working out where the kernel is dying during
                      startup.
```

그리고 그것은 사실입니다. `do_one_initcall_debug` 함수의 구현을 살펴보면,이 함수가 `do_one_initcall` 함수와 동일하거나 `do_one_initcall_debug` 함수가 주어진 `initcall`을 호출하고 일부 정보 (예 : 현재 실행중인 작업의 [pid](https://en.wikipedia.org/wiki/Process_identifier), `initcall`의 실행 기간 등):

```C
static int __init_or_module do_one_initcall_debug(initcall_t fn)
{
	ktime_t calltime, delta, rettime;
	unsigned long long duration;
	int ret;

	printk(KERN_DEBUG "calling  %pF @ %i\n", fn, task_pid_nr(current));
	calltime = ktime_get();
	ret = fn();
	rettime = ktime_get();
	delta = ktime_sub(rettime, calltime);
	duration = (unsigned long long) ktime_to_ns(delta) >> 10;
	printk(KERN_DEBUG "initcall %pF returned %d after %lld usecs\n",
		 fn, ret, duration);

	return ret;
}
```

`initcall`은 `do_one_initcall` 또는 `do_one_initcall_debug` 함수 중 하나에 의해 호출되었으므로, `do_one_initcall` 함수의 끝에 두 가지 검사가 있을 수 있습니다. 첫 번째는 실행 된 initcall 내부에서 가능한 `__preempt_count_add` 및 `__preempt_count_sub` 호출의 양을 확인하고 이 값이 선점 형 카운터의 이전 값과 같지 않으면 메시지 버퍼에 `preemption imbalance` 문자열을 추가합니다. 선점 카운터의 올바른 값을 설정하십시오:

```C
if (preempt_count() != count) {
	sprintf(msgbuf, "preemption imbalance ");
	preempt_count_set(count);
}
```

나중에 이 오류 문자열이 인쇄됩니다. 마지막으로 로컬 [IRQs](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)의 상태를 확인하고 비활성화 된 경우 메시지 버퍼에 ``비활성화 된 인터럽트` 문자열을 추가하고 활성화합니다. `initcall`에 의해 `IRQs`가 비활성화되고 다시 활성화되지 않은 상태를 방지하기 위해 현재 프로세서에 대한 `IRQs`:

```C
if (irqs_disabled()) {
	strlcat(msgbuf, "disabled interrupts ", sizeof(msgbuf));
	local_irq_enable();
}
```

그게 다입니다. 이런 식으로 Linux 커널은 많은 서브 시스템을 올바른 순서로 초기화합니다. 이제부터 리눅스 커널에서 `initcall` 메커니즘이 무엇인지 알게되었다. 이 부분에서는 `initcall` 메커니즘의 주요 부분을 다루었지만 몇 가지 중요한 개념을 남겼습니다. 이 개념들을 간단히 살펴 봅시다.

우선, 우리는 한 단계의 `initcalls`를 놓쳤습니다. 이것이 바로 `rootfs initcalls`입니다. `rootfs_initcall`의 정의는 [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) 헤더 파일에서 모두 찾을 수 있습니다. 이 부분에서 본 유사한 매크로:

```C
#define rootfs_initcall(fn)		__define_initcall(fn, rootfs)
```

매크로 이름에서 알 수 있듯이 주요 목적은 [rootfs](https://en.wikipedia.org/wiki/Initramfs)와 관련된 콜백을 저장하는 것입니다. 이 목표 외에, 장치 관련 항목이 초기화되지 않은 경우에만 파일 시스템 수준과 관련된 초기화 후 다른 항목을 초기화하는 것이 유용 할 수 있습니다. 예를 들어, [init/initramfs.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/initramfs.c) 소스 코드 파일의 `populate_rootfs` 함수에서 발생한 [initramfs](https://en.wikipedia.org/wiki/Initramfs)의 압축 해제:

```C
rootfs_initcall(populate_rootfs);
```

이 곳에서 우리는 익숙한 결과를 볼 수 있습니다:

```
[    0.199960] Unpacking initramfs...
```

`rootfs_initcall` 레벨 외에도 추가적인 `console_initcall`,`security_initcall` 및 기타 보조 `initcall` 레벨이 있습니다. 마지막으로 놓친 것은 `* _initcall_sync` 레벨 세트입니다. 이 부분에서 보았던 거의 모든 `* _initcall` 매크로는 `_sync` 접두어와 매크로를 동반합니다:

```C
#define core_initcall_sync(fn)		__define_initcall(fn, 1s)
#define postcore_initcall_sync(fn)	__define_initcall(fn, 2s)
#define arch_initcall_sync(fn)		__define_initcall(fn, 3s)
#define subsys_initcall_sync(fn)	__define_initcall(fn, 4s)
#define fs_initcall_sync(fn)		__define_initcall(fn, 5s)
#define device_initcall_sync(fn)	__define_initcall(fn, 6s)
#define late_initcall_sync(fn)		__define_initcall(fn, 7s)
```

이러한 추가 레벨의 주요 목표는 특정 레벨에 대한 모든 모듈 관련 초기화 루틴이 완료 될 때까지 기다리는 것입니다.

그게 답니다.

결론
--------------------------------------------------------------------------------

이 부분에서 우리는 리눅스 커널이 초기화하는 동안 리눅스 커널의 현재 상태에 의존하는 함수를 호출 할 수있는 중요한 리눅스 커널 메커니즘을 보았다.

궁금한 점이나 제안이 있으시면 트위터 [0xAX](https://twitter.com/0xAX)에 저를 핑(ping)하거나 [이메일](anotherworldofworld@gmail.com)로 보내거나 [문제](https://github.com/0xAX/linux-insides/issues/new).

** 영어는 제 모국어가 아니여서 불편을 끼쳐 드려 죄송합니다. 실수를 발견하면 PR을 [linux-insides](https://github.com/0xAX/linux-insides)로 보내주십시오. **.

링크
--------------------------------------------------------------------------------

* [callback](https://en.wikipedia.org/wiki/Callback_%28computer_programming%29)
* [debugfs](https://en.wikipedia.org/wiki/Debugfs)
* [integer type](https://en.wikipedia.org/wiki/Integer)
* [symbols concatenation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)
* [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [Link time optimization](https://gcc.gnu.org/wiki/LinkTimeOptimization)
* [Introduction to linkers](https://0xax.gitbooks.io/linux-insides/content/Misc/linux-misc-3.html)
* [Linux kernel command line](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)
* [Process identifier](https://en.wikipedia.org/wiki/Process_identifier)
* [IRQs](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [rootfs](https://en.wikipedia.org/wiki/Initramfs)
* [previous part](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
