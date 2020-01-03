인터럽트와 인터럽트 처리. Part 4.
================================================================================

비초기(non-early) 인터럽트 게이트의 초기화
--------------------------------------------------------------------------------

이것은 리눅스 커널에서의 인터럽트와 예외 처리에 관한 네 번째 파트입니다. 그리고 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)에서 우리는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에서 처음의 초기 `#DB` 및`#BP` 예외 처리기(exception handler)를 보았습니다. 우리는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/setup.c)에 정의 된 `setup_arch` 함수에서 호출 된 `early_trap_init` 함수 바로 뒤에서 멈췄었습니다. 이번 파트에서 우리는 지난 파트에서 멈춘 곳부터 이어서, 계속 `x86_64` 리눅스 커널에서 인터럽트와 예외 처리에 대해 계속해서 배울 것입니다. 인터럽트 및 예외 처리와 관련한 첫 번째는 `early_trap_pf_init` 함수를 사용하여 `# PF` 또는 [page fault](https://en.wikipedia.org/wiki/Page_fault) 핸들러를 설정하는 것입니다. 그럼 시작해봅시다.

초기 페이지 폴트 핸들러(Early page fault handler)
--------------------------------------------------------------------------------

`early_trap_pf_init` 함수는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에 정의되어 있습니다. 이 함수는 주어진 엔트리로 [Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)을 채우는`set_intr_gate` 매크로를 사용합니다 :

```C
void __init early_trap_pf_init(void)
{
#ifdef CONFIG_X86_64
         set_intr_gate(X86_TRAP_PF, page_fault);
#endif
}
```

이 매크로는 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/desc.h)에 정의되어 있습니다. 우리는 이미 지난 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)에서 `set_system_intr_gate`와  `set_intr_gate_ist` 등, 이와 같은 매크로를 보았습니다. 이 매크로는 주어진 벡터 번호가 `255` (최대 벡터 번호)보다 크지 않은지 확인하고 `set_system_intr_gate`와 `set_intr_gate_ist`와 같이 `_set_gate` 함수를 호출합니다.

```C
#define set_intr_gate(n, addr)                                  \
do {                                                            \
        BUG_ON((unsigned)n > 0xFF);                             \
        _set_gate(n, GATE_INTERRUPT, (void *)addr, 0, 0,        \
                  __KERNEL_CS);                                 \
        _trace_set_gate(n, GATE_INTERRUPT, (void *)trace_##addr,\
                        0, 0, __KERNEL_CS);                     \
} while (0)
```

`set_intr_gate` 는 두개의 매개 변수를 갖습니다:

* 인터럽트의 벡터 번호;
* 인터럽트 핸들러의 주소;

이 경우에 이 둘은 다음과 같습니다:

* `X86_TRAP_PF` - `14`;
* `page_fault` - 인터럽트 핸들러의 엔트리 포인트.

`X86_TRAP_PF` 는 열거형(enum)의 원소이며 이는 [arch/x86/include/asm/traprs.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/traprs.h)에 정의되어 있습니다:

```C
enum {
	...
	...
	...
	...
	X86_TRAP_PF,            /* 14, Page Fault */
	...
	...
	...
}
```

`early_trap_pf_init`가 호출 될 때, `set_intr_gate`가 `_set_gate`의 호출로 확장되며, `_set_gate`는 `IDT`를 페이지 결함에 대한 핸들러로 채웁니다. 이제`page_fault` 핸들러의 구현을 살펴 봅시다. `page_fault` 핸들러는 다른 모든 예외 처리기와 마찬가지로 [arch/x86/kernel/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/entry_64.S) 어셈블리 소스 코드 파일에 정의되어 있습니다. 한 번 살펴봅시다:

```assembly
trace_idtentry page_fault do_page_fault has_error_code=1
```

앞의 [part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)에서 `#DB`와`#BP` 핸들러가 어떻게 정의되는지 보았습니다. 그 둘은 `idtentry` 매크로로 정의되었지만 여기서는 `trace_idtentry`로 정의된 것을 볼 수 있습니다. 이 매크로는  `CONFIG_TRACING` 커널 구성 옵션에 의존하며 동일한 소스 코드 파일에 정의되어 있습니다.

```assembly
#ifdef CONFIG_TRACING
.macro trace_idtentry sym do_sym has_error_code:req
idtentry trace(\sym) trace(\do_sym) has_error_code=\has_error_code
idtentry \sym \do_sym has_error_code=\has_error_code
.endm
#else
.macro trace_idtentry sym do_sym has_error_code:req
idtentry \sym \do_sym has_error_code=\has_error_code
.endm
#endif
```

우리는 지금 [Tracing](https://en.wikipedia.org/wiki/Tracing_%28software%29) 예외를 다루지는 않을 것입니다. 만약 `CONFIG_TRACING`이 설정되어 있지 않으면 `trace_idtentry` 매크로가 그저 일반적인 `idtentry`로 확장되는 것을 볼 수 있습니다. 우리는 이미 이전의 [part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)에서 `idtentry` 매크로의 구현을 보았으므로 `page_fault` 예외 처리기부터 시작해봅시다.

`idtentry`의 정의에서 볼 수 있듯이, `page_fault`의 처리기(handler)는 [arch/x86/mm/fault.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/fault.c)에 정의 된 `do_page_fault` 함수이며 모든 예외 처리기와 같이 두 가지 인수를 가집니다.

* `regs` - 중단된 프로세스의 상태를 가지고 있는 `pt_regs`구조체;
* `error_code` - 페이지 폴트 예외의 오류 코드.

이 함수에 대해 자세히 들여다봅시다. 우선 [cr2](https://en.wikipedia.org/wiki/Control_register) 제어 레지스터의 내용을 읽어봅니다.

```C
dotraplinkage void notrace
do_page_fault(struct pt_regs *regs, unsigned long error_code)
{
	unsigned long address = read_cr2();
	...
	...
	...
}
```

이 레지스터는 `page fault`를 일으킨 선형(linear) 주소를 가지고 있습니다. 다음 단계에서는 [include/linux/context_tracking.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/context_tracking.h)에서 `exception_enter` 함수를 호출합니다. `exception_enter` 및 `exception_exit`는 프로세서가 사용자 공간에서 실행되는 동안 타이머 틱에 대한 의존성을 제거하기 위해 [RCU](https://en.wikipedia.org/wiki/Read-copy-update)에서 사용하는 리눅스 커널의 컨텍스트 추적 서브시스템(context tracking subsystem)의 함수입니다. 거의 모든 예외 처리기에서 다음과 같이 비슷한 코드를 볼 수 있습니다.

```C
enum ctx_state prev_state;
prev_state = exception_enter();
...
... // exception handler here
...
exception_exit(prev_state);
```

`exception_enter`함수는 `context_tracking_is_enabled`로 `context tracking`이 활성화되어 있는지 확인하고 활성화 된 상태이면 `this_cpu_read`와 함께 이전 컨텍스트를 얻습니다. (`this_cpu_*` 작업에 대한 자세한 내용은 [문서](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt)를 참고하세요). 그런 다음 컨텍스트 추적에 프로세서가 사용자공간 모드(userspace mode)를 종료하고 커널에 들어가고 있음을 알리는`context_tracking_user_exit` 함수를 호출합니다.

```C
static inline enum ctx_state exception_enter(void)
{
        enum ctx_state prev_ctx;

        if (!context_tracking_is_enabled())
                return 0;

        prev_ctx = this_cpu_read(context_tracking.state);
        context_tracking_user_exit();

        return prev_ctx;
}
```

상태(state)는 다음 중 하나입니다.

```C
enum ctx_state {
    IN_KERNEL = 0,
	IN_USER,
} state;
```

그리고 마지막에는 이전 문맥을 반환합니다. `exception_enter`와 `exception_exit` 사이에서 우리는 실제 페이지 오류 처리기를 호출합니다.

```C
__do_page_fault(regs, error_code, address);
```

`__do_page_fault`는 `do_page_fault`와 동일한 소스 코드 파일-[arch/x86/mm/fault.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/fault.c)-에 정의되어 있습니다. `__do_page_fault`의 시작 부분에선 [kmemcheck](https://www.kernel.org/doc/Documentation/kmemcheck.txt) 검사기(checker)의 상태를 확인합니다. `kmemcheck`는 일부 초기화되지 않은 메모리의 사용에 대한 경고를 탐지합니다. kmemcheck로 인해 페이지 폴트가 발생할 수 있으므로 이에 대해서 확인해봐야합니다.

```C
if (kmemcheck_active(regs))
		kmemcheck_hide(regs);
	prefetchw(&mm->mmap_sem);
```

그 다음으로 동일한 [name](http://www.felixcloutier.com/x86/PREFETCHW.html) (이것은 독점적인 캐시 라인([cache line](https://en.wikipedia.org/wiki/CPU_cache))을 얻기 위해 [X86_FEATURE_3DNOW](https://en.wikipedia.org/?title=3DNow!)를 fetch합니다) 으로 명령을 실행하는`prefetchw`를 호출하는 것을 볼 수 있습니다. After this we can see the call of the `prefetchw` which executes instruction with the same [name](http://www.felixcloutier.com/x86/PREFETCHW.html) which fetches [X86_FEATURE_3DNOW](https://en.wikipedia.org/?title=3DNow!) to get exclusive [cache line](https://en.wikipedia.org/wiki/CPU_cache). 미리 페치하는 것(prefetching)의 주요 목적은 메모리 액세스의 대기 시간을 '숨기는' 것입니다. 다음 단계에서는 다음 조건에서 커널 공간에 페이지 오류가 없는지 확인합니다.

```C
if (unlikely(fault_in_kernel_space(address))) {
...
...
...
}
```

여기서 `fault_in_kernel_space` 는:

```C
static int fault_in_kernel_space(unsigned long address)
{
        return address >= TASK_SIZE_MAX;
}
```

`TASK_SIZE_MAX` 매크로는 다음과 같이 확장되거나:

```C
#define TASK_SIZE_MAX   ((1UL << 47) - PAGE_SIZE)
```

`0x00007ffffffff000`로 확장됩니다. `unlikely` 매크로에 주의하십시오. Linux 커널에는 두 가지 매크로가 있습니다.

```C
#define likely(x)      __builtin_expect(!!(x), 1)
#define unlikely(x)    __builtin_expect(!!(x), 0)
```

리눅스 커널 코드에서 [종종](http://lxr.free-electrons.com/ident?i=unlikely) 이러한 매크로를 찾을 수 있습니다. 이러한 매크로의 주요 목적은 최적화입니다. 때때로 이런 상황에선 코드의 상태를 점검해야하며, `true`이거나 또는 `false`인 경우는 거의 없습니다. 이러한 매크로를 사용하면 컴파일러에 이러한 것을 알릴 수 있습니다. 가령 예를들어,

```C
static int proc_root_readdir(struct file *file, struct dir_context *ctx)
{
        if (ctx->pos < FIRST_PROCESS_ENTRY) {
                int error = proc_readdir(file, ctx);
                if (unlikely(error <= 0))
                        return error;
...
...
...
}
```

여기 리눅스 [VFS](https://en.wikipedia.org/wiki/Virtual_file_system)가 `root` 디렉토리 내용을 읽어야 하는 상황에서 호출되는 `proc_root_readdir` 함수가 있습니다. 조건이 `unlikely`로 표시된 경우, 컴파일러는 분기(branch) 바로 다음에 `false`코드를 넣을 수 있습니다. 이제 주소 확인으로 돌아 갑시다. 주어진 주소와 `0x00007ffffffff000`를 비교하는 것으로 커널 모드 또는 사용자 모드에서 페이지 폴트가 있었음을 알 수 있습니다. 이 확인으로 우리는 그것을 알게 되었습니다. 이 `__do_page_fault` 루틴은 페이지 폴트 예외를 일으킨 문제에 대한 이해를 시도한 다음 적절한 루틴으로 주소를 전달합니다. 문제는 `kmemcheck` 오류, spurious 오류, [kprobes](https://www.kernel.org/doc/Documentation/kprobes.txt) 오류 등등일 수 있습니다.  리눅스 커널에서 제공하는 다양한 개념을 알아봐야 하기 때문에 이번 파트에서는 페이지 결함 예외 처리기의 구현 세부 사항에 대해서는 다루지 않을 것이며, 나중에 리눅스 커널의 [메모리 관리](https://0xax.gitbooks.io/linux-insides/content/MM/index.html) 챕터에서 자세히 다루도록 하겠습니다. 

start_kernel로 돌아가기
--------------------------------------------------------------------------------

서로 다른 커널 서브 시스템에서 `setup_arch` 함수의  `early_trap_pf_init` 다음에 다른 함수 호출은 많이 있지만 인터럽트와 예외 처리와 관련된 호출은 없습니다. 그러므로 우리와 왔던 곳 -[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c#L492)의 `start_kernel` 함수-으로 돌아 가야합니다. `setup_arch` 바로 다음은 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)의`trap_init` 함수입니다. 이 함수는 남은 예외 처리기를 초기화합니다(우리가 이미 `#DB`-디버그 예외,`#BP`-중단 점 예외 및`#PF`-페이지 결함 예외에 대한 3개의 핸들러를 설정했던 것을 기억하십시오). `trap_init` 함수는 EISA([Extended Industry Standard Architecture](https://en.wikipedia.org/wiki/Extended_Industry_Standard_Architecture))의 점검에서부터 시작됩니다.

```C
#ifdef CONFIG_EISA
        void __iomem *p = early_ioremap(0x0FFFD9, 4);

        if (readl(p) == 'E' + ('I'<<8) + ('S'<<16) + ('A'<<24))
                EISA_bus = 1;
        early_iounmap(p, 4);
#endif
```

이 함수는 EISA 지원을 나타내는 `CONFIG_EISA` 커널 구성 매개 변수에 의존하는 것에 유의하십시오. 여기서 `early_ioremap` 함수를 사용하여 페이지 테이블에서`I/O` 메모리를 매핑합니다. 우리는`readl` 함수를 사용하여 매핑 된 영역에서 첫  `4` 바이트를 읽어들입니다. 그리고 그들이 `EISA`문자열과 같으면 `EISA_bus`를 1로 설정합니다. 마지막에는 우리는 이전에 매핑 된 영역을 매핑 해제합니다. `early_ioremap`에 대한 자세한 내용은 [Fix-Mapped Addresses and ioremap](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-2.html)을 설명하는 파트에서 읽을 수 있습니다.

다음으로 서로 다른 인터럽트 게이트들로 '인터럽터 디스크립터 테이블'(`Interrupt Descriptor Table`)을 채우기 시작합니다. 우선, `# DE`  또는 `Divide Error` 그리고 `# NMI`  또는 `Non-maskable Interrupt`를 설정합니다 :

```C
set_intr_gate(X86_TRAP_DE, divide_error);
set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
```

`set_intr_gate` 매크로를 사용하여 `#DE` 예외의 인터럽트 게이트를 설정하고, `set_intr_gate_ist`매크로를 사용하여 `#NMI`예외의 인터럽트 게이트를 설정합니다. 페이지 폴트 처리기, 디버그 처리기 등에 대한 인터럽트 게이트를 설정할 때 이미 이러한 매크로를 사용했음을 기억하십시오. 이에 대한 설명은 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)에서 찾아볼 수 있습니다. 그 다음은 다음 예외에 대해 예외 게이트를 설정합니다.

```C
set_system_intr_gate(X86_TRAP_OF, &overflow);
set_intr_gate(X86_TRAP_BR, bounds);
set_intr_gate(X86_TRAP_UD, invalid_op);
set_intr_gate(X86_TRAP_NM, device_not_available);
```

여기서 다음과 같은 것들을 확인할 수 있습니다:

* `#OF` 또는 `Overflow` 예외. 이 예외는 특수한 [INTO](http://x86.renejeschke.de/html/file_module_x86_id_142.html) 명령이 실행되었을 때 오버 플로우 트랩이 발생했음을 나타냅니다.;
* `#BR` 또는 `BOUND Range exceeded` 예외. 이 예외는 [BOUND](http://pdos.csail.mit.edu/6.828/2005/readings/i386/BOUND.htm) 명령이 실행될 때 `BOUND-range-exceed` 오류가 발생했음을 나타냅니다;
* `#UD` 또는 `Invalid Opcode` 예외. 프로세서가 유효하지 않거나 예약 된 [opcode](https://en.wikipedia.org/?title=Opcode)를 실행하려고 시도하거나 프로세서가 유효하지 않은 피연산자 등을 사용하여 명령을 실행하려고 할 때 등의 상황에서 발생합니다;
* `#NM` 또는 `Device Not Available` 예외. 프로세서가 [control register](https://en.wikipedia.org/wiki/Control_register#CR0) `cr0`에 `EM` 플래그가 설정되어 있는 동안 `x87 FPU` 부동 소수점 명령을 실행하려고 할 때 발생합니다.

다음 단계에서 `# DF` 또는 `Double fault` 예외에 대한 인터럽트 게이트를 설정합니다 :

```C
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

이 예외는 프로세서가 이전 예외에 대한 예외 핸들러를 호출하는 동안 두 번째 예외를 감지 한 경우 발생합니다. 보통은 프로세서가 예외 처리기를 호출하는 동안 다른 예외를 감지하면 두 예외를 순차적으로 처리 할 수 있습니다. 그러나 만약 프로세서가 순차적으로 처리 할 수 없으면 이중 오류(double-fault) 또는 `#DF` 예외를 알립니다.

그 다음으로 오는 인터럽트 게이트들은 다음과 같습니다.

```C
set_intr_gate(X86_TRAP_OLD_MF, &coprocessor_segment_overrun);
set_intr_gate(X86_TRAP_TS, &invalid_TSS);
set_intr_gate(X86_TRAP_NP, &segment_not_present);
set_intr_gate_ist(X86_TRAP_SS, &stack_segment, STACKFAULT_STACK);
set_intr_gate(X86_TRAP_GP, &general_protection);
set_intr_gate(X86_TRAP_SPURIOUS, &spurious_interrupt_bug);
set_intr_gate(X86_TRAP_MF, &coprocessor_error);
set_intr_gate(X86_TRAP_AC, &alignment_check);
```

여기서 다음으로 오는 예외 핸들러들의 설정을 확인할 수 있습니다.

* `#CSO` 또는 `Coprocessor Segment Overrun` - 이 예외는 이전 프로세서의 수학 코프로세서([coprocessor](https://en.wikipedia.org/wiki/Coprocessor))가 페이지 또는 세그먼트 위반을 감지했음을 나타냅니다. 최신 프로세서들은 이 예외를 생성하지 않습니다
* `#TS` 또는 `Invalid TSS` 예외 - 작업 상태 세그먼트([Task Sate Segment](https://en.wikipedia.org/wiki/Task_state_segment))와 관련된 오류가 있음을 나타냅니다.
* `# NP` 또는`Segment Not Present` 예외는`cs`,`ds`,`es`,`fs` 또는 `gs` 레지스터 중 하나를 로드하려고 시도하는 동안 세그먼트 또는 게이트 디스크립터의 `present flag`가 clear임을 나타냅니다. 
* `# SS` 또는 `Stack Fault` 예외는 스택과 관련 조건 중 하나가 감지되었음을 나타냅니다. 예를 들어 `ss` 레지스터를 로드하려고 할 때 존재하지 않는 스택 세그먼트가 감지되는 것이 있습니다.
* `# GP` 또는`General Protection` 예외는 프로세서가 일반 보호 위반(general-protection violations)이라고하는 보호 위반 클래스 중 하나를 감지했음을 나타냅니다. General-protection 예외를 일으킬 수 있는 조건에는 여러 가지가 있습니다. 예를 들어, 시스템 세그먼트의 세그먼트 선택기로 `ss`, `ds`, `es`, `fs` 또는 `gs` 레지스터를 로드하여 코드 세그먼트 또는 읽기 전용인 데이터 세그먼트에 쓰기를 한다거나, 인터럽트, 트랩 또는 작업 게이트 등이 아닌 (인터럽트 또는 예외 뒤에 따라오는) '인터럽트 디스크립터 테이블'(`Interrupt Descriptor Table`)의 항목을 참조하거나 그 외에도 아주 많습니다.
* `Spurious Interrupt` - 원하지 않는 하드웨어 인터럽트.
* `#MF` 또는 `x87 FPU Floating-Point Error` 예외는 [x87 FPU](https://en.wikipedia.org/wiki/X86_instruction_listings#x87_floating-point_instructions)에서 부동 소수점 오류가 감지된 경우에 발생합니다.
* `#AC` 또는 `Alignment Check` 예외는 정렬 검사(alignment checking)가 활성화되었을 때 프로세서가 정렬되지 않은 메모리 피연산자를 감지했음을 나타냅니다.

이러한 예외 게이트의 설정 뒤에는 `Machine-Check` 예외의 설정이 있습니다.

```C
#ifdef CONFIG_X86_MCE
	set_intr_gate_ist(X86_TRAP_MC, &machine_check, MCE_STACK);
#endif
```

`CONFIG_X86_MCE` 커널 구성 옵션에 따라 달라지며 프로세서가 내부 [machine error](https://en.wikipedia.org/wiki/Machine-check_exception)나 버스 오류를 감지했거나 외부 에이전트가 버스 오류를 감지했음을 나타낸다는 것에 유의하세요. 다음 예외 게이트는 [SIMD](https://en.wikipedia.org/?title=SIMD) 부동 소수점 예외입니다.

```C
set_intr_gate(X86_TRAP_XF, &simd_coprocessor_error);
```

이것은 프로세서가`SSE` 또는`SSE2` 또는`SSE3` SIMD 부동 소수점 예외를 감지했음을 나타냅니다. SIMD 부동 소수점 명령어를 실행하는 동안 발생할 수있는 6 가지 숫자(numeric) 예외 조건 클래스가 있습니다.

* Invalid operation (잘못된 작업)
* Divide-by-zero (0으로 나누기)
* Denormal operand (비정상적인 피연산자)
* Numeric overflow (숫자 오버플로우)
* Numeric underflow (숫자 언더플로우)
* Inexact result (Precision) (부정확 한 결과 (정밀도))

다음 단계에서는 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/desc.h) 헤더 파일에 정의 된 `used_vectors`배열을 채우고 첫 `32`개 인터럽트에 대해 아래와 같이 `bitmap`을 나타냅니다:

```C
DECLARE_BITMAP(used_vectors, NR_VECTORS);
```

(리눅스 커널의 비트맵에 대한 자세한 내용은 [cpu 마스크 및 비트맵](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)에 대해 설명하는 파트에서 읽을 수 있습니다)

```C
for (i = 0; i < FIRST_EXTERNAL_VECTOR; i++)
	set_bit(i, used_vectors)
```

여기서 `FIRST_EXTERNAL_VECTOR`는 다음과 같습니다.

```C
#define FIRST_EXTERNAL_VECTOR           0x20
```

그런 다음 `ia32_syscall`에 대한 인터럽트 게이트를 설정하고 `used_vectors` 비트맵에 `0x80`을 추가합니다 :

```C
#ifdef CONFIG_IA32_EMULATION
        set_system_intr_gate(IA32_SYSCALL_VECTOR, ia32_syscall);
        set_bit(IA32_SYSCALL_VECTOR, used_vectors);
#endif
```

`x86_64` Linux 커널에는 `CONFIG_IA32_EMULATION` 커널 구성 옵션이 있습니다. 이 옵션은 호환성 모드(compatibility-mode)에서 32 비트 프로세스를 실행하는 기능을 제공합니다. 다음 파트에서 우리는 그것이 어떻게 작동하는지 볼 것입니다. 하지만 다음 파트로 넘어가기 전에, `IDT`에 벡터 번호가 `0x80` 인 또 다른 인터럽트 게이트가 있다는 것을 알아야 합니다. 다음 단계에서 우리는 `IDT`를 fixmap 영역에 매핑하고 :

```C
__set_fixmap(FIX_RO_IDT, __pa_symbol(idt_table), PAGE_KERNEL_RO);
idt_descr.address = fix_to_virt(FIX_RO_IDT);
```

그리고 그 주소를`idt_descr.address`에 씁니다(fix-map된 주소에 대한 자세한 내용은 [Linux 커널 메모리 관리](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-2.html) 챕터의 두 번째 파트에서 볼 수 있습니다). 이 후에 우리는 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d93/arch/x86/kernel/cpu/common.c)에 정의 된 `cpu_init` 함수의 호출을 볼 수 있습니다. 이 함수는 모든 `per-cpu` 상태를 초기화합니다. `cpu_init`의 시작 부분에서 하는 일은 다음과 같습니다 : 가장 먼저 현재 CPU가 초기화되는 동안 대기한 다음 아래 보이는 함수들의 호출이 필요할 경우 (현재 CPU에 대한 `cr4` 제어 레지스터의 쉐도우 복사본을 저장하고 CPU 마이크로코드를 로드하는) `cr4_init_shadow` 함수를 호출합니다.

```C
wait_for_master_cpu(cpu);
cr4_init_shadow();
load_ucode_ap();
```

다음으로 현재 cpu와 `orig_ist` 구조체에 대한 작업 상태 세그먼트'(`Task State Segment`)를 얻습니다. 이는 다음과 함께 원래(origin)의 `Interrupt Stack Table` 값을 나타냅니다.

```C
t = &per_cpu(cpu_tss, cpu);
oist = &per_cpu(orig_ist, cpu);
```

현재 프로세서에 대한 '작업 상태 세그먼트'(`Task State Segment`) 및 '인터럽트 스택 테이블'(`Interrupt Stack Table`)의 값을 얻었으므로 `cr4` 제어 레지스터에서 다음 비트를 지웁니다:

```C
cr4_clear_bits(X86_CR4_VME|X86_CR4_PVI|X86_CR4_TSD|X86_CR4_DE);
```

이를 통해 `vm86` 확장, 가상 인터럽트, 타임 스탬프 ([RDTSC](https://en.wikipedia.org/wiki/Time_Stamp_Counter)는 최고 권한으로만 실행될 수 있음) 및 디버그 확장을 비활성화합니다. 그 후에 우리는 다음과 같이 `Global Descriptor Table`과 `Interrupt Descriptor Table`을 다시로드합니다 :

```C
	switch_to_new_gdt(cpu);
	loadsegment(fs, 0);
	load_current_idt();
```

그런 다음 스레드-로컬 스토리지 디스크립터(Thread-local Storage Descriptors) 배열을 설정하고 [NX](https://en.wikipedia.org/wiki/NX_bit)를 구성하고 CPU 마이크로 코드를 로드합니다. 이제는 `per-cpu` 작업 상태 세그먼트(Task State Segments)를 설정하고 로드 할 차례입니다. 우리는 `N_EXCEPTION_STACKS` 또는 `4` 인 모든 예외 스택을 돌면서 `Interrupt Stack Tables`로 채울 겁니다.

```C
	if (!oist->ist[0]) {
		char *estacks = per_cpu(exception_stacks, cpu);

		for (v = 0; v < N_EXCEPTION_STACKS; v++) {
			estacks += exception_stack_sizes[v];
			oist->ist[v] = t->x86_tss.ist[v] =
					(unsigned long)estacks;
			if (v == DEBUG_STACK-1)
				per_cpu(debug_stack_addr, cpu) = (unsigned long)estacks;
		}
	}
```

'작업 상태 세그먼트'(`Task State Segments`)를 '인터럽트 스택 테이블'(`Interrupt Stack Table`)로 채웠으므로 현재 프로세서에 대해 `TSS` 디스크립터를 설정하고 다음과 같이 로드 할 수 있습니다.

```C
set_tss_desc(cpu, t);
load_TR_desc();
```

여기서 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/desc.h)의 `set_tss_desc` 매크로가 주어진 프로세서의 `Global Descriptor Table`에 주어진 디스크립터를 작성합니다 :

```C
#define set_tss_desc(cpu, addr) __set_tss_desc(cpu, GDT_ENTRY_TSS, addr)
static inline void __set_tss_desc(unsigned cpu, unsigned int entry, void *addr)
{
        struct desc_struct *d = get_cpu_gdt_table(cpu);
        tss_desc tss;
        set_tssldt_descriptor(&tss, (unsigned long)addr, DESC_TSS,
                              IO_BITMAP_OFFSET + IO_BITMAP_BYTES +
                              sizeof(unsigned long) - 1);
        write_gdt_entry(d, entry, &tss, DESC_TSS);
}
```

그리고 `load_TR_desc` 매크로는`ltr` 또는`Load Task Register` 명령어로 확장됩니다 :

```C
#define load_TR_desc()                          native_load_tr_desc()
static inline void native_load_tr_desc(void)
{
        asm volatile("ltr %w0"::"q" (GDT_ENTRY_TSS*8));
}
```

`trap_init` 함수의 끝부분에선 다음과 같은 코드를 볼 수 있습니다 :

```C
set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
...
...
...
#ifdef CONFIG_X86_64
        memcpy(&nmi_idt_table, &idt_table, IDT_ENTRIES * 16);
        set_nmi_gate(X86_TRAP_DB, &debug);
        set_nmi_gate(X86_TRAP_BP, &int3);
#endif
```

여기에서 `idt_table`을 `nmi_dit_table`에 복사하고 `# DB` 또는 `Debug exception`과 `# BR` 또는 `Breakpoint exception`에 대한 예외 처리기를 설정합니다. 이전의 [part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)에서 이러한 인터럽트 게이트를 이미 설정했는데 왜 다시 설정해야 할까요? 이전에 `early_trap_init` 함수에서 초기화했을 때, `Task Task Segment`는 아직 준비되지 않았었지만 이제는 `cpu_init` 함수를 호출 한 후라서 준비가 되었기 때문에 다시 설정하는 것입니다.

이것으로 끝입니다. 곧 이어서는 이러한 인터럽트/예외의 모든 처리기에 대해 생각해 볼 것입니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널에서의 인터럽트와 인터럽트 처리에 관한 네 번째 파트가 끝났습니다. 이번 파트에서 [Task State Segment](https://en.wikipedia.org/wiki/Task_state_segment)의 초기화와 `Divide Error`,`Page Fault` 예외 등과 같은 다른 인터럽트 핸들러들의 초기화를 보았습니다. 우리는 그저 초기화 작업만을 보았으며 이러한 예외에 대한 처리기에 대한 세부 정보는 이제부터 살펴볼 것임을 알아두세요. 이는 다음 파트에서부터 시작하겠습니다.

만약 질문이나 의견이 있으시다면, [트위터](https://twitter.com/0xAX)에서 저를 핑해주시거나, 코멘트를 달아주세요.

**영어는 제 모국어가 아닙니다, 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수들을 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한국 번역](https://github.com/junsooo/linux-insides-ko)로 PR을 보내주세요.**

참고 링크
--------------------------------------------------------------------------------

* [page fault](https://en.wikipedia.org/wiki/Page_fault)
* [Interrupt Descriptor Table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Tracing](https://en.wikipedia.org/wiki/Tracing_%28software%29)
* [cr2](https://en.wikipedia.org/wiki/Control_register)
* [RCU](https://en.wikipedia.org/wiki/Read-copy-update)
* [this_cpu_* operations](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/this_cpu_ops.txt)
* [kmemcheck](https://www.kernel.org/doc/Documentation/kmemcheck.txt)
* [prefetchw](http://www.felixcloutier.com/x86/PREFETCHW.html)
* [3DNow](https://en.wikipedia.org/?title=3DNow!)
* [CPU caches](https://en.wikipedia.org/wiki/CPU_cache)
* [VFS](https://en.wikipedia.org/wiki/Virtual_file_system) 
* [Linux kernel memory management](https://0xax.gitbooks.io/linux-insides/content/MM/index.html)
* [Fix-Mapped Addresses and ioremap](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-2.html)
* [Extended Industry Standard Architecture](https://en.wikipedia.org/wiki/Extended_Industry_Standard_Architecture)
* [INT isntruction](https://en.wikipedia.org/wiki/INT_%28x86_instruction%29)
* [INTO](http://x86.renejeschke.de/html/file_module_x86_id_142.html)
* [BOUND](http://pdos.csail.mit.edu/6.828/2005/readings/i386/BOUND.htm)
* [opcode](https://en.wikipedia.org/?title=Opcode)
* [control register](https://en.wikipedia.org/wiki/Control_register#CR0)
* [x87 FPU](https://en.wikipedia.org/wiki/X86_instruction_listings#x87_floating-point_instructions)
* [MCE exception](https://en.wikipedia.org/wiki/Machine-check_exception)
* [SIMD](https://en.wikipedia.org/?title=SIMD)
* [cpumasks and bitmaps](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [NX](https://en.wikipedia.org/wiki/NX_bit)
* [Task State Segment](https://en.wikipedia.org/wiki/Task_state_segment)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)
