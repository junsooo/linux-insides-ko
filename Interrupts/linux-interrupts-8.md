인터럽트와 인터럽트 처리. Part 8.
================================================================================

IRQ의 초기가 아닐때의 초기화
--------------------------------------------------------------------------------

이것은 리눅스 커널에서 인터럽트와 인터럽트 처리 [챕터](https://junsoolee.gitbook.io/linux-insides-ko/summary/interrupts)의 8번째 파트이며 이전 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/interrupts/linux-interrupts-7)에서 우리는 외부 하드웨어 [인터럽트](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)를 알아보기 시작했습니다. [kernel/irq/irqdesc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/irqdesc.c) 소스 코드 파일에서 `early_irq_init` 함수의 구현을 살펴보고 이 함수에서 `irq_desc` 구조체의 초기화를 보았습니다. `irq_desc` 구조체([include/linux/irqdesc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irqdesc.h#L46)에 정의 됨)는 리눅스 커널의 인터럽트 관리 코드의 기초이며 인터럽트 디스크립터를 나타냅니다. 이 파트에서 우리는 외부 하드웨어 인터럽트와 관련이 있는 초기화에 대해 계속 알아볼 것입니다.

[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에서 `early_irq_init` 함수를 호출 한 직후에 우리는 `init_IRQ` 함수의 호출을 볼 수 있습니다. 이 기능은 아키텍처에 따라 다르며 [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c)에 정의되어 있습니다. `init_IRQ` 함수는 동일한 [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) 소스 코드 파일에 정의 된 `vector_irq` [percpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) 변수를 초기화합니다. :

```C
...
DEFINE_PER_CPU(vector_irq_t, vector_irq) = {
         [0 ... NR_VECTORS - 1] = -1,
};
...
```

그리고 인터럽트 벡터 번호의 `percpu`배열을 나타냅니다. `vector_irq_t`는 [arch/x86/include/asm/hw_irq.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/hw_irq.h)에 정의 되어 있고, 다음으로 확장합니다.:

```C
typedef int vector_irq_t[NR_VECTORS];
```

여기서 NR_VECTORS는 벡터 번호의 개수이라는 것을 이 챕터의 첫번째 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/interrupts/linux-interrupts-1)에서 알 수 있습니다. [x86_64](https://en.wikipedia.org/wiki/X86-64)의 경우 `256`입니다.

```C
#define NR_VECTORS                       256
```

이제 `init_IRQ` 함수의 시작에서 우리는 `legacy` 인터럽트의 벡터 번호로 `vector_irq` [percpu](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-1) 배열을 채 웁니다. :

```C
void __init init_IRQ(void)
{
	int i;

	for (i = 0; i < nr_legacy_irqs(); i++)
		per_cpu(vector_irq, 0)[IRQ0_VECTOR + i] = i;
...
...
...
}
```

이 `vector_irq`는 [arch/x86/kernel/irq.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irq.c)의 `do_IRQ` 함수에서 외부 하드웨어 인터럽트 처리의 첫 단계 동안 사용됩니다. :

```C
__visible unsigned int __irq_entry do_IRQ(struct pt_regs *regs)
{
	...
	...
	...
	irq = __this_cpu_read(vector_irq[vector]);

	if (!handle_irq(irq, regs)) {
		...
		...
		...
	}

	exiting_irq();
	...
	...
	return 1;
}
```

왜 `legacy` 가 여기에 있을까요? 실제로 모든 인터럽트는 최신 [IO-APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#I.2FO_APICs) 컨트롤러에 의해 처리됩니다. 그러나 [Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)와 같은 레거시 인터럽트 컨트롤러에 의한 이러한 인터럽트 (`0x30`에서`0x3f`)가 `I / O APIC`에 의해 처리되면 이 벡터 공간은 해제되고 재사용됩니다. 이 코드를 자세히 살펴 봅시다. 우선 [arch/x86/include/asm/i8259.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/i8259.h)에 정의 된 모든 `nr_legacy_irqs`는 `legacy_pic` 구조체에서 `nr_legacy_irqs` 필드를 반환합니다. :

```C
static inline int nr_legacy_irqs(void)
{
        return legacy_pic->nr_legacy_irqs;
}
```

이 구조는 동일한 헤더 파일에 정의되어 있으며 현대적이지 않은 프로그래밍 가능한 인터럽트 컨트롤러를 나타냅니다. :

```C
struct legacy_pic {
        int nr_legacy_irqs;
        struct irq_chip *chip;
        void (*mask)(unsigned int irq);
        void (*unmask)(unsigned int irq);
        void (*mask_all)(void);
        void (*restore_mask)(void);
        void (*init)(int auto_eoi);
        int (*irq_pending)(unsigned int irq);
        void (*make_irq)(unsigned int irq);
};
```
레거시 인터럽트의 실제 기본 최대 수는 [arc/x86/include/asm/irq_vectors.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irq_vectors.h)에 있는 `NR_IRQ_LEGACY` 매크로로 표현합니다. :

```C
#define NR_IRQS_LEGACY                    16
```

루프에서 우리는 `IRQ0_VECTOR + i` 인덱스에 의해 `per_cpu` 매크로를 사용하여 CPU 당 배열인 `vecto_irq` 에 접근하고, 그곳에 레거시 벡터 번호를 씁니다. [arch/x86/include/asm/irq_vectors.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irq_vectors.h) 헤더 파일에 정의 된 `IRQ0_VECTOR` 매크로는 `0x30`으로 확장됩니다. :

```C
#define FIRST_EXTERNAL_VECTOR           0x20

#define IRQ0_VECTOR                     ((FIRST_EXTERNAL_VECTOR + 16) & ~15)
```
왜 여기에 `0x30`이 있을까요? `0` 부터 `31`까지의 첫 32 개의 벡터 번호는 프로세서가 예약하고 아키텍처 기반의 예외 및 인터럽트 처리에 사용된다는 것을 이 챕터의 첫 번째 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/interrupts/linux-interrupts-1)에서 배웠습니다. `0x30`부터 `0x3f`까지의 벡터 번호는 [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) 용으로 예약되어 있습니다. 즉, `32`와 동일한 `IRQ0_VECTOR`의 `vector_irq`를 `IRQ0_VECTOR + 16` (`0x30` 이전)로 채운다는 것을 의미합니다.

[arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c) 소스 코드 파일의 `init_IRQ` 함수 끝에서 다음 함수의 호출을 볼 수 있습니다. :

```C
x86_init.irqs.intr_init();
```

리눅스 커널 초기화 과정에 대한 [챕터](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization)를 읽었다면 `x86_init` 구조체를 기억할 수 있을 것입니다. 이 구조체에는 플랫폼 설정과 관련된 기능(우리의 경우 `x86_64`)을 가리키는 몇 개의 파일이 포함되어 있습니다. 예를 들어 메모리 자원과 관련된 `resources`. [MultiProcessor Configuration Table](https://en.wikipedia.org/wiki/MultiProcessor_Specification) 테이블을 파싱하는 것과 관련된 'mpparse' 등이 있습니다. 우리가 볼 수 있듯, `x86_init`에는 세 개의 다음 필드를 포함하는 `irqs` 필드도 포함되어 있습니다. :

```C
struct x86_init_ops x86_init __initdata
{
	...
	...
	...
    .irqs = {
                .pre_vector_init        = init_ISA_irqs,
                .intr_init              = native_init_IRQ,
                .trap_init              = x86_init_noop,
	},
	...
	...
	...
}
```

이제 우리는 `native_init_IRQ`에 흥미를 가집니다. 우리가 알 수 있듯, `native_init_IRQ` 함수의 이름은 `native_` 접두사를 포함하는데, 이는이 함수가 아키텍처에 따라 다르다는 것을 의미합니다. 이 함수는 [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c)에 정의되어 있으며 [Local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#Integrated_local_APICs)의 일반 초기화 및 [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture) irq들의 초기화를 실행합니다. 앞으로 `native_init_IRQ` 함수의 구현을 살펴보고 거기서 발생하는 것들을 이해해봅시다. `native_init_IRQ` 함수는 다음 함수의 실행에서 시작합니다. :

```C
x86_init.irqs.pre_vector_init();
```

위에서 볼 수 있듯이 `pre_vector_init`는 동일한 [소스 코드](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irqinit.c) 파일에 정의 된 `init_ISA_irqs` 함수를 가리키며 함수 이름에서 알 수 있듯이 ISA 관련 인터럽트를 초기화합니다. `init_ISA_irqs` 함수는 `irq_chip` 형식을 갖는 `chip`변수의 정의로 시작됩니다.

```C
void __init init_ISA_irqs(void)
{
	struct irq_chip *chip = legacy_pic->chip;
	...
	...
	...
```

[include/linux/irq.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/irq.h) 헤더 파일에 정의 된 `irq_chip` 구조체는 하드웨어 인터럽트 칩 디스크립터를 나타냅니다. 이것은 다음을 포함합니다. :

* `name` - name of a device. Used in the `/proc/interrupts`:

```C
$ cat /proc/interrupts
           CPU0       CPU1       CPU2       CPU3       CPU4       CPU5       CPU6       CPU7       
  0:         16          0          0          0          0          0          0          0   IO-APIC   2-edge      timer
  1:          2          0          0          0          0          0          0          0   IO-APIC   1-edge      i8042
  8:          1          0          0          0          0          0          0          0   IO-APIC   8-edge      rtc0
```

마지막 열을 보십시오;

* `(*irq_mask)(struct irq_data *data)`  - 인터럽트 소스를 마스킹;
* `(*irq_ack)(struct irq_data *data)` - 새 인터럽트의 시작;
* `(*irq_startup)(struct irq_data *data)` - 인터럽트 시작;
* `(*irq_shutdown)(struct irq_data *data)` - 인터럽트를 종료
* 등등.

'irq_data' 구조체는 칩 기능으로 전달 된 irq 당 칩 데이터 세트를 나타냅니다. 여기에는 칩 레지스터에 엑세스하기 위해 사전 계산 된 비트 마스크 인 `mask`,인터럽트 번호인 `irq`, 하드웨어 인터럽트 번호인 `hwirq`, 인터럽트 도메인 칩 하위 레벨 인터럽트 하드웨어 액세스 등이 포함됩니다.

이 명령은 `CONFIG_X86_64`와 커널 설정 옵션인 `CONFIG_X86_LOCAL_APIC`에 따라 [arch/x86/kernel/apic/apic.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/apic/apic.c)에서 `init_bsp_APIC` 함수를 호출합니다. :

```C
#if defined(CONFIG_X86_64) || defined(CONFIG_X86_LOCAL_APIC)
	init_bsp_APIC();
#endif
```

이 함수는 부트스트랩 프로세서(또는 먼저 시작하는 프로세서)의 [APIC]를 초기화합니다. 이는 [SMP](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller) 설정을 확인하는 것(Linux 커널 초기화 프로세스 챕터의 여섯번째 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-6)에 자세히 설명되어 있음)에서 시작하며, 이 프로세서에는 `APIC`이 있습니다. :

```C
if (smp_found_config || !cpu_has_apic)
	return;
```

이제 이 함수에서 리턴합니다. 다음 단계에서는 로컬 `APIC`(자세한 내용은 '고급 프로그래밍 가능 인터럽트 컨트롤러'에 대한 챕터에서 설명)을 끝내고, `unsigned int value`를 `APIC_SPIV_APIC_ENABLED`로 설정함으로써 첫번째 프로세서의 `APIC`을 가능하게 하는 동일한 소스 코드 파일내의 `clear_local_APIC` 함수를 호출합니다. :

```C
value = apic_read(APIC_SPIV);
value &= ~APIC_VECTOR_MASK;
value |= APIC_SPIV_APIC_ENABLED;
```

그리고 `apic_write` 함수의 도움으로 이를 write 합니다. :

```C
apic_write(APIC_SPIV, value);
```

부트 스트랩 프로세서에 `APIC`를 활성화 한 후 `init_ISA_irqs` 함수로 돌아가 그 다음 단계에서 레거시 `Programmable Interrupt Controller`를 초기화하고 각 레거시 irq에 대한 레거시 칩과 핸들러를 설정합니다. :

```C
legacy_pic->init(0);

for (i = 0; i < nr_legacy_irqs(); i++)
    irq_set_chip_and_handler(i, chip, handle_level_irq);
```

`init`함수는 어디에서 찾을 수 있을까요? [arch/x86/kernel/i8259.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/i8259.c)에 정의 된 `legacy_pic`은 다음과 같습니다. :

```C
struct legacy_pic *legacy_pic = &default_legacy_pic;
```

Where the `default_legacy_pic` is:

```C
struct legacy_pic default_legacy_pic = {
	...
	...
	...
	.init = init_8259A,
	...
	...
	...
}
```

동일한 소스 코드 파일에 정의 된 `init_8259A` 함수는 [Intel 8259](https://en.wikipedia.org/wiki/Intel_8259) `Programmable Interrupt Controller`의 초기화를 실행합니다. (자세한 내용은 `Programmable Interrupt Controllers`과 `APIC`에 대한 별도의 챕터에 있습니다.)

`init_ISA_irqs` 함수가 작업을 마친 후에, `native_init_IRQ` 함수로 돌아갈 수 있습니다. 다음 단계는 [내부 프로세서 인터럽트] (https://en.wikipedia.org/wiki/Inter-processor_interrupt)를 위해 [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing) 아키텍처에서 사용되는 특수 인터럽트 게이트를 할당하는 `apic_intr_init` 함수를 호출 하는 것입니다. 인터럽트 디스크립터 할당에 사용되는 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h)의 `alloc_intr_gate` 매크로가 있습니다. :

```C
#define alloc_intr_gate(n, addr)                        \
do {                                                    \
        alloc_system_vector(n);                         \
        set_intr_gate(n, addr);                         \
} while (0)
```

보시다시피, 우선 이것은 `used_vectors` 비트 맵에서 주어진 벡터 번호를 확인하는 `alloc_system_vector` 함수의 호출로 확장되고 (이전 [파트]에서 볼 수 있음), `used_vectors` 에 설정되지 않은 경우에는 비트 맵을 설정합니다. 그런 다음 우리는 `first_system_vector`가 주어진 인터럽트 벡터 수보다 큰지 테스트하고 더 큰 경우 할당합니다 :

```C
if (!test_bit(vector, used_vectors)) {
	set_bit(vector, used_vectors);
    if (first_system_vector > vector)
		first_system_vector = vector;
} else {
	BUG();
}
```

우리는 이미 `set_bit` 매크로를 보았습니다. 이제 `test_bit`와 `first_system_vector`를 봅시다. [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/bitops.h)에 정의 된 첫 번째 `test_bit` 매크로는 다음과 같습니다. :

```C
#define test_bit(nr, addr)                      \
        (__builtin_constant_p((nr))             \
         ? constant_test_bit((nr), (addr))      \
         : variable_test_bit((nr), (addr)))
```


[삼항 연산자](https://en.wikipedia.org/wiki/Ternary_operation)는 [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)에 내장된 테스트를 통해 확인할 수 있습니다. 함수 `__builtin_constant_p`는 컴파일 타임에 주어진 벡터 번호(`nr`)가 알려져 있는지 테스트합니다. `__builtin_constant_p`에 대한 잘못된 이해가 있다면 간단한 테스트를 만들 수 있습니다. :


```C
#include <stdio.h>

#define PREDEFINED_VAL 1

int main() {
	int i = 5;
	printf("__builtin_constant_p(i) is %d\n", __builtin_constant_p(i));
	printf("__builtin_constant_p(PREDEFINED_VAL) is %d\n", __builtin_constant_p(PREDEFINED_VAL));
	printf("__builtin_constant_p(100) is %d\n", __builtin_constant_p(100));

	return 0;
}
```

그리고 결과를 봅시다. :

```
$ gcc test.c -o test
$ ./test
__builtin_constant_p(i) is 0
__builtin_constant_p(PREDEFINED_VAL) is 1
__builtin_constant_p(100) is 1
```

이제 여러분이 이에 대해 명확히 이해했다고 생각합니다. 다시 `test_bit` 매크로로 돌아 갑시다. `__builtin_constant_p`가  0이 아닌 값을 반환하면 `constant_test_bit` 함수를 호출합니다 :

```C
static inline int constant_test_bit(int nr, const void *addr)
{
	const u32 *p = (const u32 *)addr;

	return ((1UL << (nr & 31)) & (p[nr >> 5])) != 0;
}
```

다른 방법으로 `variable_test_bit` 가 있습니다.:

```C
static inline int variable_test_bit(int nr, const void *addr)
{
        u8 v;
        const u32 *p = (const u32 *)addr;

        asm("btl %2,%1; setc %0" : "=qm" (v) : "m" (*p), "Ir" (nr));
        return v;
}
```

이 두 기능의 차이점은 무엇이고, 동일한 목적을 위해 두 개의 다른 기능이 필요한 이유는 무엇일까요? 이미 짐작할 수 있듯이 주된 목적은 최적화입니다. 다음 함수를 사용하여 간단한 예를 작성하면 다음과 같습니다. :

```C
#define CONST 25

int main() {
	int nr = 24;
	variable_test_bit(nr, (int*)0x10000000);
	constant_test_bit(CONST, (int*)0x10000000)
	return 0;
}
```

예제의 어셈블리 출력을 살펴보면 `constant_test_bit`에 대한 다음 어셈블리 코드가 표시됩니다.

```assembly
pushq	%rbp
movq	%rsp, %rbp

movl	$268435456, %esi
movl	$25, %edi
call	constant_test_bit
```

그리고 `variable_test_bit`에 대한 어셈블리 코드가 표시됩니다.:

```assembly
pushq	%rbp
movq	%rsp, %rbp

subq	$16, %rsp
movl	$24, -4(%rbp)
movl	-4(%rbp), %eax
movl	$268435456, %esi
movl	%eax, %edi
call	variable_test_bit
```

이 두 코드 목록은 같은 부분으로 시작합니다. 우선 현재 스택 프레임의 기본을 `%rbp` 레지스터에 저장합니다. 하지만 두 예에서 이 코드는 다릅니다. 첫 번째 예에서는 `$268435456`(여기서 $268435456은 두 번째 매개 변수 인 `0x10000000`)을 `esi`에, `$25`(첫 번째 매개 변수)를 `edi` 레지스터에 넣고 `constant_test_bit`를 호출합니다. x86_64 아키텍처를 위한 리눅스 커널을 배우면서 System V AMD64 ABI(호출 규칙)를 사용하기 때문에 함수 파라미터를 esi와 edi 레지스터에 넣었습니다. 이 작업은 모두 매우 간단합니다. 우리가 미리 정의 된 상수를 사용할 때, 컴파일러가 그 값을 대체 할 수 있습니다. 이제 두 번째 예를 살펴 보겠습니다. 여기서 볼 수 있듯이 컴파일러가 `nr` 변수의 값을 대체 할 수 없습니다. 이 경우 컴파일러는 프로그램의 [스택 프레임](https://ko.wikipedia.org/wiki/Call_stack)에서 오프셋을 계산해야합니다. 로컬 변수 데이터에 스택을 할당하기 위해 rsp 레지스터에서 `16`을 빼고 `$24` (`nr` 변수의 값)를 `-4` 오프셋을 가진 `rbp`에 넣습니다. 스택 프레임은 다음과 같습니다. :

```
         <- stack grows

	          %[rbp]
                 |
+----------+ +---------+ +---------+ +--------+
|          | |         | | return  | |        |
|    nr    |-|         |-|         |-|  argc  |
|          | |         | | address | |        |
+----------+ +---------+ +---------+ +--------+
                 |
              %[rsp]
```

그런 다음 이 값을 `eax`에 넣습니다. 따라서 `eax` 레지스터는 이제 `nr`의 값을 포함합니다. 결국 우리는 첫 번째 예에서 `$ 268435456`(`variable_test_bit` 함수의 첫 번째 매개 변수)과 `eax`의 값(`nr`의 값)을 `edi` 레지스터(`variable_test_bit` 함수의 두 번째 매개 변수)에 넣는 과정과 동일하게 수행합니다.

`apic_intr_init` 함수가 이 작업을 완료 한 후, 다음 단계는`FIRST_EXTERNAL_VECTOR` 또는 `0x20` 에서 `0x256`에 이르는 인터럽트 게이트 설정입니다. :

```C
i = FIRST_EXTERNAL_VECTOR;

#ifndef CONFIG_X86_LOCAL_APIC
#define first_system_vector NR_VECTORS
#endif

for_each_clear_bit_from(i, used_vectors, first_system_vector) {
	set_intr_gate(i, irq_entries_start + 8 * (i - FIRST_EXTERNAL_VECTOR));
}
```

그러나 우리는 `for_each_clear_bit_from` 헬퍼를 사용하면서 초기화되지 않은 인터럽트 게이트만 설정했습니다. 이후에는  동일한 `for_each_clear_bit_from` 헬퍼를 사용하여 인터럽트 테이블의 채워지지 않은 인터럽트 게이트를 `spurious_interrupt`로 채웁니다. :

```C
#ifdef CONFIG_X86_LOCAL_APIC
for_each_clear_bit_from(i, used_vectors, NR_VECTORS)
    set_intr_gate(i, spurious_interrupt);
#endif
```

`spurious_interrupt` 함수는 `spurious` 인터럽트를 위한 인터럽트 핸들러를 나타냅니다. 여기서 `used_vectors`는 이미 초기화 된 인터럽트 게이트를 포함하는 `unsigned long`입니다. 우리는 이미 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c) 소스 코드 파일에서 `trap_init` 함수의 첫 번째 `32` 인터럽트 벡터를 채웠습니다. :

```C
for (i = 0; i < FIRST_EXTERNAL_VECTOR; i++)
    set_bit(i, used_vectors);
```

이 장의 여섯 번째 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/interrupts/linux-interrupts-6)에서 이를 어떻게 수행했는지 보았습니다.

`native_init_IRQ` 함수의 끝에서 우리는 다음과 같은 점검을 볼 수 있습니다. :

```C
if (!acpi_ioapic && !of_ioapic && nr_legacy_irqs())
	setup_irq(2, &irq2);
```

우선 조건을 살펴 봅시다. `acpi_ioapic` 변수는 [I/O APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#I.2FO_APICs)의 유무를 나타냅니다. 이는 [arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/acpi/boot.c)에 정의되어 있습니다. 이 변수는 `Multiple APIC Description Table`을 처리하는동안 호출 된 `acpi_set_irq_model_ioapic` 함수에 설정되어 있습니다. 이것은 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c)에서 아키텍처 특정 항목을 초기화하는 동안 발생합니다(자세한 내용은 [APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)에 대해 다루는 다른 챕터에서 알 수 있습니다). `acpi_ioapic` 변수의 값은 `CONFIG_ACPI`와  Linux 커널 설정 옵션 `CONFIG_X86_LOCAL_APIC`에 따라 다릅니다. 이 옵션을 설정하지 않으면이 변수는 0이됩니다. :

```C
#define acpi_ioapic 0
```

두 번째 조건인 `!of_ioapic && nr_legacy_irqs()` 는 [Open Firmware](https://en.wikipedia.org/wiki/Open_Firmware) `I/O APIC`와 레거시 인터럽트 컨트롤러를 사용하지 않는지 확인합니다. 우리는 이미 `nr_legacy_irqs`에 대해 알고 있습니다. 두 번째는 [arch/x86/kernel/devicetree.c]에 정의 된 `of_ioapic` 변수이며, [devicetree](https://en.wikipedia.org/wiki/Device_tree)에서 `APICs` 에 대한 정보를 빌드하는 `dtb_ioapic_setup` 함수에서 초기화됩니다. `of_ioapic` 변수는 Linux 커널 설정 옵션 `CONFIG_OF` 에 따라 다릅니다. 이 옵션을 설정하지 않으면 `of_ioapic`의 값도 0이 됩니다 :


```C
#ifdef CONFIG_OF
extern int of_ioapic;
...
...
...
#else
#define of_ioapic 0
...
...
...
#endif
```

조건이 0이 아닌 값을 반환하면 다음 함수를 호출합니다. :

```C
setup_irq(2, &irq2);
```

우선 `irq2`에 대해 알아봅시다. `irq2`는 [arch/x86/kernel/irqinit.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/irqinit.c) 소스 코드 파일에 정의 된 `irqaction` 구조체이며 연결된 종속 장치를 조회하는 데 사용되는 `IRQ 2` 라인을 나타냅니다.

```C
static struct irqaction irq2 = {
	.handler = no_action,
    .name = "cascade",
    .flags = IRQF_NO_THREAD,
};
```

앞에서 인터럽트 컨트롤러는 두 개의 칩으로 구성되었고 하나는 두 번째 칩에 연결되었습니다. 이 `IRQ 2` 라인을 통해 첫 번째 칩에 연결된 두 번째 칩은 `8`부터 `15`까지의 라인들을 서비스했고, 이 다음에 첫번째 라인을 서비스했습니다. 예를 들어 [Intel 8259A](https://en.wikipedia.org/wiki/Intel_8259)는 다음과 같습니다. :

* `IRQ 0`  - 시스템 시간;
* `IRQ 1`  - 키보드;
* `IRQ 2`  - 종속 연결 된 장치들에 사용;
* `IRQ 8`  - [RTC](https://en.wikipedia.org/wiki/Real-time_clock);
* `IRQ 9`  - 예약됨;
* `IRQ 10` - 예약됨;
* `IRQ 11` - 예약됨;
* `IRQ 12` - `ps/2` 마우스;
* `IRQ 13` - 보조 프로세서;
* `IRQ 14` - 하드 드라이브 컨트롤러;
* `IRQ 1`  - 예약됨;
* `IRQ 3`  - `COM2` 와 `COM4`;
* `IRQ 4`  - `COM1` 와 `COM3`;
* `IRQ 5`  - `LPT2`;
* `IRQ 6`  - 드라이브 컨트롤러;
* `IRQ 7`  - `LPT1`.

[kernel/irq/manage.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/irq/manage.c)에 정의 된 `setup_irq` 함수는 두 개의 매개 변수를 사용합니다. :

* 인터럽트의 벡터 수;
* 인터럽트와 관련 된 `irqaction` 구조체.

이 함수는 처음에 주어진 벡터 번호에서 인터럽트 디스크립터를 초기화합니다. :

```C
struct irq_desc *desc = irq_to_desc(irq);
```

그리고 인터럽트가 주어지면 설정하는 `__setup_irq` 함수를 호출합니다. :

```C
chip_bus_lock(desc);
retval = __setup_irq(irq, desc, act);
chip_bus_sync_unlock(desc);
return retval;
```

`__setup_irq` 함수가 진행되는 동안 인터럽트 디스크립터가 잠기게 됩니다. `__setup_irq` 함수는 여러 가지를 만듭니다: 스레드 함수가 제공되고 인터럽트가 다른 인터럽트 스레드에 중첩되지 않을 때 핸들러 스레드를 만들고, 칩의 플래그를 설정하고. `irqaction`를 채우는 등의 많고 많은 일을 합니다.

위의 모든 것들은 `/prov/vector_number` 디렉토리를 생성하고 채웁니다. 그러나 최신 컴퓨터를 사용하는 경우 모든 값은 0입니다.

```
$ cat /proc/irq/2/node
0

$cat /proc/irq/2/affinity_hint
00

cat /proc/irq/2/spurious
count 0
unhandled 0
last_unhandled 0 ms
```

아마도 `APIC`는 우리 컴퓨터의 인터럽트를 처리하기 때문입니다.

이것으로 끝입니다.

결론
--------------------------------------------------------------------------------


[인터럽트와 인터럽트 처리](https://junsoolee.gitbook.io/linux-insides-ko/summary/interrupts) 챕터의 8 번째 파트가 끝났으며, 우리는 이 파트에서 외부 하드웨어 인터럽트에 대해 계속 연구했습니다. 이전 파트에서 우리는 이를 시작했고 `IRQs`의 초기 초기화를 보았습니다. 그리고 이 파트에서 우리는 이미 `init_IRQ` 함수에서 초기 인터럽트가 아닌 인터럽트를 보았습니다. 인터럽트의 벡터 번호를 저장하는 CPU 당 `vector_irq`의 초기화를 보았으며, 외부 하드웨어 인터럽트와 관련된 다른 것들의 인터럽트 처리 및 초기화 중에 사용될 것입니다.

다음 파트에서는 관련 내용을 처리하는 인터럽트를 계속 배우고 `softirqs`의 초기화를 보게 될 것입니다.

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견이나 핑을 남겨주세요.

**영어는 제 모국어가 아닙니다. 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수를 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한글 번역](https://github.com/junsooo/linux-insides-ko)으로 PR을 보내주세요.**

Links
--------------------------------------------------------------------------------

* [IRQ](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [percpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Intel 8259](https://en.wikipedia.org/wiki/Intel_8259)
* [Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)
* [ISA](https://en.wikipedia.org/wiki/Industry_Standard_Architecture)
* [MultiProcessor Configuration Table](https://en.wikipedia.org/wiki/MultiProcessor_Specification)
* [Local APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#Integrated_local_APICs)
* [I/O APIC](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller#I.2FO_APICs)
* [SMP](https://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [Inter-processor interrupt](https://en.wikipedia.org/wiki/Inter-processor_interrupt)
* [ternary operator](https://en.wikipedia.org/wiki/Ternary_operation)
* [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [calling convention](https://en.wikipedia.org/wiki/X86_calling_conventions)
* [PDF. System V Application Binary Interface AMD64](http://x86-64.org/documentation/abi.pdf)
* [Call stack](https://en.wikipedia.org/wiki/Call_stack)
* [Open Firmware](https://en.wikipedia.org/wiki/Open_Firmware)
* [devicetree](https://en.wikipedia.org/wiki/Device_tree)
* [RTC](https://en.wikipedia.org/wiki/Real-time_clock)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-7.html)
