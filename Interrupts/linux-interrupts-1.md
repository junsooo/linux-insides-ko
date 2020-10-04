Interrupts and Interrupt Handling. Part 1.
================================================================================

Introduction
--------------------------------------------------------------------------------

이번 챕터는 [linux insides](https://0xax.gitbooks.io/linux-insides/content/)책의 새로운 챕터의 첫 부분입니다. 이전 챕터에서 많은 것을 보았습니다. 우리는 커널 초기화의 가장 첫 [과정](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) of kernel initialization and finished with the [launch](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-10.html)으로 시작해서 첫 프로세스인 `init`의 [시작](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-10.html)으로 이전 챕터를 끝마쳤습니다. 네, 우리는 다양한 커널의 서브시스템에 관련된 몇몇의 초기화 과정들을 지켜봤습니다. 그러나 우리는 아직 이러한 서브시스템들을 심층적으로 보지 못했습니다. 이번 챕터에서, 우리는 다양한 커널 서브시스템들이 어떻게 작동하는지 이해할 것이고, 그것들이 어떻게 구현되어있는지 살펴볼 것 입니다. 챕터의 제목으로부터 이미 유추할 수 있듯이, 첫 서브시스템은 [Interrupt](http://en.wikipedia.org/wiki/Interrupt)입니다. 

Interrupt란 무엇인가?
--------------------------------------------------------------------------------
우리는 이미 이 책의 여러 부분에서 `interrupt`라는 단어를 봐왔습니다. 우리는 심지어 interrupt handler들의 여러 예도 봤습니다. 이제 이번 챕터에서 우리는 다음과 같은 내용을 시작할 것입니다. 
* `interrupts`란 무엇인가?
* `interrupt handlers`란 무엇인가?

그리고 나서 우리는 `interrupts`에 대해 더 깊게 파보고, 리눅스 커널에서 `interrupts`를 어떻게 다루는지 공부할 것입니다.

우리가 `interrupt`라는 단어를 처음 접했을 때 생기는 첫 질문은 `interrupt란 무엇일까?`라는 것입니다. Interrupt는 소프트웨어나 하드웨어로 인해 발생하는 CPU의 참여를 필요로 하는 `event`로 의할 수 있습니다. 예를 들면, 키보드의 버튼을 누르면, 우리는 무엇을 기대할까요? 운영체제와 컴퓨터는 키보드 버튼을 누른다음에 무엇을 해야할까요? 문제를 단순화시키기 위해서, 각 주변 장치들은 단 하나의 CPU로의 interrupt line을 가지고 있다고 가정해봅시다. 장치는 CPU로 interrupt 신호를 보내기 위해 interrupt line을 사용할 수 있습니다. 그러나 interrupts는 CPU에 직접적으로 전달되지 않습니다. 오래된 기계에서는, Programmable Interrupt Controller[PIC](http://en.wikipedia.org/wiki/Programmable_interrupt_controller)라는 여러대의 장치로부터 들어오는 여러개의 Interrupt 요청들을 순차적으로 처리하기 위한 별도의 칩이 있습니다. 요즘 기계에는 `APIC`라고 불리는 [Advanced Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)가 있습니다. `APIC`는 두개의 분리된 장치로 구성됩니다. 

* `Local APIC`
* `I/O APIC`

The first - `Local APIC`는 각 CPU 코어에 있습니다. local APIC는 CPU-특정적인(CPU-Specific) interrupt 설정을 담당합니다. Local APIC는 보통 APIC-timer, 열 센서(thermal sensor) 그리고 기기에 직접 연결되어있는 I/O 장치에서 발생하는 interrupt를 다룹니다. 

The Second - `I/O APIC`는 다중 프로세서 interrupt 관리를 제공합니다. I/O APIC는 CPU 코어들 사이의 외부 interrupt들을 분산시키기 위해 사용됩니다. Local과 I/O APIC들에 대한 자세한 내용은 이번 챕터에서 다룰것입니다. 당신이 이해할 수 있듯이, interrupt는 언제든지 발생할 수 있습니다. Interrupt가 발생하면, 운영체제는 발생한 interrupt를 즉시 처리해야합니다. 그런데 `interrupt를 처리한다`라는 것의 의미는 무엇일까요? Interrupt가 발생할 때, 운영체제는 반드시 다음과 같은 과정을 따라야합니다. 

* 커널은 반드시 현재 프로세스의 실행을 멈춰야합니다.(현재 작업을 선점한다)
* 커널은 반드시 해당 Interrupt의 핸들러와 전송 컨트롤을 찾아야합니다.(Interrupt 핸들러를 실행한다)
* Interrupt 핸들러가 실행을 완료한 후, Interrupt로 인해 멈춰진 프로세스(Interrupted Process)는 실행을 재개 할 수 있습니다. 

물론, Interrupt들을 처리하는 과정에는 수많은 복잡한 사항들이 포함되어 있습니다. 그러나 위의 3 단계는 Interrupt 처리 과정의 기본적인 골격을 형성합니다. 

각 Interrupt 핸들러의 주소는 `Interrupt Descriptor Table` 또는 `IDT`라고 불리는 틀별한 장소에서 유지됩니다. 프로세서는 Interrupt나 Exception(예외)의 종류를 인식하기 위해 고유의 숫자를 사용합니다. 이 숫자를 `vector number`라고 부릅니다. Vector number는 `IDT`의 인덱스입니다. Vector number의 갯수는 `0` ~ `255`까지로 정해져있습니다. 리눅스 커널 소스코드에서 다음과 같이 vector number에 대한 범위 체크를 할 수 있습니다.   

```C
BUG_ON((unsigned)n > 0xFF);
```

Interrupt 셋업(예를 들면, [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h)내의 `set_intr_gate`, `void set_system_intr_gate`)와 연관된 리눅스 커널 소스에서 위의 확인코드를 찾을 수 있습니다. `0`부터 `31`까지의 첫 32개의 vector number는 프로세서에 의해 예약되어 있고, 아키텍처에 맞춰 디자인 된 Exception들과 interrupt들을 처리하기 위해 사용됩니다. 리눅스 커널 초기화의 두번째 파트-[Early interrupt and exception handling](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html)에서 이 vector number들을 설명하는 표를 확인 할 수 있습니다. `32`부터 `255`까지 vector number들은 사용자-정의 interrupts로 지정되어있고, 이들은 프로세서에 의해 예약되어있지 않습니다. 이러한 interrupt들은 일반적으로 외부 장치들이 프로세서에 interrupts를 보내는 것을 허용하기 위해 사용됩니다. 

자, 이제 Interrupts의 종류에 대해 이야기해봅시다. 넓은 의미에서 보자면, Interrupt는 2개의 주요 클래스로 나눠집니다:

* 외부 또는 하드웨어 생성 Interrupt
* 소프트웨어 생성 Interrupt

The first - 외부 Interrupt들은 `Local APIC` 또는 `Local APIC`와 연결된 프로세서의 핀들을 통해 유입됩니다. The second - 소프트웨어 생성 Interrupt들은 프로세서 스스로의 예외 조건들에 의해서 생성됩니다. (때로는 특별한 아키텍처-특화 명령어에 의해서도 생성됩니다). 예외 조건의 일반적인 사례는 `division by zero`입니다. 또 다른 예는 `syscall` 명령어를 통한 프르그램 종료입니다.

처음에 언급했던 것처럼, Interrupt는 코드와 CPU가 제어할 수 없는 이유로 언제든지 발생할 수 있습니다. 반면, 예외는 프로그램의 실행과 `동기화`되어있고, 또한 3개의 카테고리로 분류할 수 있습니다:

* `Faults`
* `Traps`
* `Aborts`

`fault`는 "faulty" 명령어(이 오류는 수정할 수 있습니다)가 실행되기 전에 보고되는 예외입니다. 오류가 수정되면, `fault`는 interrupt 된 프로그램을 다시 시작합니다.  
다음은 `trap`입니다. `trap`은 `trap` 명령어의 실행에 따라 즉시 보고되는 예외입니다. `trap`은 `fault`와 마찬가지로 interrupt된 프로그램을 다시 실행 시킬 수 있게합니다.  
마지막으로, `abort`는 이 예외를 발생시킨 명령어를 보고하지 않는 예외입니다. 그리고 `abort`는 interrupt 된 프로그램의 재실행을 허락하지 않습니다. 


우리는 이미 이전 [부분](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html)에서 interrupt들이 `maskable`과 `non-maskable`로 구분 될 수 있다는 것을 배웠습니다. Maskable interrupt는 `x86_64`구조에서 다음과 같은 두개에 명령어에 의해 차단 될 수 있는 interrupt입니다 - `sti`, `cli`. 이와 관련된 코드들을 리눅스 커널에서 찾을 수 있습니다. 

```C
static inline void native_irq_disable(void)
{
        asm volatile("cli": : :"memory");
}
```

and

```C
static inline void native_irq_enable(void)
{
        asm volatile("sti": : :"memory");
}
```
이 두 명령어는 interrupt 레지스터 내부의 `IF`(Interrupt Flag) 플래그 비트를 수정합니다. `sti` 명령어는 `IF` 플래그를 설정(1)하고, `cli` 명령어는 `IF` 플래그를 지웁니다(0). Non-maskable interrupt들은 항상 보고됩니다. 보통 하드웨어의 실패는 non-maskable interrupt로 맵핑됩니다.

만약 다중의 예와나 interrupt들이 동시에 발생하면, 프로세서는 이것들을 미리 정의되어있는 우선순위의 순서대로 처리합니다. 아래 표에 나와있듯, 가장 높은 우선순위부터 가장 낮은 우선순위를 아래 표에서 확인할 수 있습니다:

```
+----------------------------------------------------------------+
|              |                                                 |
|   Priority   | Description                                     |
|              |                                                 |
+--------------+-------------------------------------------------+
|              | Hardware Reset and Machine Checks               |
|     1        | - RESET                                         |
|              | - Machine Check                                 |
+--------------+-------------------------------------------------+
|              | Trap on Task Switch                             |
|     2        | - T flag in TSS is set                          |
|              |                                                 |
+--------------+-------------------------------------------------+
|              | External Hardware Interventions                 |
|              | - FLUSH                                         |
|     3        | - STOPCLK                                       |
|              | - SMI                                           |
|              | - INIT                                          |
+--------------+-------------------------------------------------+
|              | Traps on the Previous Instruction               |
|     4        | - Breakpoints                                   |
|              | - Debug Trap Exceptions                         |
+--------------+-------------------------------------------------+
|     5        | Nonmaskable Interrupts                          |
+--------------+-------------------------------------------------+
|     6        | Maskable Hardware Interrupts                    |
+--------------+-------------------------------------------------+
|     7        | Code Breakpoint Fault                           |
+--------------+-------------------------------------------------+
|     8        | Faults from Fetching Next Instruction           |
|              | Code-Segment Limit Violation                    |
|              | Code Page Fault                                 |
+--------------+-------------------------------------------------+
|              | Faults from Decoding the Next Instruction       |
|              | Instruction length > 15 bytes                   |
|     9        | Invalid Opcode                                  |
|              | Coprocessor Not Available                       |
|              |                                                 |
+--------------+-------------------------------------------------+
|     10       | Faults on Executing an Instruction              |
|              | Overflow                                        |
|              | Bound error                                     |
|              | Invalid TSS                                     |
|              | Segment Not Present                             |
|              | Stack fault                                     |
|              | General Protection                              |
|              | Data Page Fault                                 |
|              | Alignment Check                                 |
|              | x87 FPU Floating-point exception                |
|              | SIMD floating-point exception                   |
|              | Virtualization exception                        |
+--------------+-------------------------------------------------+
```

지금까지 다양한 종류의 interrupts와 예외에 대해 가볍게 살펴봤다면, 이제는 좀 더 실용적인 부분으로 넘어갈 차례입니다. `Interrupt Descriptor Table`을 먼저 알아보겠습니다. 이전에 언급했듯이, `IDT`는 interrupt들과 예외 핸들러들의 엔트리 포인트를 저장합니다. `IDT`는 [Kernel booting process](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)의 두번째 부분에서 봤던 `Global Descriptor Table`과 비슷한 구조를 가지고 있습니다. 물론 차이점도 존재합니다. `descriptors` 대산, `IDT` 엔트리들은 `gates`라고 불립니다. `gates`는 다음 게이트 들 중 하나를 포함할 수 있습니다:

* Interrupt gates
* Task gates
* Trap gates.

in the `x86` architecture. Only [long mode](http://en.wikipedia.org/wiki/Long_mode) interrupt gates and trap gates can be referenced in the `x86_64`. Like the `Global Descriptor Table`, the `Interrupt Descriptor table` is an array of 8-byte gates on `x86` and an array of 16-byte gates on `x86_64`. We can remember from the second part of the [Kernel booting process](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html), that `Global Descriptor Table` must contain `NULL` descriptor as its first element. Unlike the `Global Descriptor Table`, the `Interrupt Descriptor Table` may contain a gate; it is not mandatory. For example, you may remember that we have loaded the Interrupt Descriptor table with the `NULL` gates only in the earlier [part](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-3.html) while transitioning into [protected mode](http://en.wikipedia.org/wiki/Protected_mode):

```C
/*
 * Set up the IDT
 */
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

from the [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/pm.c). The `Interrupt Descriptor table` can be located anywhere in the linear address space and the base address of it must be aligned on an 8-byte boundary on `x86` or 16-byte boundary on `x86_64`. The base address of the `IDT` is stored in the special register - `IDTR`. There are two instructions on `x86`-compatible processors to modify the `IDTR` register:

* `LIDT`
* `SIDT`

The first instruction `LIDT` is used to load the base-address of the `IDT` i.e., the specified operand into the `IDTR`. The second instruction `SIDT` is used to read and store the contents of the `IDTR` into the specified operand. The `IDTR` register is 48-bits on the `x86` and contains the following information:

```
+-----------------------------------+----------------------+
|                                   |                      |
|     Base address of the IDT       |   Limit of the IDT   |
|                                   |                      |
+-----------------------------------+----------------------+
47                                16 15                    0
```

Looking at the implementation of `setup_idt`, we have prepared a `null_idt` and loaded it to the `IDTR` register with the `lidt` instruction. Note that `null_idt` has `gdt_ptr` type which is defined as:

```C
struct gdt_ptr {
        u16 len;
        u32 ptr;
} __attribute__((packed));
```

Here we can see the definition of the structure with the two fields of 2-bytes and 4-bytes each (a total of 48-bits) as we can see in the diagram. Now let's look at the `IDT` entries structure. The `IDT` entries structure is an array of the 16-byte entries which are called gates in the `x86_64`. They have the following structure:

```
127                                                                             96
+-------------------------------------------------------------------------------+
|                                                                               |
|                                Reserved                                       |
|                                                                               |
+--------------------------------------------------------------------------------
95                                                                              64
+-------------------------------------------------------------------------------+
|                                                                               |
|                               Offset 63..32                                   |
|                                                                               |
+-------------------------------------------------------------------------------+
63                               48 47      46  44   42    39             34    32
+-------------------------------------------------------------------------------+
|                                  |       |  D  |   |     |      |   |   |     |
|       Offset 31..16              |   P   |  P  | 0 |Type |0 0 0 | 0 | 0 | IST |
|                                  |       |  L  |   |     |      |   |   |     |
 -------------------------------------------------------------------------------+
31                                   16 15                                      0
+-------------------------------------------------------------------------------+
|                                      |                                        |
|          Segment Selector            |                 Offset 15..0           |
|                                      |                                        |
+-------------------------------------------------------------------------------+
```

To form an index into the IDT, the processor scales the exception or interrupt vector by sixteen. The processor handles the occurrence of exceptions and interrupts just like it handles calls of a procedure when it sees the `call` instruction. A processor uses a unique number or `vector number` of the interrupt or the exception as the index to find the necessary `Interrupt Descriptor Table` entry. Now let's take a closer look at an `IDT` entry.

As we can see, `IDT` entry on the diagram consists of the following fields:

* `0-15` bits  - offset from the segment selector which is used by the processor as the base address of the entry point of the interrupt handler;
* `16-31` bits - base address of the segment select which contains the entry point of the interrupt handler;
* `IST` - a new special mechanism in the `x86_64`, will see it later;
* `DPL` - Descriptor Privilege Level;
* `P` - Segment Present flag;
* `48-63` bits - second part of the handler base address;
* `64-95` bits - third part of the base address of the handler;
* `96-127` bits - and the last bits are reserved by the CPU.

And the last `Type` field describes the type of the `IDT` entry. There are three different kinds of handlers for interrupts:

* Interrupt gate
* Trap gate
* Task gate

The `IST` or `Interrupt Stack Table` is a new mechanism in the `x86_64`. It is used as an alternative to the legacy stack-switch mechanism. Previously the `x86` architecture provided a mechanism to automatically switch stack frames in response to an interrupt. The `IST` is a modified version of the `x86` Stack switching mode. This mechanism unconditionally switches stacks when it is enabled and can be enabled for any interrupt in the `IDT` entry related with the certain interrupt (we will soon see it). From this we can understand that `IST` is not necessary for all interrupts. Some interrupts can continue to use the legacy stack switching mode. The `IST` mechanism provides up to seven `IST` pointers in the [Task State Segment](http://en.wikipedia.org/wiki/Task_state_segment) or `TSS` which is the special structure which contains information about a process. The `TSS` is used for stack switching during the execution of an interrupt or exception handler in the Linux kernel. Each pointer is referenced by an interrupt gate from the `IDT`.

The `Interrupt Descriptor Table` represented by the array of the `gate_desc` structures:

```C
extern gate_desc idt_table[];
```

where `gate_desc` is:

```C
#ifdef CONFIG_X86_64
...
...
...
typedef struct gate_struct64 gate_desc;
...
...
...
#endif
```

and `gate_struct64` defined as:

```C
struct gate_struct64 {
        u16 offset_low;
        u16 segment;
        unsigned ist : 3, zero0 : 5, type : 5, dpl : 2, p : 1;
        u16 offset_middle;
        u32 offset_high;
        u32 zero1;
} __attribute__((packed));
```

Each active thread has a large stack in the Linux kernel for the `x86_64` architecture. The stack size is defined as `THREAD_SIZE` and is equal to:

```C
#define PAGE_SHIFT      12
#define PAGE_SIZE       (_AC(1,UL) << PAGE_SHIFT)
...
...
...
#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

The `PAGE_SIZE` is `4096`-bytes and the `THREAD_SIZE_ORDER` depends on the `KASAN_STACK_ORDER`. As we can see, the `KASAN_STACK` depends on the `CONFIG_KASAN` kernel configuration parameter and is defined as:

```C
#ifdef CONFIG_KASAN
    #define KASAN_STACK_ORDER 1
#else
    #define KASAN_STACK_ORDER 0
#endif
```

`KASan` is a runtime memory [debugger](http://lwn.net/Articles/618180/). Thus, the `THREAD_SIZE` will be `16384` bytes if `CONFIG_KASAN` is disabled or `32768` if this kernel configuration option is enabled. These stacks contain useful data as long as a thread is alive or in a zombie state. While the thread is in user-space, the kernel stack is empty except for the `thread_info` structure (details about this structure are available in the fourth [part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html) of the Linux kernel initialization process) at the bottom of the stack. The active or zombie threads aren't the only threads with their own stack. There also exist specialized stacks that are associated with each available CPU. These stacks are active when the kernel is executing on that CPU. When the user-space is executing on the CPU, these stacks do not contain any useful information. Each CPU has a few special per-cpu stacks as well. The first is the `interrupt stack` used for the external hardware interrupts. Its size is determined as follows:

```C
#define IRQ_STACK_ORDER (2 + KASAN_STACK_ORDER)
#define IRQ_STACK_SIZE (PAGE_SIZE << IRQ_STACK_ORDER)
```

or `16384` bytes. The per-cpu interrupt stack represented by the `irq_stack_union` union in the Linux kernel for `x86_64`:

```C
union irq_stack_union {
	char irq_stack[IRQ_STACK_SIZE];

    struct {
		char gs_base[40];
		unsigned long stack_canary;
	};
};
```

The first `irq_stack` field is a 16 kilobytes array. Also you can see that `irq_stack_union` contains a structure with the two fields:

* `gs_base` - The `gs` register always points to the bottom of the `irqstack` union. On the `x86_64`, the `gs` register is shared by per-cpu area and stack canary (more about `per-cpu` variables you can read in the special [part](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)).  All per-cpu symbols are zero based and the `gs` points to the base of the per-cpu area. You already know that [segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation) is abolished in the long mode, but we can set the base address for the two segment registers - `fs` and `gs` with the [Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register) and these registers can be still be used as address registers. If you remember the first [part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html) of the Linux kernel initialization process, you can remember that we have set the `gs` register:

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

where `initial_gs` points to the `irq_stack_union`:

```assembly
GLOBAL(initial_gs)
.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

* `stack_canary` - [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries) for the interrupt stack is a `stack protector`
to verify that the stack hasn't been overwritten. Note that `gs_base` is a 40 bytes array. `GCC` requires that stack canary will be on the fixed offset from the base of the `gs` and its value must be `40` for the `x86_64` and `20` for the `x86`.

The `irq_stack_union` is the first datum in the `percpu` area, we can see it in the `System.map`:

```
0000000000000000 D __per_cpu_start
0000000000000000 D irq_stack_union
0000000000004000 d exception_stacks
0000000000009000 D gdt_page
...
...
...
```

We can see its definition in the code:

```C
DECLARE_PER_CPU_FIRST(union irq_stack_union, irq_stack_union) __visible;
```

Now, it's time to look at the initialization of the `irq_stack_union`. Besides the `irq_stack_union` definition, we can see the definition of the following per-cpu variables in the [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/processor.h):

```C
DECLARE_PER_CPU(char *, irq_stack_ptr);
DECLARE_PER_CPU(unsigned int, irq_count);
```

The first is the `irq_stack_ptr`. From the variable's name, it is obvious that this is a pointer to the top of the stack. The second - `irq_count` is used to check if a CPU is already on an interrupt stack or not. Initialization of the `irq_stack_ptr` is located in the `setup_per_cpu_areas` function in [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup_percpu.c):

```C
void __init setup_per_cpu_areas(void)
{
...
...
#ifdef CONFIG_X86_64
for_each_possible_cpu(cpu) {
    ...
    ...
    ...
    per_cpu(irq_stack_ptr, cpu) =
            per_cpu(irq_stack_union.irq_stack, cpu) +
            IRQ_STACK_SIZE - 64;
    ...
    ...
    ...
#endif
...
...
}
```

Here we go over all the CPUs one-by-one and setup `irq_stack_ptr`. This turns out to be equal to the top of the interrupt stack minus `64`. Why `64`?TODO  [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c) source code file is following:

```C
void load_percpu_segment(int cpu)
{
        ...
        ...
        ...
        loadsegment(gs, 0);
        wrmsrl(MSR_GS_BASE, (unsigned long)per_cpu(irq_stack_union.gs_base, cpu));
}
```

and as we already know the `gs` register points to the bottom of the interrupt stack.

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr

	GLOBAL(initial_gs)
	.quad	INIT_PER_CPU_VAR(irq_stack_union)
```

Here we can see the `wrmsr` instruction which loads the data from `edx:eax` into the [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register) pointed by the `ecx` register. In our case the model specific register is `MSR_GS_BASE` which contains the base address of the memory segment pointed by the `gs` register. `edx:eax` points to the address of the `initial_gs` which is the base address of our `irq_stack_union`.

We already know that `x86_64` has a feature called `Interrupt Stack Table` or `IST` and this feature provides the ability to switch to a new stack for events non-maskable interrupt, double fault etc. There can be up to seven `IST` entries per-cpu. Some of them are:

* `DOUBLEFAULT_STACK`
* `NMI_STACK`
* `DEBUG_STACK`
* `MCE_STACK`

or

```C
#define DOUBLEFAULT_STACK 1
#define NMI_STACK 2
#define DEBUG_STACK 3
#define MCE_STACK 4
```

All interrupt-gate descriptors which switch to a new stack with the `IST` are initialized with the `set_intr_gate_ist` function. For example:

```C
set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
...
...
...
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

where `&nmi` and `&double_fault` are addresses of the entries to the given interrupt handlers:

```C
asmlinkage void nmi(void);
asmlinkage void double_fault(void);
```

defined in the [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/entry_64.S)

```assembly
idtentry double_fault do_double_fault has_error_code=1 paranoid=2
...
...
...
ENTRY(nmi)
...
...
...
END(nmi)
```

When an interrupt or an exception occurs, the new `ss` selector is forced to `NULL` and the `ss` selector’s `rpl` field is set to the new `cpl`. The old `ss`, `rsp`, register flags, `cs`, `rip` are pushed onto the new stack. In 64-bit mode, the size of interrupt stack-frame pushes is fixed at 8-bytes, so we will get the following stack:

```
+---------------+
|               |
|      SS       | 40
|      RSP      | 32
|     RFLAGS    | 24
|      CS       | 16
|      RIP      | 8
|   Error code  | 0
|               |
+---------------+
```

If the `IST` field in the interrupt gate is not `0`, we read the `IST` pointer into `rsp`. If the interrupt vector number has an error code associated with it, we then push the error code onto the stack. If the interrupt vector number has no error code, we go ahead and push the dummy error code on to the stack. We need to do this to ensure stack consistency. Next, we load the segment-selector field from the gate descriptor into the CS register and must verify that the target code-segment is a 64-bit mode code segment by the checking bit `21` i.e. the `L` bit in the `Global Descriptor Table`. Finally we load the offset field from the gate descriptor into `rip` which will be the entry-point of the interrupt handler. After this the interrupt handler begins to execute and when the interrupt handler finishes its execution, it must return control to the interrupted process with the `iret` instruction. The `iret` instruction unconditionally pops the stack pointer (`ss:rsp`) to restore the stack of the interrupted process and does not depend on the `cpl` change.

That's all.

Conclusion
--------------------------------------------------------------------------------

It is the end of the first part of `Interrupts and Interrupt Handling` in the Linux kernel. We covered some theory and the first steps of initialization of stuffs related to interrupts and exceptions. In the next part we will continue to dive into the more practical aspects of interrupts and interrupt handling.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me a PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [PIC](http://en.wikipedia.org/wiki/Programmable_Interrupt_Controller)
* [Advanced Programmable Interrupt Controller](https://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [long mode](http://en.wikipedia.org/wiki/Long_mode)
* [kernel stacks](https://www.kernel.org/doc/Documentation/x86/kernel-stacks)
* [Task State Segment](http://en.wikipedia.org/wiki/Task_state_segment)
* [segmented memory model](http://en.wikipedia.org/wiki/Memory_segmentation)
* [Model specific registers](http://en.wikipedia.org/wiki/Model-specific_register)
* [Stack canary](http://en.wikipedia.org/wiki/Stack_buffer_overflow#Stack_canaries)
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)
