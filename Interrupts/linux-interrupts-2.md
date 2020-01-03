인터럽트와 인터럽트 처리. Part 2.
================================================================================

Linux 커널에서 인터럽트 및 예외 처리에 대해 자세히 알아보기
--------------------------------------------------------------------------------

우리는 [이전 부분](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-1.html)에서 인터럽트와 예외 처리에 대한 이론을 보았습니다. 우리는 여기에서 리눅스 커널 소스 코드의 인터럽트와 예외를 공부하기 시작할 것이다. 이전 부분에서는 대부분 이론적인 측면을 설명했고 이 부분에서는 Linux 커널 소스 코드를 직접 살펴보기 시작할 겁니다. 다른 챕터에서 했던 것처럼 처음부터 바로 시작합니다. 가장 초기의 [코드 라인](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S#L292)에서 Linux 커널 소스 코드를 볼 수 없습니다. [Linux 커널 부팅 프로세스](https://0xax.gitbooks.io/linux-insides/content/Booting/index.html)장의 예제이지만 인터럽트 및 예외와 관련된 가장 빠른 코드부터 시작합니다. 이 부분에서는 Linux 커널 소스 코드에서 찾을 수 있는 모든 인터럽트 및 예외 관련 항목을 살펴 봅니다.

이전 부분을 읽었다면 인터럽트와 관련된 리눅스 커널 `x86_64` 아키텍처 특정 소스 코드의 가장 빠른 위치는 [arch/x86/boot/pm.c]( https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c) 소스 코드 파일이며 [Interrupt Descriptor Table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)의 첫 번째 설정을 나타냅니다. `setup_idt`를 호출하여 `go_to_protected_mode` 함수에서 [보호 모드](http://en.wikipedia.org/wiki/Protected_mode)로 전환하기 직전에 발생합니다.

```C
void go_to_protected_mode(void)
{
	...
	setup_idt();
	...
}
```

`setup_idt` 함수는 `go_to_protected_mode` 함수와 동일한 소스 코드 파일에 정의되어 있으며 `NULL`인터럽트 설명자 테이블의 주소만 로드합니다.

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

여기서 `gdt_ptr`은 `Global Descriptor Table`의 기본 주소를 포함해야하는 특수 48 비트 `GDTR`레지스터를 나타냅니다:

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

물론 우리의 경우 `gdt_ptr`은 `GDTR` 레지스터를 나타내지 않고 `Interrupt Descriptor Table`을 설정 한 이후 `IDTR`을 나타냅니다. `idt_ptr`구조체가 리눅스 커널 소스 코드에 있다면 `gdt_ptr`과 동일하지만 이름이 다르기 때문에 찾을 수 없습니다. 따라서 이름만 다른 두 개의 유사한 구조체를 갖는 것은 의미가 없습니다. 이 시점에서 인터럽트 나 예외를 처리하기에는 너무 이르기 때문에 `인터럽트 디스크립터 테이블`에 엔트리를 채우지 않습니다. 그렇기 때문에 우리는 `IDT`를 `NULL`로 채웁니다.

[Interrupt Descriptor Table](http://en.wikipedia.org/wiki/Interrupt_descriptor_table), [Global Descriptor Table](http://en.wikipedia.org/wiki/GDT) 및 기타 사항을 설정 한 후 -[arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pmjump.S)의 [보호 모드](http://en.wikipedia.org/wiki/Protected_mode)로 이동하세요. 보호 모드로의 전환을 설명하는 [부분](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html)에서 자세한 내용을 읽을 수 있습니다.

우리는 이미 초기 부분에서 보호 모드로의 진입은 `boot_params.hdr.code32_start`에 있으며 보호 모드의 시작과 `boot_params`는 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c)끝에서 `protected_mode_jump`에 전달됨을 알 수 있습니다. [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c):

```C
protected_mode_jump(boot_params.hdr.code32_start,
			    (u32)&boot_params + (ds() << 4));
```

`protected_mode_jump`는 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pmjump.S)에 정의되어 있으며 [conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)을 호출하는 [8086](http://en.wikipedia.org/wiki/Intel_8086)을 사용하여 `ax`와 `dx`레지스터에 있는 이 두 가지 매개변수를 얻습니다:

```assembly
GLOBAL(protected_mode_jump)
	...
	...
	...
	.byte	0x66, 0xea		# ljmpl opcode
2:	.long	in_pm32			# offset
	.word	__BOOT_CS		# segment
...
...
...
ENDPROC(protected_mode_jump)
```

여기서 `in_pm32`는 32 비트 진입점으로의 점프를 포함 합니다:

```assembly
GLOBAL(in_pm32)
	...
	...
	jmpl	*%eax // %eax contains address of the `startup_32`
	...
	...
ENDPROC(in_pm32)
```

알다시피 32 비트 진입 점은 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)에 있습니다. 어셈블리 파일이지만 이름에 `_64`가 포함되어 있습니다. `arch/x86/boot/compressed` 디렉토리에서 비슷한 두 파일을 볼 수 있습니다:

* `arch/x86/boot/compressed/head_32.S`.
* `arch/x86/boot/compressed/head_64.S`;

하지만 32 비트 모드 진입점은 이 경우 두 번째 파일입니다. 첫 번째 파일은 `x86_64`용으로 컴파일되지도 않았습니다. [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/Makefile)을 봅시다:

```
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
...
...
```

여기서 `head_ *`는 아키텍쳐에 의존하는 `$(BITS)` 변수에 의존한다는 것을 알 수 있습니다. [arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)에서 찾을 수 있습니다:

```
ifeq ($(CONFIG_X86_32),y)
...
	BITS := 32
else
	BITS := 64
	...
endif
```

이제 우리가 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)에서 `startup_32`로 넘어 갔을 때 여기서 인터럽트 처리와 관련된 것을 찾을 수 없습니다. `startup_32`에는 [롱 모드](http://en.wikipedia.org/wiki/Long_mode)로 전환하기 전에 준비하고 직접 들어가는 코드가 들어 있습니다. `롱 모드` 항목은 `startup_64`에 있으며 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/misc.c)의 `decompress_kernel`에서 발생하는 [kernel 압축](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-5.html) 전에 준비합니다. 커널이 압축 해제 된 후 [arch /x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S)에서 `startup_64`로 이동합니다. `startup_64`에서 우리는 신원 매핑 된 페이지를 만들기 시작합니다. 신원 매핑 된 페이지를 구축하고 [NX](http://en.wikipedia.org/wiki/NX_bit) 비트를 확인하고 `확장 기능 활성화 레지스터`(링크 참조)를 설정 한 다음 `lgdt` 명령으로 초기`글로벌 디스크립터 테이블`을 업데이트 한 후 설정해야합니다. `gs`는 다음 코드로 등록합니다:

```assembly
movl	$MSR_GS_BASE,%ecx
movl	initial_gs(%rip),%eax
movl	initial_gs+4(%rip),%edx
wrmsr
```

우리는 이미 이 코드를 [이전 부분](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-1.html)에서 보았습니다. 우선 마지막 `wrmsr`명령에 주의를 기울이십시오. 이 명령어는`edx : eax` 레지스터의 데이터를 `ecx` 레지스터에 의해 지정된 [모델 특정 레지스터](http://en.wikipedia.org/wiki/Model-specific_register)에 씁니다. `ecx`에 [arch/x86/include/uapi/asm/msr-index.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/msr-index.h)에 선언 된`$ MSR_GS_BASE`가 포함되어 있음을 알 수 있습니다. 다음과 같습니다:

```C
#define MSR_GS_BASE             0xc0000101
```

이것으로부터 우리는 `MSR_GS_BASE`가 `model specific register`의 수를 정의한다는 것을 이해할 수 있습니다. 레지스터 `cs`, `ds`, `es` 및 `ss`는 64 비트 모드에서 사용되지 않으므로 해당 필드는 무시됩니다. 그러나 `fs`와 `gs` 레지스터를 통해 메모리에 접근 할 수 있습니다. 모델 특정 레지스터는 이 세그먼트 레지스터의 숨겨진 부분에 `후문`을 제공하며 `fs` 및 `gs`로 주소가 지정된 세그먼트 레지스터에 64 비트 기본 주소를 사용할 수 있습니다. 따라서 `MSR_GS_BASE`는 숨겨진 부분이며 이 부분은 `GS.base`필드에 매핑됩니다. `initial_gs`를 봅시다:

```assembly
GLOBAL(initial_gs)
	.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

`irq_stack_union` 심볼을 `INIT_PER_CPU_VAR`매크로에 전달합니다.이 매크로는 `init_per_cpu__ `접두사와 주어진 심볼을 연결합니다. 이 경우 `init_per_cpu__irq_stack_union` 심볼이 나타납니다. [링커](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vmlinux.lds.S) 스크립트를 살펴 보겠습니다. 우리는 다음과 같은 정의를 볼 수 있습니다:

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(irq_stack_union);
```

`init_per_cpu__irq_stack_union`의 주소는 `irq_stack_union + __per_cpu_load`가 됩니다. 이제 우리는 `init_per_cpu__irq_stack_union`과 `__per_cpu_load`의 의미를 이해해야합니다. 첫 번째 `irq_stack_union`은 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h)에 정의되어 있습니다. `DECLARE_INIT_PER_CPU` 매크로를 사용하여 `init_per_cpu_var` 매크로를 호출하도록 확장합니다:

```C
DECLARE_INIT_PER_CPU(irq_stack_union);

#define DECLARE_INIT_PER_CPU(var) \
       extern typeof(per_cpu_var(var)) init_per_cpu_var(var)

#define init_per_cpu_var(var)  init_per_cpu__##var
```

모든 매크로를 확장하면 `INIT_PER_CPU`매크로를 확장 한 후와 동일한 `init_per_cpu__irq_stack_union`이 표시되지만 이는 단순한 심볼이 아니라 변수라는 것을 알 수 있습니다. `typeof(per_cpu_var (var))`식을 봅시다. 우리의 `var`은 `irq_stack_union`이고 `per_cpu_var` 매크로는 [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d93/93/973/arch/x86/ include/asm/percpu.h)에 정의되어 있습니다:

```C
#define PER_CPU_VAR(var)        %__percpu_seg:var
```

where:

```C
#ifdef CONFIG_X86_64
    #define __percpu_seg gs
endif
```

그래서 우리는 `gs : irq_stack_union`에 접근하고 타입 `irq_union`을 얻습니다. 자, 우리는 첫 번째 변수를 정의하고 그것의 주소를 알았습니다. 이제 두 번째 `__per_cpu_load`심볼을 보도록합시다. 이 심볼 뒤에 위치한 두 개의 `per-cpu`변수가 있습니다. `__per_cpu_load`는 [include/asm-generic/sections.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic-sections.h)에 정의되어 있습니다:

```C
extern char __per_cpu_load[], __per_cpu_start[], __per_cpu_end[];
```

그리고 데이터 영역에서 `per-cpu`변수의 기본 주소를 제시했습니다. 따라서 우리는 `irq_stack_union`, `__per_cpu_load`의 주소를 알고 있으며 `init_per_cpu__irq_stack_union`은 `__per_cpu_load` 바로 뒤에 위치해야 한다는 것을 알고 있습니다. 그리고 우리는 [System.map](http://en.wikipedia.org/wiki/System.map)에서 볼 수 있습니다:

```
...
...
...
ffffffff819ed000 D __init_begin
ffffffff819ed000 D __per_cpu_load
ffffffff819ed000 A init_per_cpu__irq_stack_union
...
...
...
```

이제 우리는 `initial_gs`에 대해 알고 있으므로 코드를 보자:

```assembly
movl	$MSR_GS_BASE,%ecx
movl	initial_gs(%rip),%eax
movl	initial_gs+4(%rip),%edx
wrmsr
```

여기서 우리는 `MSR_GS_BASE`로 모델 특정 레지스터를 지정하고 `initial_gs`의 64 비트 주소를 `edx : eax` 쌍에 넣고 `gs`레지스터를 인터럽트 스택의 맨 아래에있는 `init_per_cpu__irq_stack_union`의 기본 주소로 채우기 위해  `wrmsr` 명령을 실행합니다. 그런 다음 [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head64.c)에서 `x86_64_start_kernel`의 C 코드로 건너 뜁니다. `x86_64_start_kernel` 함수에서 우리는 일반 및 아키텍처 독립적 인 커널 코드로 넘어 가기 전에 마지막 준비를 수행하고 이러한 준비 중 하나는 초기 `Interrupt Descriptor Table`을 인터럽트 처리기 항목 또는 `early_idt_handlers`로 채우는 것입니다. [Early interrupt and exception handling](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html)에 대한 부분을 읽은 것을 기억하고 기억할 수 있습니다 다음 코드:

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handlers[i]);

load_idt((const struct desc_ptr *)&idt_descr);
```

Linux 커널 버전이 3.18 일 때 `초기 인터럽트와 예외 처리` 부분을 썼습니다. 하지만 현재 리눅스 커널의 실제 버전은 `4.1.0-rc6 +`이고 `Andy Lutomirski`는 [패치](https://lkml.org/lkml/2015/6/2/106)를 보냈고 곧 `early_idt_handlers`의 동작을 변경하는 메인 라인 커널에 있어야 합니다. ** 참고** 이 부분을 작성하는 동안 [패치](https://github.com/torvalds/linux/commit/425be5679fd292a3c36cb1fe423086708a99f11a)는 이미 Linux 커널 소스 코드로 설정되었습니다. 살펴 봅시다. 다음과 같습니다:

```C
for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
	set_intr_gate(i, early_idt_handler_array[i]);

load_idt((const struct desc_ptr *)&idt_descr);
```

보시다시피 인터럽트 처리기 진입점의 배열 이름에는 하나의 차이점만 있습니다. 이제 `early_idt_handler_arry`입니다:

```C
extern const char early_idt_handler_array[NUM_EXCEPTION_VECTORS][EARLY_IDT_HANDLER_SIZE];
```

where `NUM_EXCEPTION_VECTORS` and `EARLY_IDT_HANDLER_SIZE` are defined as:

```C
#define NUM_EXCEPTION_VECTORS 32
#define EARLY_IDT_HANDLER_SIZE 9
```

따라서, `early_idt_handler_array`는 인터럽트 핸들러 엔트리 포인트의 배열이며 9 바이트마다 하나의 엔트리 포인트를 포함합니다. 이전 `early_idt_handlers`는 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S)에 정의되어 있음을 기억할 수 있습니다. `early_idt_handler_array`는 동일한 소스 코드 파일에도 정의되어 있습니다:

```assembly
ENTRY(early_idt_handler_array)
...
...
...
ENDPROC(early_idt_handler_common)
```

`early_idt_handler_arry`를 `.rept NUM_EXCEPTION_VECTORS`로 채우고 `early_make_pgtable` 인터럽트 핸들러의 엔트리를 포함합니다(구현에 대한 자세한 내용은 [초기 인터럽트와 예외 처리](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html)에서 읽을 수 있습니다. 이제 우리는 `x86_64` 아키텍처 특정 코드의 끝 부분에 도달했으며 다음 부분은 일반적인 커널 코드입니다. 물론 우리는 `setup_arch` 함수와 다른 곳에서 아키텍처 특정 코드로 돌아갈 것이라는 것을 이미 알 수 있지만 이것은 `x86_64` 초기 코드의 끝입니다.

인터럽트 스택을 위한 스택 canary 설정
-------------------------------------------------------------------------------

[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에서 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S) 가장 큰 `start_kernel` 기능 다음에 멈춥니다. 리눅스 커널 초기화 과정에 대한 이전의 [챕터](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)를 읽었다면 기억해야 합니다. 이 함수는 커널이 [pid](https://en.wikipedia.org/wiki/Process_identifier)-`1`로 먼저 `init` 프로세스를 시작하기 전에 모든 초기화 작업을 수행합니다. 인터럽트 및 예외 처리와 관련된 첫 번째 것은 `boot_init_stack_canary`함수의 호출입니다.

이 함수는 인터럽트 스택 오버 플로우를 보호하기 위해 [canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) 값을 설정합니다. 이전 부분에서 `boot_init_stack_canary`의 구현에 대해 약간의 세부 사항을 이미 보았으므로 이제 자세히 살펴 보겠습니다. 이 기능의 구현은 [arch/x86/include/asm/stackprotector.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/stackprotector.h)에서 찾을 수 있고 `CONFIG_CC_STACKPROTECTOR` 커널 구성 옵션에 따라 다릅니다. 이 옵션을 설정하지 않으면 이 기능은 아무 작업도 수행하지 않습니다:

```C
#ifdef CONFIG_CC_STACKPROTECTOR
...
...
...
#else
static inline void boot_init_stack_canary(void)
{
}
#endif
```

`CONFIG_CC_STACKPROTECTOR` 커널 설정 옵션이 설정되면,`boot_init_stack_canary` 기능은 [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)를 나타내는 검사 통계 `irq_stack_union`에서 시작됩니다 인터럽트 스택은`stack_canary` 값에서 40 바이트와 같은 오프셋을 갖습니다:

```C
#ifdef CONFIG_X86_64
        BUILD_BUG_ON(offsetof(union irq_stack_union, stack_canary) != 40);
#endif
```

앞의 [부분](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-1.html)에서 읽을 수 있듯이 다음 조합으로 표시되는 `irq_stack_union`:

```C
union irq_stack_union {
	char irq_stack[IRQ_STACK_SIZE];

    struct {
		char gs_base[40];
		unsigned long stack_canary;
	};
};
```

[arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h)에 정의되어 있습니다. [C](http://en.wikipedia.org/wiki/C_%28programming_language%29) 프로그래밍 언어의 [공용체](http://en.wikipedia.org/wiki/Union_type)은 하나의 필드만 메모리에 저장하는 데이터 구조체 입니다. 여기서 구조체에는 첫 번째 필드 인 `gs_base`가 있으며 이는 40 바이트 크기이며 `irq_stack`의 맨 아래를 나타냅니다. 따라서 이 후에 `BUILD_BUG_ON` 매크로를 사용한 검사가 성공적으로 종료됩니다. (`BUILD_BUG_ON` 매크로 에 관심이 있다면 리눅스 커널 초기화 [프로세스](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)에 대한 첫 번째 부분을 읽을 수 있습니다).

그런 다음 난수와 [타임 스탬프 카운터](http://en.wikipedia.org/wiki/Time_Stamp_Counter)를 기반으로 새로운 `Canary`값을 계산합니다:

```C
get_random_bytes(&canary, sizeof(canary));
tsc = __native_read_tsc();
canary += tsc + (tsc << 32UL);
```

`this_cpu_write` 매크로를 사용하여 `canary` 값을 `irq_stack_union`에 씁니다:

```C
this_cpu_write(irq_stack_union.stack_canary, canary);
```

`this_cpu_ *` 작업에 대한 자세한 내용은 [Linux 커널 설명서](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt)에서 읽을 수 있습니다.

로컬 인터럽트 비활성화/활성화
--------------------------------------------------------------------------------

`canary`값을 인터럽트 스택으로 설정 한 후 인터럽트 및 인터럽트 처리와 관련된 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)의 다음 단계는 `local_irq_disable` 매크로의 호출입니다.

이 매크로는 [include/linux/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqflags.h) 헤더 파일에 정의되어 있으며 이해할 수 있는 대로 비활성화 할 수 있습니다. 이 매크로를 호출하여 CPU를 인터럽트합니다. 구현을 살펴 봅시다. 우선 `CONFIG_TRACE_IRQFLAGS_SUPPORT` 커널 설정 옵션에 의존합니다:

```C
#ifdef CONFIG_TRACE_IRQFLAGS_SUPPORT
...
#define local_irq_disable() \
         do { raw_local_irq_disable(); trace_hardirqs_off(); } while (0)
...
#else
...
#define local_irq_disable()     do { raw_local_irq_disable(); } while (0)
...
#endif
```

그것들은 비슷하며 한 가지 차이점이 있습니다. `local_irq_disable` 매크로는 `CONFIG_TRACE_IRQFLAGS_SUPPORT`가 활성화 된 경우 `trace_hardirqs_off`의 호출을 포함합니다. [lockdep](http://lwn.net/Articles/321663/) 서브 시스템에는 `hardirq` 및 `softirq` 상태를 추적하기 위한 `irq-flags tracing` 기능이 있습니다. 우리의 경우`lockdep` 서브 시스템은 시스템에서 발생하는 하드 / 소프트 irqs on / off 이벤트에 대한 흥미로운 정보를 제공 할 수 있습니다. [kernel/locking/lockdep.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/locking/lockdep.c)에 정의 된`trace_hardirqs_off` 함수:

```C
void trace_hardirqs_off(void)
{
         trace_hardirqs_off_caller(CALLER_ADDR0);
}
EXPORT_SYMBOL(trace_hardirqs_off);
```

`trace_hardirqs_off_caller` 함수만 호출하면 됩니다. `trace_hardirqs_off_caller`는 현재 프로세스의 `hardirqs_enabled` 필드를 점검하고 `local_irq_disable`의 호출이 중복 된 경우 `redundant_hardirqs_off`를 늘리거나 그렇지 않은 경우 `hardirqs_off_events`를 증가시킵니다. 이 두 필드와 다른 `lockdep`통계 관련 필드는 [kernel/locking/lockdep_insides.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/locking/lockdep_insides.h)에 정의되어 있습니다. `lockdep_stats` 구조에 있습니다:

```C
struct lockdep_stats {
...
...
...
int     softirqs_off_events;
int     redundant_softirqs_off;
...
...
...
}
```

`CONFIG_DEBUG_LOCKDEP` 커널 설정 옵션을 설정하면 `lockdep_stats_debug_show` 함수는 모든 추적 정보를`/proc/lockdep`에 씁니다:

```C
static void lockdep_stats_debug_show(struct seq_file *m)
{
#ifdef CONFIG_DEBUG_LOCKDEP
	unsigned long long hi1 = debug_atomic_read(hardirqs_on_events),
	                         hi2 = debug_atomic_read(hardirqs_off_events),
							 hr1 = debug_atomic_read(redundant_hardirqs_on),
    ...
	...
	...
    seq_printf(m, " hardirq on events:             %11llu\n", hi1);
    seq_printf(m, " hardirq off events:            %11llu\n", hi2);
    seq_printf(m, " redundant hardirq ons:         %11llu\n", hr1);
#endif
}
```

그리고 그 결과를 볼 수 있습니다:

```
$ sudo cat /proc/lockdep
 hardirq on events:             12838248974
 hardirq off events:            12838248979
 redundant hardirq ons:               67792
 redundant hardirq offs:         3836339146
 softirq on events:                38002159
 softirq off events:               38002187
 redundant softirq ons:                   0
 redundant softirq offs:                  0
```

자, 이제 우리는 추적에 대해 조금 알고 있지만, 더 많은 정보는 `lockdep`와 `tracing`에 대한 별도의 부분에 있습니다. `local_disable_irq` 매크로는 `raw_local_irq_disable`과 동일한 부분을 가지고 있음을 알 수 있습니다. 이 매크로는 [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irqflags.h)에 정의되어 있으며 호출로 확장됩니다:

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

그리고 `cli` 명령어는 프로세서가 인터럽트 나 예외를 처리하는 능력을 결정하는 [IF](http://en.wikipedia.org/wiki/Interrupt_flag) 플래그를 지운다는 것을 이미 기억해야합니다. 이미 알 수 있듯이 `local_irq_disable` 외에도 역 매크로 인 `local_irq_enable`이 있습니다. 이 매크로는 동일한 추적 메커니즘을 가지고 있으며 `local_irq_enable`과 매우 유사하지만 이름에서 알 수 있듯이 `sti` 명령으로 인터럽트를 활성화합니다:

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```

이제 `local_irq_disable`과 `local_irq_enable`의 작동 방식을 알았습니다. `local_irq_disable` 매크로의 첫 번째 호출이지만 Linux 커널 소스 코드에서 이러한 매크로를 여러 번 만나게됩니다. 그러나 지금 우리는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)의 `start_kernel` 기능에 있으며 `local`을 비활성화했습니다. 인터럽트. 왜 현지에, 왜 그렇게 했습니까? 이전에는 커널이 모든 프로세서에서 인터럽트를 비활성화하는 방법을 제공했으며 이를 `cli`라고했습니다. 이 기능은 [삭제](https://lwn.net/Articles/291956/)되었으며 현재 로컬 프로세서에서 인터럽트를 비활성화 또는 활성화하기위한 `local_irq_ {enabled, disable}`이 있습니다. `local_irq_disable` 매크로로 인터럽트를 비활성화 한 후 다음을 설정합니다:

```C
early_boot_irqs_disabled = true;
```

[include/linux/kernel.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/kernel.h)에 정의 된 `early_boot_irqs_disabled` 변수:

```C
extern bool early_boot_irqs_disabled;
```

다른 장소에서 사용됩니다. 예를 들어 인터럽트가 비활성화 될 때 가능한 교착 상태를 확인하기 위해 [kernel/smp.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/smp.c)의 `smp_call_function_many` 함수에서 사용되었습니다:

```C
WARN_ON_ONCE(cpu_online(this_cpu) && irqs_disabled()
                     && !oops_in_progress && !early_boot_irqs_disabled);
```

커널 초기화 중 초기 트랩 초기화
--------------------------------------------------------------------------------

`local_disable_irq` 이후의 다음 함수는 `boot_cpu_init` 및 `page_address_init`이지만 인터럽트 및 예외와 관련이 없습니다 (이 기능에 대한 자세한 내용은 Linux 커널 [초기화 프로세스](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html) 장에서 읽을 수 있습니다). 다음은`setup_arch` 함수입니다. [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel.setup.c) 소스 코드 파일에 있는 이 기능을 기억하고 많은 다른 아키텍처에 의존하는 [stuff](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)를 초기화합니다. `setup_arch`에서 볼 수 있는 첫 번째 인터럽트 관련 함수는 - early_trap_init 함수입니다. 이 함수는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)에 정의되어 있으며 `인터럽트 디스크립터 테이블`을 몇 개의 항목으로 채 웁니다:

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
#ifdef CONFIG_X86_32
        set_intr_gate(X86_TRAP_PF, page_fault);
#endif
        load_idt(&idt_descr);
}
```

여기서 우리는 세 가지 다른 함수의 호출을 볼 수 있습니다:

* `set_intr_gate_ist`
* `set_system_intr_gate_ist`
* `set_intr_gate`

[arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h) 및 비슷한 일을 하지만 동일하지는 않습니다. 첫 번째 `set_intr_gate_ist` 함수는 `IDT`에 새로운 인터럽트 게이트를 삽입합니다. 구현에 대해 살펴 봅시다:

```C
static inline void set_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS);
}
```

우선 인터럽트의 [벡터 번호](http://en.wikipedia.org/wiki/Interrupt_vector_table) 인 `n`이 `0xff` 또는 255보다 크지 않은 것을 확인할 수 있습니다. [이전 부분](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-1.html)에서 인터럽트의 벡터 번호는 `0`과 `255`. 다음 단계에서는 주어진 인터럽트 게이트를 IDT 테이블에 설정하는 `_set_gate` 함수의 호출을 볼 수 있습니다:

```C
static inline void _set_gate(int gate, unsigned type, void *addr,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate_desc s;

        pack_gate(&s, type, (unsigned long)addr, dpl, ist, seg);
        write_idt_entry(idt_table, gate, &s);
        write_trace_idt_entry(gate, &s);
}
```

여기서 우리는 `gate_desc` 구조로 표현 된 깨끗한 `IDT` 엔트리를 취해이를 기본 주소와 한계로 채우는 `pack_gate` 함수에서 시작합니다. [인터럽트 스택 테이블](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks), [Privilege level](http://en.wikipedia.org/wiki/Privilege_level), 다음 값 중 하나 일 수 있는 인터럽트 유형:

* `GATE_INTERRUPT`
* `GATE_TRAP`
* `GATE_CALL`
* `GATE_TASK`

주어진 `IDT` 엔트리에 대한 현재 비트를 설정합니다:

```C
static inline void pack_gate(gate_desc *gate, unsigned type, unsigned long func,
                             unsigned dpl, unsigned ist, unsigned seg)
{
        gate->offset_low        = PTR_LOW(func);
        gate->segment           = __KERNEL_CS;
        gate->ist               = ist;
        gate->p                 = 1;
        gate->dpl               = dpl;
        gate->zero0             = 0;
        gate->zero1             = 0;
        gate->type              = type;
        gate->offset_middle     = PTR_MIDDLE(func);
        gate->offset_high       = PTR_HIGH(func);
}
```

그런 다음 우리는 `write_idt_entry` 매크로를 사용하여 채워진 인터럽트 게이트를 `IDT`에 쓰고 `native_write_idt_entry`로 확장하고 주어진 인덱스에 의해 인터럽트 게이트를 `idt_table` 테이블에 복사합니다:

```C
#define write_idt_entry(dt, entry, g)           native_write_idt_entry(dt, entry, g)

static inline void native_write_idt_entry(gate_desc *idt, int entry, const gate_desc *gate)
{
        memcpy(&idt[entry], gate, sizeof(*gate));
}
```

여기서 `idt_table`은 `gate_desc`의 배열입니다:

```C
extern gate_desc idt_table[];
```

그게 다입니다. 두 번째 `set_system_intr_gate_ist` 함수는 `set_intr_gate_ist`와 하나의 차이점만 있습니다:

```C
static inline void set_system_intr_gate_ist(int n, void *addr, unsigned ist)
{
        BUG_ON((unsigned)n > 0xFF);
        _set_gate(n, GATE_INTERRUPT, addr, 0x3, ist, __KERNEL_CS);
}
```

봤습니까? `_set_gate`의 네 번째 매개 변수를 보십시오. `0x3`입니다. `set_intr_gate`에서는 `0x0`입니다. 이 매개 변수는 `DPL` 또는 권한 수준을 나타냅니다. 또한 `0`이 가장 높은 권한 수준이고 `3`이 가장 낮다는 것을 알고 이제 `set_system_intr_gate_ist`,`set_intr_gate_ist`,`set_intr_gate`의 작동 방식을 알고 `early_trap_init` 함수로 돌아갈 수 있습니다. 다시 한번 살펴 봅시다:

```C
set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
```

`# DB` 인터럽트와 `int3`에 대해 두 개의 `IDT` 항목을 설정했습니다. 이 함수는 동일한 매개 변수 집합을 사용합니다:

* 인터럽트의 벡터 번호;
* 인터럽트 핸들러의 주소;
* 인터럽트 스택 테이블 인덱스.

그게 다야. 다음 부분에서 알게 될 인터럽트 및 핸들러에 대해 자세히 알아보십시오.

결론
--------------------------------------------------------------------------------

리눅스 커널에서의 인터럽트와 인터럽트 처리에 관한 두 번째 부분이 끝났습니다. 우리는 이전 부분에서 일부 이론을 보았고 현재 부분에서 인터럽트 및 예외 처리로 뛰어 들기 시작했습니다. 우리는 인터럽트와 관련된 Linux 커널 소스 코드의 가장 빠른 부분부터 시작했습니다. 다음 부분에서 우리는이 흥미로운 주제를 계속해서 살펴보고 인터럽트 처리 프로세스에 대해 더 많이 알게 될 것입니다.

질문이나 제안 사항이 있으면 의견을 보내거나 [트위터]https://twitter.com/0xAX)에 핑(Ping) 해주십시오.

**영어는 모국어가 아니면 죄송합니다. 실수를 발견하면 PR을 [linux-insides](https://github.com/0xAX/linux-insides)로 보내주십시오..**

링크
--------------------------------------------------------------------------------

* [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [List of x86 calling conventions](http://en.wikipedia.org/wiki/X86_calling_conventions#List_of_x86_calling_conventions)
* [8086](http://en.wikipedia.org/wiki/Intel_8086)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [Extended Feature Enable Register](http://en.wikipedia.org/wiki/Control_register#Additional_Control_registers_in_x86-64_series)
* [Model-specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Process identifier](https://en.wikipedia.org/wiki/Process_identifier)
* [lockdep](http://lwn.net/Articles/321663/)
* [irqflags tracing](https://www.kernel.org/doc/Documentation/irqflags-tracing.txt)
* [IF](http://en.wikipedia.org/wiki/Interrupt_flag)
* [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)
* [Union type](http://en.wikipedia.org/wiki/Union_type)
* [this_cpu_* operations](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt)
* [vector number](http://en.wikipedia.org/wiki/Interrupt_vector_table)
* [Interrupt Stack Table](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [Privilege level](http://en.wikipedia.org/wiki/Privilege_level)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-1.html)
