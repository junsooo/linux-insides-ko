커널 초기화. Part 2.
================================================================================

초기 인터럽트와 예외 처리
--------------------------------------------------------------------------------

이전의 [시간](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)에 우리는 초기 인터럽트 핸들러를 설정하기 전까지를 다뤘습니다. 지금 우리는 압축 해제 된 Linux 커널에 있고 초기 부팅을위한 기본 [paging](https://en.wikipedia.org/wiki/Page_table) 구조체를 가지고 있으며 현재 목표는 기본 커널 코드가 작동을 시작하기 전에 초기 준비를 완료하는 것입니다.

우리는 이미 이번 [챕터](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)의 지난 [첫](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) 시간부터 이 준비를 시작했었습니다. 우리는 이 부분에서 계속해서 인터럽트와 예외 처리에 대해 더 많이 알게 될 것입니다.

우리가 이전에 [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)의

```C
	idt_setup_early_handler();
```

함수에서 멈췄다는 것을 기억하십니까? 그러나 이 함수를 해결하기 전에 먼저 인터럽트와 핸들러에 대해 알아야합니다.

몇가지 이론
--------------------------------------------------------------------------------

인터럽트는 소프트웨어 또는 하드웨어로 인해 발생하는 CPU에 대한 이벤트입니다. 예를 들어 사용자가 키보드에서 키를 눌렀습니다. 인터럽트가 발생하면 CPU는 현재 작업을 중지하고 제어권을 [인터럽트 핸들러](https://en.wikipedia.org/wiki/Interrupt_handler)라고 불리는 특별한 루틴에게 보냅니다. 인터럽트 핸들러는 처리하고, 인터럽트하고, 이전에 중지 된 작업으로 제어권를 되돌려줍니다. 인터럽트는 세 가지 유형으로 나눌 수 있습니다:

* 소프트웨어 인터럽트 - 소프트웨어가 CPU에 신호를 보내 커널의 주목이 필요하다는 신호를 보낼 때. 이 인터럽트는 일반적으로 시스템 호출에 사용됩니다.
* 하드웨어 인터럽트 - 하드웨어 이벤트가 발생할 때 (예 : 키보드의 버튼을 누르는 경우)
* 예외-CPU가 오류 (예 : 0으로 나누기 또는 RAM에없는 메모리 페이지에 액세스)를 감지할 때 CPU에서 생성 되는 인터럽트입니다.

모든 인터럽트와 예외에는 `vector number`라고 하는 고유 번호가 할당됩니다. `Vector number`는 `0`부터 `255`까지의 어떤 숫자든 될 수 있습니다. 예외에는 `32` 벡터 번호부터를 사용하는 것이 일반적인 관습이며, `32`에서`255`까지의 벡터 번호는 사용자 정의 인터럽트에 사용됩니다.

CPU는 `인터럽트 디스크립터 테이블 (Interrupt Descriptor Table)`에서 벡터 번호를 인덱스로 사용합니다 (곧 이에 대한 설명을 볼 것입니다). CPU는 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller),  또는 그것의 핀을 통해 인터럽트를 포착합니다. 다음 표는`0-31` 예외를 보여줍니다.

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                   |
----------------------------------------------------------------------------------------------
|0     | #DE    |Divide Error        |Fault|NO        |DIV and IDIV                          |
|---------------------------------------------------------------------------------------------
|1     | #DB    |Reserved            |F/T  |NO        |                                      |
|---------------------------------------------------------------------------------------------
|2     | ---    |NMI                 |INT  |NO        |external NMI                          |
|---------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                                 |
|---------------------------------------------------------------------------------------------
|4     | #OF    |Overflow            |Trap |NO        |INTO  instruction                     |
|---------------------------------------------------------------------------------------------
|5     | #BR    |Bound Range Exceeded|Fault|NO        |BOUND instruction                     |
|---------------------------------------------------------------------------------------------
|6     | #UD    |Invalid Opcode      |Fault|NO        |UD2 instruction                       |
|---------------------------------------------------------------------------------------------
|7     | #NM    |Device Not Available|Fault|NO        |Floating point or [F]WAIT             |
|---------------------------------------------------------------------------------------------
|8     | #DF    |Double Fault        |Abort|YES       |An instruction which can generate NMI |
|---------------------------------------------------------------------------------------------
|9     | ---    |Reserved            |Fault|NO        |                                      |
|---------------------------------------------------------------------------------------------
|10    | #TS    |Invalid TSS         |Fault|YES       |Task switch or TSS access             |
|---------------------------------------------------------------------------------------------
|11    | #NP    |Segment Not Present |Fault|NO        |Accessing segment register            |
|---------------------------------------------------------------------------------------------
|12    | #SS    |Stack-Segment Fault |Fault|YES       |Stack operations                      |
|---------------------------------------------------------------------------------------------
|13    | #GP    |General Protection  |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|14    | #PF    |Page fault          |Fault|YES       |Memory reference                      |
|---------------------------------------------------------------------------------------------
|15    | ---    |Reserved            |     |NO        |                                      |
|---------------------------------------------------------------------------------------------
|16    | #MF    |x87 FPU fp error    |Fault|NO        |Floating point or [F]Wait             |
|---------------------------------------------------------------------------------------------
|17    | #AC    |Alignment Check     |Fault|YES       |Data reference                        |
|---------------------------------------------------------------------------------------------
|18    | #MC    |Machine Check       |Abort|NO        |                                      |
|---------------------------------------------------------------------------------------------
|19    | #XM    |SIMD fp exception   |Fault|NO        |SSE[2,3] instructions                 |
|---------------------------------------------------------------------------------------------
|20    | #VE    |Virtualization exc. |Fault|NO        |EPT violations                        |
|---------------------------------------------------------------------------------------------
|21-31 | ---    |Reserved            |INT  |NO        |External interrupts                   |
----------------------------------------------------------------------------------------------
```

CPU 인터럽트에 반응하기 위해선 인터럽트 디스크립터 테이블(Interrupt Descriptor Table) 또는 IDT라고 불리는 특수한 구조체를 사용합니다. IDT는 Global Descriptor Table과 같은 8 바이트 디스크립터 배열이지만 IDT의 엔트리(항목)들은 '게이트'(`gates`) 라고 불립니다. CPU는 벡터 번호에 8을 곱하여 IDT 엔트리를 찾습니다. 그러나 64 비트 모드에선 IDT는 16 바이트 디스크립터 배열이며 CPU는 벡터 번호에 16을 곱하여 IDT에서 엔트리를 찾습니다. 이전 부분의 내용을 기억하시듯, CPU는 전역 디스크립터 테이블(`Global Descriptor Table`)을 찾기 위해 특수한 `GDTR`레지스터를 사용하므로 CPU는 인터럽트 디스크립터 테이블에 `IDTR`이라는 특수한 레지스터를 사용하고 테이블의 기본 주소를 이 레지스터에 로드하기 위해 `lidt`명령을 사용합니다.

64비트 모드의 IDT 요소들은 다음 구조를 따릅니다:

```
127                                                                             96
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved                                       |
|                                                                               |
 --------------------------------------------------------------------------------
95                                                                              64
 --------------------------------------------------------------------------------
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
 --------------------------------------------------------------------------------
63                               48 47      46  44   42    39             34    32
 --------------------------------------------------------------------------------
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 --------------------------------------------------------------------------------
31                                   16 15                                      0
 --------------------------------------------------------------------------------
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
 --------------------------------------------------------------------------------
```

여기서:

* `Offset` - 인터럽트 핸들러의 엔트리 포인트까지의 오프셋;
* `DPL` -    Descriptor Privilege Level (디스크립터 권한 레벨);
* `P` -      세그먼트 존재여부(Present) 플래그;
* `Segment selector` - GDT 또는 LDT의 코드 세그먼트 셀렉터
* `IST` -    인터럽트 처리를 위해 새 스택으로 전환하는 기능을 제공

그리고 마지막 `Type` 필드는 `IDT` 엔트리의 유형(타입)을 기술합니다. 인터럽트에는 세 가지 종류의 게이트가 있습니다.

* 작업 게이트(Task gate)
* 인터럽트 게이트(Interrupt gate)
* 트랩 게이트(Trap gate)

 인터럽트 및 트랩 게이트에는 인터럽트 핸들러의 엔트리 포인트에 대한 원거리 포인터(far pointer)가 포함되어 있습니다. 이 두 유형들 사이의 단 한 가지 차이점은 CPU가 'IF'플래그를 처리하는 방법입니다. 인터럽트 게이트를 통해 인터럽트 핸들러에 액세스한 경우 CPU는`IF` 플래그를 지워 현재 인터럽트 핸들러가 실행되는 동안 다른 인터럽트를 방지합니다. 현재 인터럽트 핸들러가 실행 된 후 CPU는`iret` 명령으로`IF` 플래그를 다시 설정합니다.

인터럽트 디스크립터의 다른 비트는 예약되어 있으며 0이어야합니다. 이제 CPU가 인터럽트를 처리하는 방법을 살펴봅시다.

* CPU는 플래그 레지스터,`CS` 및 명령어 포인터를 스택에 저장합니다.
* 인터럽트가 에러 코드 (예시: `#PF`)를 유발하면 CPU는 스택의 명령 포인터 다음에 에러를 저장합니다.
* 인터럽트 핸들러가 실행된 후에는, 다시 돌아오기 위해 `iret` 명령이 사용됩니다.

이제 코드로 돌아가 봅시다.

IDT 채우기 및 불러오기
--------------------------------------------------------------------------------

우리는 다음 함수에서 멈췄습니다.

```C
	idt_setup_early_handler();
```

`idt_setup_early_handler`는 다음과 같이 [arch / x86 / kernel / idt.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/idt.c)에 정의되어 있습니다 :

```C
void __init idt_setup_early_handler(void)
{
	int i;

	for (i = 0; i < NUM_EXCEPTION_VECTORS; i++)
		set_intr_gate(i, early_idt_handler_array[i]);

	load_idt(&idt_descr);
}
```

그리고 여기서 `NUM_EXCEPTION_VECTORS`는 `32`로 확장됩니다. 보시다시피, 우리는 루프에서 처음 32 개의 `IDT` 엔트리만 채우는데, 왜냐하면 초기 설정은 모두 인터럽트가 비활성화 된 상태로 실행되기 때문이고, 그렇기 때문에`32`보다 큰 벡터에 대해서는 인터럽트 핸들러를 설정할 필요가 없습니다. 여기 루프에서`set_intr_gate`를 호출하는데에는 두 개의 매개 변수가 필요합니다 :

* 인터럽트 번호 또는 `vector number`;
* idt 핸들러의 주소.

그리고 `&idt_descr` 배열로 표현되는`IDT` 테이블에 인터럽트 게이트를 삽입합니다. 

`early_idt_handler_array` 배열은 [arch / x86 / include / asm / segment.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/segment.h)헤더 파일에 선언되어 있으며 처음 `32`개 예외(exception) 핸들러의 주소를 포함합니다.

```C
#define EARLY_IDT_HANDLER_SIZE   9
#define NUM_EXCEPTION_VECTORS	32

extern const char early_idt_handler_array[NUM_EXCEPTION_VECTORS][EARLY_IDT_HANDLER_SIZE];
```

`early_idt_handler_array`는`288` 바이트 배열이며 매 9 바이트마다 예외 엔트리 포인트의 주소를 가지고 있습니다. 이 배열의 모든 각각의 9 바이트는 예외가 오류 코드를 제공하지 않는 경우 더미 오류 코드를 푸시하기위한 2 바이트의 옵션 명령어, 벡터 번호를 스택으로 푸시하기위한 2 바이트 명령어 및 공통 예외 핸들러 코드로 `jump`하기 위한 5바이트의 명령어를 가지고 있습니다. 다음 단락에서 자세한 내용을 볼 수 있습니다.

`set_intr_gate` 함수는 [arch / x86 / kernel / idt.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/idt.c) 소스 파일에 정의되어 있으며 다음과 같습니다 :

```C
static void set_intr_gate(unsigned int n, const void *addr)
{
	struct idt_data data;

	BUG_ON(n > 0xFF);

	memset(&data, 0, sizeof(data));
	data.vector	= n;
	data.addr	= addr;
	data.segment	= __KERNEL_CS;
	data.bits.type	= GATE_INTERRUPT;
	data.bits.p	= 1;

        idt_setup_from_table(idt_table, &data, 1, false);
}
```

함수는 먼저 `BUG_ON` 매크로를 사용하여 전달 된 벡터 번호가 `255`보다 크지 않은지 확인합니다. 이는 최대 `256`개의 인터럽트를 갖도록 제한되어 있기 때문에 필요한 작업입니다. 그런 다음, 주어진 인자와 `idt_setup_from_table`로 전달된 다른 인자로 idt 데이터를 채웁니다. `idt_setup_from_table` 함수는 다음과 같이 `set_intr_gate` 함수와 동일한 파일에 정의되어 있습니다 :

```C
static void
idt_setup_from_table(gate_desc *idt, const struct idt_data *t, int size, bool sys)
{
	gate_desc desc;

	for (; size > 0; t++, size--) {
		desc.offset_low    = (u16) t->addr;
		desc.segment	   = (u16) t->segment
		desc.bits	   = t->bits;
		desc.offset_middle = (u16) (t->addr >> 16);
		desc.offset_high   = (u32) (t->addr >> 32);
		desc.reserved	   = 0;
		memcpy(&idt[t->vector], &desc, sizeof(desc));
		if (sys)
			set_bit(t->vector, system_vectors);
	}
}
```

위 함수는 주어진 인자 등으로 임시 idt 설명자를 채웁니다. 그런 다음 `idt_table` 배열의 특정 요소에 복사합니다. `idt_table`은 idt 엔트리의 배열입니다 :

```C
gate_desc idt_table[IDT_ENTRIES] __page_aligned_bss;
```

이제 우리는 메인 루프 코드로 돌아갑니다. 메인 루프가 끝나면 :

```C
	load_idt((const struct desc_ptr *)&idt_descr);
```

-을 호출하여 `Interrupt Descriptor table`을 로드할 수 있습니다. 여기서 `idt_descr`는:

```C
struct desc_ptr idt_descr __ro_after_init = {
	.size		= (IDT_ENTRIES * 2 * sizeof(unsigned long)) - 1,
	.address	= (unsigned long) idt_table,
};
```

이고, `load_idt`는 단지 `lidt` 명령을 실행합니다 :

```C
	asm volatile("lidt %0"::"m" (idt_descr));
```

자 이제 우리는`Interrupt Descriptor Table`을 채우고 로드했으며, 인터럽트 동안 CPU가 어떻게 동작하는지 알고 있습니다. 이제 인터럽트 핸들러를 다룰 시간입니다.

초기 인터럽트 핸들러(Early interrupts handlers)
--------------------------------------------------------------------------------

위에서 볼 수 있듯이, IDT는`early_idt_handler_array`의 주소로 채워졌습니다. 이 섹션에서는 이에 대해 더 자세히 살펴 보겠습니다. 이것은 [arch / x86 / kernel / head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 어셈블리 파일에서 찾아 볼 수 있습니다.

```assembly
ENTRY(early_idt_handler_array)
	i = 0
	.rept NUM_EXCEPTION_VECTORS
	.if ((EXCEPTION_ERRCODE_MASK >> i) & 1) == 0
		UNWIND_HINT_IRET_REGS
		pushq $0	# Dummy error code, to make stack frame uniform
	.else
		UNWIND_HINT_IRET_REGS offset=8
	.endif
	pushq $i		# 72(%rsp) Vector number
	jmp early_idt_handler_common
	UNWIND_HINT_IRET_REGS
	i = i + 1
	.fill early_idt_handler_array + i*EARLY_IDT_HANDLER_SIZE - ., 1, 0xcc
	.endr
	UNWIND_HINT_IRET_REGS offset=16
END(early_idt_handler_array)
```

여기서 처음 `32`개 예외에 대한 인터럽트 핸들러 생성을 볼 수 있습니다. 여기를 살펴보면, 예외를 확인하고 예외에서 에러 코드가 있으면  아무것도 하지 않으며, 예외가 오류 코드를 반환하지 않으면 스택에 0을 푸시합니다. 이는 그 스택이 균일하기 때문입니다. 그런 다음 스택으로 '벡터 번호'를 푸시하고 일단 일반 인터럽트 핸들러(generic interrupt handler) 인 `early_idt_handler_common`으로 이동합니다. 결국, `early_idt_handler_array` 배열의 모든 각 9 바이트는 에러 코드의 선택적인 푸시, 벡터 번호(`vector number`)의 푸시 및 `early_idt_handler_common`으로의 점프 명령어로 구성되는 것입니다. `objdump` 유틸리티의 출력에서 이를 볼 수 있습니다 :

```
$ objdump -D vmlinux
...
...
...
ffffffff81fe5000 <early_idt_handler_array>:
ffffffff81fe5000:       6a 00                   pushq  $0x0
ffffffff81fe5002:       6a 00                   pushq  $0x0
ffffffff81fe5004:       e9 17 01 00 00          jmpq   ffffffff81fe5120 <early_idt_handler_common>
ffffffff81fe5009:       6a 00                   pushq  $0x0
ffffffff81fe500b:       6a 01                   pushq  $0x1
ffffffff81fe500d:       e9 0e 01 00 00          jmpq   ffffffff81fe5120 <early_idt_handler_common>
ffffffff81fe5012:       6a 00                   pushq  $0x0
ffffffff81fe5014:       6a 02                   pushq  $0x2
...
...
...
```

아시다시피 CPU는 인터럽트 핸들러를 호출하기 전에 스택에서 플래그 레지스터, `CS`, `RIP`를 푸시합니다. 따라서 `early_idt_handler_common`이 실행되기 전에 스택에는 다음 데이터가 포함됩니다.

```
|--------------------|
| %rflags            |
| %cs                |
| %rip               |
| error code         | <-- %rsp
|--------------------|
```

이제`early_idt_handler_common` 구현을 살펴 봅시다. 이것도 동일한 [arch / x86 / kernel / head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S) 어셈블리 파일에 있습니다. 가장 먼저, `early_idt_handler_common`에서 재귀를 막기 위해 `early_recursion_flag`를 증가시킵니다 :

```assembly
	incl early_recursion_flag(%rip)
```

다음으로 일반(general) 레지스터를 스택에 저장합니다.

```assembly
	pushq %rsi
	movq 8(%rsp), %rsi
	movq %rdi, 8(%rsp)
	pushq %rdx
	pushq %rcx
	pushq %rax
	pushq %r8
	pushq %r9
	pushq %r10
	pushq %r11
	pushq %rbx
	pushq %rbp
	pushq %r12
	pushq %r13
	pushq %r14
	pushq %r15
	UNWIND_HINT_REGS
```

인터럽트 핸들러에서 돌아올 때(리턴할 때) 레지스터 값이 잘못되는 것을 방지하기 위해 이를 수행해야합니다. 그 다음엔 벡터 번호를 확인하고 벡터 번호가`#PF` 혹은 [Page Fault](https://en.wikipedia.org/wiki/Page_fault)이면 `cr2`의 값을 `rdi`레지스터에 넣고 `early_make_pgtable`(이에 대해선 곧 보게 될 것입니다)을 호출합니다:

```assembly
	cmpq $14,%rsi
	jnz 10f
	GET_CR2_INTO(%rdi)
	call early_make_pgtable
	andl %eax,%eax
	jz 20f
```

그렇지 않은 경우에는 커널 스택 포인터를 전달하여 `early_fixup_exception` 함수를 호출합니다.

```assembly
10:
	movq %rsp,%rdi
	call early_fixup_exception
```

`early_fixup_exception` 함수의 구현은 나중에 보도록 합시다.

```assembly
20:
	decl early_recursion_flag(%rip)
	jmp restore_regs_and_return_to_kernel
```

`early_recursion_flag`를 줄인 후에는, 이전에 스택에다 저장해둔 레지스터를 복원하고 `iretq`로 핸들러에서 돌아옵니다(리턴합니다).

이것으로 인터럽트 핸들러는 끝입니다. 이제부턴 페이지 결함 처리 및 기타 예외 처리를 순서대로 살펴 보겠습니다.

페이지 오류(Page fault) 처리
--------------------------------------------------------------------------------

이전 단락에서 벡터 번호가 페이지 오류인지 확인하고 새 페이지 테이블이 있는 경우 이를 새로 만들기(build) 위해 `early_make_pgtable`을 호출하는 초기 인터럽트 핸들러를 보았습니다. 이 단계에서 커널의 `4G` 이상을 로드하고 `4G` 이상의 `boot_params` 구조체에 액세스 할 수 있는 기능을 추가 할 계획이므로 이 단계에서는 `#PF` 핸들러가 필요합니다.

[arch / x86 / kernel / head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)에서 `early_make_pgtable`의 구현을 찾아볼 수 있으며, 하나의 매개 변수-`cr2` 레지스터의 값으로, 페이지 오류를 일으킨 주소를 포함합니다-를 가지고 있음을 확인할 수 있습니다. 한 번 살펴 봅시다.

```C
int __init early_make_pgtable(unsigned long address)
{
	unsigned long physaddr = address - __PAGE_OFFSET;
	pmdval_t pmd;

	pmd = (physaddr & PMD_MASK) + early_pmd_flags;

	return __early_make_pgtable(address, pmd);
}
```

우리는 `pmd`를 초기화하고 `address`와 함께 `__early_make_pgtable` 함수에 전달합니다. `__early_make_pgtable` 함수는 다음과 같이`early_make_pgtable` 함수와 동일한 파일에 정의되어 있습니다:

```C
int __init __early_make_pgtable(unsigned long address, pmdval_t pmd)
{
	unsigned long physaddr = address - __PAGE_OFFSET;
	pgdval_t pgd, *pgd_p;
	p4dval_t p4d, *p4d_p;
	pudval_t pud, *pud_p;
	pmdval_t *pmd_p;
	...
	...
	...
}
```

이것은 `* val_t` 타입을 가진 일부 변수의 정의에서부터 시작합니다. 이 타입들은 모두 `typedef`를 사용하여 `unsigned long`의 별칭(alias)으로 선언됩니다.

유효하지 않은 주소가 없는지 확인한 후 페이지 상단 디렉토리(Page Upper Directory)의 기본 주소가 포함 된 페이지 전역 디렉토리(Page Global Directory) 엔트리의 주소를 가져 와서 그 값을 `pgd` 변수에 넣습니다.

```C
again:
	pgd_p = &early_top_pgt[pgd_index(address)].pgd;
	pgd = *pgd_p;
```

그리고 `pgd`가 존재하는지 확인합니다. 만약 그렇다면, 페이지 상단 디렉토리 테이블의 기본 주소를`pud_p`에 할당합니다 :

```C
	pud_p = (pudval_t *)((pgd & PTE_PFN_MASK) + __START_KERNEL_map - phys_base);
```

여기서 `PTE_PFN_MASK`는`(pte|pmd|pud|pgd)val_t`의 하위 12 비트를 마스킹하는 매크로입니다.

만약 `pgd`가 존재하지 않으면, 우리는 `next_early_pgt`가 `EARLY_DYNAMIC_PAGE_TABLES`의 `64`보다 크지 않은지 확인하고 요청시 새로운 페이지 테이블을 설정하기 위해 고정 된 수의 버퍼를 제공합니다. `next_early_pgt`가 `EARLY_DYNAMIC_PAGE_TABLES`보다 크다면 페이지 테이블을 재설정하고  `again` 라벨에서 다시 시작합니다. 만약 `next_early_pgt`가 `EARLY_DYNAMIC_PAGE_TABLES`보다 작다면, 다음 번 `early_dynamic_pgts`의 엔트리를 `pud_p`에 할당하고 페이지 상단 디렉토리의 전체 엔트리를 '0 '으로 채운 다음, 페이지 전역 디렉토리 엔트리를 기본 주소와 일부 접근 권한으로 채웁니다 :

```C
	if (next_early_pgt >= EARLY_DYNAMIC_PAGE_TABLES) {
		reset_early_page_tables();
		goto again;
	}
		
	pud_p = (pudval_t *)early_dynamic_pgts[next_early_pgt++];
	memset(pud_p, 0, sizeof(*pud_p) * PTRS_PER_PUD);
	*pgd_p = (pgdval_t)pud_p - __START_KERNEL_map + phys_base + _KERNPG_TABLE;
```

그리고 우리는`pud_p`를 올바른 엔트리를 가리키게 수정하고 그 값을`pud`에 다음과 같이 할당합니다 :

```C
	pud_p += pud_index(address);
	pud = *pud_p;
```

그런 다음 페이지 중간 디렉토리(page middle directory)에 위와 동일한 루틴을 수행합니다.

마지막으로 우리는 `early_make_pgtable` 함수에 의해 전달된 `pmd`를 커널 텍스트와 데이터 가상 주소를 매핑하는 페이지 중간 디렉토리의 특정 엔트리에 할당합니다 :

```C
	pmd_p[pmd_index(address)] = pmd;
```

페이지 오류 핸들러가 작업을 완료 한 후 그 결과로 `early_top_pgt`는 유효한 주소를 가리키는 엔트리가 담기게 됩니다.

다른 예외 처리
--------------------------------------------------------------------------------

초기 인터럽트 단계에서 페이지 오류 이외의 예외들은 [arch/x86/mm/extable.c](https://github.com/torvalds/linux/blob/master/arch/x86/mm/extable.c)에 정의 된`early_fixup_exception` 함수로 처리되며 이 함수는 두 개의 매개 변수를 사용합니다 -저장된 레지스터를 가지고 있는 커널 스택에 대한 포인터, 그리고 벡터 번호 :

```C
void __init early_fixup_exception(struct pt_regs *regs, int trapnr)
{
	...
	...
	...
}
```

먼저 다음과 같이 몇가지를 확인해야합니다.

```C
	if (trapnr == X86_TRAP_NMI)
		return;

	if (early_recursion_flag > 2)
		goto halt_loop;

	if (!xen_pv_domain() && regs->cs != __KERNEL_CS)
		goto fail;
```

여기서는 [NMI](https://en.wikipedia.org/wiki/Non-maskable_interrupt)를 그냥 무시하고 재귀 상황이 아닌 것을 확실히 합니다.

그 다음은 다음과 같습니다:

```C
	if (fixup_exception(regs, trapnr))
		return;
```

`fixup_exception` 함수는 실제 핸들러를 찾아서 호출합니다. 이 함수는 다음과 같이 `early_fixup_exception` 함수와 동일한 파일에 정의되어 있습니다.

```C
int fixup_exception(struct pt_regs *regs, int trapnr)
{
	const struct exception_table_entry *e;
	ex_handler_t handler;

	e = search_exception_tables(regs->ip);
	if (!e)
		return 0;

	handler = ex_fixup_handler(e);
	return handler(e, regs, trapnr);
}
```

`ex_handler_t`는 다음과 같이 정의된 함수 포인터 타입입니다.

```C
typedef bool (*ex_handler_t)(const struct exception_table_entry *,
                            struct pt_regs *, int)
```

`search_exception_tables` 함수는 예외 테이블(exception table)에서 주어진 주소를 찾습니다 (즉, ELF 섹션에서의`__ex_table`). 그 후, `ex_fixup_handler` 함수로 실제 주소를 얻습니다. 마지막으로 실제 핸들러를 호출합니다. 예외 테이블에 대한 자세한 내용은 [Documentation / x86 / exception-tables.txt](https://github.com/torvalds/linux/blob/master/Documentation/x86/exception-tables.txt)를 참고하세요.

`early_fixup_exception` 함수로 돌아갑시다. 다음 단계는 다음과 같습니다:

```C
	if (fixup_bug(regs, trapnr))
		return;
```

`fixup_bug` 함수는 [arch / x86 / kernel / traps.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/traps.c)에 정의되어 있습니다. 함수의 구현에 대해 살펴봅시다.

```C
int fixup_bug(struct pt_regs *regs, int trapnr)
{
	if (trapnr != X86_TRAP_UD)
		return 0;

	switch (report_bug(regs->ip, regs)) {
	case BUG_TRAP_TYPE_NONE:
	case BUG_TRAP_TYPE_BUG:
		break;

	case BUG_TRAP_TYPE_WARN:
		regs->ip += LEN_UD2;
		return 1;
	}

	return 0;
}
```

이 함수가 하는 일은 단지 `#UD` (또는 [Invalid Opcode](https://wiki.osdev.org/Exceptions#Invalid_Opcode))가 발생하고 `report_bug` 함수가 `BUG_TRAP_TYPE_WARN`을 리턴하여 예외가 발생하면 `1`을 반환하고 , 그렇지 않으면`0`을 반환하는 것입니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널 내부에 대한 두 번째 부분은 끝입니다. 만약 질문이나 의견이 있으시다면, 트위터에서 [0xAX](https://twitter.com/0xAX) 저를 핑해주시거나, [이메일](anotherworldofworld@gmail.com)을 보내주시거나, 또는 그냥 [이슈](https://github.com/0xAX/linux-insides/issues/new)를 생성해주세요. 다음 장에서는 커널 엔트리 포인트 이전의 모든 단계 `start_kernel` 함수를 살펴볼 것입니다.

**영어는 제 모국어가 아닙니다, 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수들을 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한국 번역](https://github.com/junsooo/linux-insides-ko)로 PR을 보내주세요.**

링크 모음
--------------------------------------------------------------------------------

* [GNU assembly .rept](https://sourceware.org/binutils/docs-2.23/as/Rept.html)
* [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [Page table](https://en.wikipedia.org/wiki/Page_table)
* [Interrupt handler](https://en.wikipedia.org/wiki/Interrupt_handler)
* [Page Fault](https://en.wikipedia.org/wiki/Page_fault),
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)
