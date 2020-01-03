인터럽트 및 인터럽트 처리. Part 6.
================================================================================

마스크 불가능 인터럽트 처리기
--------------------------------------------------------------------------------

[리눅스 커널에서의 인터럽트 및 인터럽트 처리기](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html) 챕터의 6번째 파트입니다. 지난 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-5.html)에서 우리는 [일반적 보호 결함](https://en.wikipedia.org/wiki/General_protection_fault)예외, 분리 예외, 무효한 [opcode](https://en.wikipedia.org/wiki/Opcode)예외 등등 일부 예외 처리기의 구현을 봤습니다. 이전 파트에서 썻듯이 이파트에서는 나머지 예외의 구현을 볼 것입니다. 다음 처리기의 구현을 봅시다:

* [마스크 불가능](https://en.wikipedia.org/wiki/Non-maskable_interrupt)인터럽트;
* [BOUND](http://pdos.csail.mit.edu/6.828/2005/readings/i386/BOUND.htm)범위 초과 예외;
* [보조프로세서](https://en.wikipedia.org/wiki/Coprocessor)예외;
* [SIMD](https://en.wikipedia.org/wiki/SIMD)보조프로세서 예외.

이 파트에서 시작해봅시다.

마스크 불가능 인터럽트 처리기
--------------------------------------------------------------------------------

[마스크 불가능](https://en.wikipedia.org/wiki/Non-maskable_interrupt)인터럽트는 표준 마스킹 기술에 의해 무시되지 않는 하드웨어 인터럽트입니다. 일반적인 방법으로 마스크 불가능한 인터럽트는 다음 두 가지 방법 중 하나로 생성될 수 있습니다;

* 외부 하드웨어는 CPU에서 마스크 불가능 인터럽트 [핀](https://en.wikipedia.org/wiki/CPU_socket)을 지정
* 프로세서는 시스템 버스 또는 APIC 직렬 버스에서 전달 모드 `NMI`로 메시지를 수신

프로세서가 이러한 소스 중 하나에서 `NMI`를 받으면, 프로세서는 번호가 `2`(첫 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-1.html)의 표 참조)인 인터럽트 벡터에 의해 지정된 `NMI`처리기를 호출하여 즉시 처리합니다. 우리는 이미 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)소스 코드 파일에서 정의된 `trap_init`함수에서 [인터럽트 디스크립터 테이블](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)을 [벡터 번호](https://en.wikipedia.org/wiki/Interrupt_vector_table), `nmi`인터럽트 처리기의 주소 및 `NMI_STACK`[인터럽트 스택 테이블 엔트리](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/kernel-stacks)으로 채웠습니다:

```C
set_intr_gate_ist(X86_TRAP_NMI, &nmi, NMI_STACK);
```

이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)에서 우리는 모든 인터럽트 처리기의 엔트리 포인트가 다음과 같이 정의되는 것을 봤습니다:

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
...
...
...
END(\sym)
.endm
```

[arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S)어셈블리 소스 코드 파일의 매크로 입니다. 그러나 `Non-Maskable`인터럽트 처리기는 이 매크로로 정의되지 않습니다:

```assembly
ENTRY(nmi)
...
...
...
END(nmi)
```

동일한 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S)어셈블리 파일
에 자체 엔트리 포인트가 있습니다. 여기에 들어가서 `Non-Maskable`인터럽트 처리기가 어떻게 작동하는지 이해해봅시다. `nmi`처리기의 호출에서 시작합니다:


```assembly
PARAVIRT_ADJUST_EXCEPTION_FRAME
```

이 매크로는 우리가 다른 챕터에서 보게 될 [반 가상화](https://en.wikipedia.org/wiki/Paravirtualization)와 관련되었기 때문에 이 파트에서는 자세히 다루지 않을 것입니다. 다음으로 스택에 `rdx`레지스터의 내용을 저장합니다:

```assembly
pushq	%rdx
```

그리고 마스크 불가능 인터럽트가 발생했을 때 `cs`가 커널 세그먼트가 아닌 확인이 할당됩니다:

```assembly
cmpl	$__KERNEL_CS, 16(%rsp)
jne	first_nmi
```

`__KERNEL_CS`매크로는 [arch/x86/include/asm/segment.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/segment.h)에 정의됐으며 [글로벌 디스크립터 테이블](https://en.wikipedia.org/wiki/Global_Descriptor_Table)에서 두번째 설명을 보여줍니다:

```C
#define GDT_ENTRY_KERNEL_CS	2
#define __KERNEL_CS	(GDT_ENTRY_KERNEL_CS*8)
```

`GDT`의 자세한 내용은 리눅스 커널 부팅 프로세스 챕터의 두번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)에서 읽을 수 있습니다. `cs`가 커널 세그먼트가 아니라면 이것은 `NMI`가 중첩되지 않음을 의미하고 우리는 `first_nmi`레이블로 넘어갈 수 있습니다. 이 경우를 생각해봅시다. 먼저 현재 스택 포인터의 주소를 `rdx`에 넣고 `1`을 `first_nmi`레이블의 스택에 넣습니다:

```assembly
first_nmi:
	movq	(%rsp), %rdx로
	pushq	$1
```

왜 스택에 `1`을 넣을까요? 이에 대한 답은 `We allow breakpoints in NMIs`입니다. [x86_64](https://en.wikipedia.org/wiki/X86-64)에서, 다른 아키텍쳐와 마찬가지로 CPU의 첫 `NMI`가 완료되기 전에 다른 `NMI`는 실행되지 않습니다. `NMI`인터럽트는 다른 인터럽츠처럼 [iret](http://faydoc.tripod.com/cpu/iret.htm)명령이 완료되고 예외는 이것을 수행합니다. `NMI`처리기가 [페이지 결함](https://en.wikipedia.org/wiki/Page_fault) 또는 [브레이크포인트](https://en.wikipedia.org/wiki/Breakpoint) 또는 `iret`명령을 사용하는 다른 예외를 사용하는 경우. `NMI`컨텍스트에서 이런 일이 일어나면, CPU는 `NMI`컨텍스트를 떠나고 새로운 `NMI`가 발생할 것입니다. `iret`은 `NMIs`를 다시 실행해 이러한 예외에서 벗어나는데 사용되고 우리는 중첩된 마스크 불가능 인터럽트를 얻을 것입니다. 문제는 `NMI`처리기가 예외가 트리거 될때의 상태로 돌아가지 않고, 대신 새로운 `NMIs`가 실행중인 `NMI`처리기를 선점할 수 있는 상태로 돌아갑니다. 첫 NMI처리기가 완료되기 전에 다른 `NMI`가 오면, 새로운 NMI는 선점된 `NMIs`스택 전체에 씁니다. 우리는 이전 `NMI`스택의 맨 위에서 사용중인 다음 `NMI`에서 중첩된 `NMIs`를 얻을 수 있습니다. 이것은 중첩된 마스크 불가능 인터럽트가 이전의 마스크 불가능 인터럽트의 스택을 손상시켜서 실행할 수 없다는 것을 의미합니다. 이것이 임시 변수를 위해 스택에 공간을 할당한 이유입니다. 우리는 이전 `NMI`가 실행될 때 변수가 설정됐는지 확인할 것이고 중첩된 `NMI`가 아니라면 제거할 것입니다. 우리는 `non-maskable` 현재 실행되었음을 나타내기 위해 스택에서 이전에 할당된 공간애 `1`을 넣습니다. `NMI` 또는 다른 예외가 일어났을 때 다음과 같은 [스택 프레임](https://en.wikipedia.org/wiki/Call_stack)이 있다는 것을 기억하십시오:


```
+------------------------+
|         SS             |
|         RSP            |
|        RFLAGS          |
|         CS             |
|         RIP            |
+------------------------+
```

또한 예외가 있는 경우 에러 코드가 있습니다. 따라서 이러한 모든 조작 후에 스택 프레임은 다음과 같습니다:

```
+------------------------+
|         SS             |
|         RSP            |
|        RFLAGS          |
|         CS             |
|         RIP            |
|         RDX            |
|          1             |
+------------------------+
```

다음 단계에서 우리는 스택의 또 다른 `40`바이트를 할당합니다:

```assembly
subq	$(5*8), %rsp
```

그리고 [.rept](http://tigcc.ticalc.org/doc/gnuasm.html#SEC116)어셈블리 지시문의 공간이 할당된 다음 오리지널 스택 프레임의 복사본을 넣습니다:

```C
.rept 5
pushq	11*8(%rsp)
.endr
```

우리는 오리지널 스택 프레임의 복사본이 필요합니다. 일반적으로 인터럽트 스택의 두 복사본이 필요합니다. 처음은 `copied`인터럽트 스택: `saved`스택 프레임과 `copied`스택 프레임입니다. 이제 우리는 오리지널 스택 프레임을 할당된 `40`바이트(`copied`스택 프레임) 다음에 위치한 `saved`스택 프레임에 넣습니다. 이 스택 프레임은 중첩된 NMI가 바꿀수 있는 `copied`스택 프레임을 수정하는데 사용됩니다. 두 번째 `copied`스택 프레임은 중첩된 `NMIs`가 트리거된 두번째 `NMI`를 첫 번째 `NMI`가 아는 것을 허용하고 우리가 첫 번째 `NMI`처리기를 반복해야 함을 알립니다. 우리는 오리지널 스택 프레임의 첫 복사본을 만들었습니다. 이제 두 번째 복사본을 만들 차례입니다:

```assembly
addq	$(10*8), %rsp

.rept 5
pushq	-6*8(%rsp)
.endr
subq	$(5*8), %rsp
```

이 모든 조작 후의 스택 프레임은 다음과 같습니다:

```
+-------------------------+
| original SS             |
| original Return RSP     |
| original RFLAGS         |
| original CS             |
| original RIP            |
+-------------------------+
| temp storage for rdx    |
+-------------------------+
| NMI executing variable  |
+-------------------------+
| copied SS               |
| copied Return RSP       |
| copied RFLAGS           |
| copied CS               |
| copied RIP              |
+-------------------------+
| Saved SS                |
| Saved Return RSP        |
| Saved RFLAGS            |
| Saved CS                |
| Saved RIP               |
+-------------------------+
```

그런 다음 이전 예외 처리기에서 이미 했던 것처럼 더미 오류 코드를 스택에 넣고 스택의 범용 레지스터를 위한 공간을 할당합니다: 

```assembly
pushq	$-1
ALLOC_PT_GPREGS_ON_STACK
```

우리는 이미 인터럽트 [챕터](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)의 세 번째 파트에서 `ALLOC_PT_GREGS_ON_STACK`매크로의 구현을 봤습니다. 이 매크로는 [arch/x86/entry/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h)에서 정의되었으며 `rdi`에서 또 다른 범용 레지스터를 위한 스택에 `120`바이트를 `r15`로 할당합니다:

```assembly
.macro ALLOC_PT_GPREGS_ON_STACK addskip=0
addq	$-(15*8+\addskip), %rsp
.endm
```

일반 레지스터를 위한 공간 할당 후 `paranoid_entry`의 호출을 볼 수 있습니다:

```assembly
call	paranoid_entry
```

이전 파트에서 이 레이블을 떠올릴 수 있습니다. 스텍에서 범용 레지스터를 넣고, `MSR_GS_BASE`[모델 특정 레지스터](https://en.wikipedia.org/wiki/Model-specific_register)를 읽고 그것의 값을 확인합니다. `MSR_GS_BASE`의 값이 음수면, 커널 모드로 가고 `paranoid_entry`를 반환합니다. 다른 방법은 우리가 사용자 모드로 갔고 커널 `gs`에서 사용자 `gs`를 바꾸기 위해 `swapgs`명령을 실행해야 함을 의미합니다:

```assembly
ENTRY(paranoid_entry)
	cld
	SAVE_C_REGS 8
	SAVE_EXTRA_REGS 8
	movl	$1, %ebx
	movl	$MSR_GS_BASE, %ecx
	rdmsr
	testl	%edx, %edx
	js	1f
	SWAPGS
	xorl	%ebx, %ebx
1:	ret
END(paranoid_entry)
```

`swapgs`명령 후 `ebx`레지스터를 0으로 설정했습니다. 다음에 우리는 이 레지스터의 내용을 확인하고 다른 방식으로 `0`과 `1`을 포함한 `ebx` 대신 실행됬는지 확인합니다. 다음 단계에서 `NMI`처리기가 `page fault`을 일으키고 컨드롤 레지스터의 값을 바꾸기 때문에 `r12`레지스터의 `cr2`[컨트롤 레지스터](https://en.wikipedia.org/wiki/Control_register)의 값을 저장합니다:

```C
movq	%cr2, %r12
```

이제 실제 `NMI`처리기를 호출할 차례입니다. `rdi`에서 `pt_regs`의 주소, `rsi`의 에러코드를 넣고 `do_nmi`처리기를 호출합니다:

```assembly
movq	%rsp, %rdi
movq	$-1, %rsi
call	do_nmi
```

우리는 이 파트에서 `do_nmi`의 후반부로 돌아갈 것이지만, 지금은 `do_nmi`의 실행이 끝난 후에 무슨 일이 일어나는지 보겠습니다. `do_nmi`처리기가 끝난 다음 `cr2`레지스터를 검사하는데 `do_nmi`이 수행되는 동안 페이지 결함이 생길 수 있고, 만약 그것을 얻으면 오리지널 `cr2`를 복원하며 다른 방법으로는 레이블 `1`로 점프합니다. 그런 다음 `ebx` 레지스터(`swapgs`명령을 사용한 경우 0을, 그렇지 않은 경우 1을 가짐)의 내용을 테스트 하고 `1`을 가지거나 `nmi_restore`레이블로 점프한 경우 `SWAPGS_UNSAFE_STACK`을 실행합니다. `SWAPGS_UNSAFE_STACK`매크로는 `swapgs`명령을 확장합니다. `nmi_restore`레이블에서 우리는 범용 레지스터를 복구하고, 이 레지스터를 위한 스택에 할당된 공간을 제거하며, 임시 변수를 지우고 `INTERRUPT_RETURN`매크로를 사용해 인터럽트 처리기를 종료합니다:

```assembly
	movq	%cr2, %rcx
	cmpq	%rcx, %r12
	je	1f
	movq	%r12, %cr2
1:
	testl	%ebx, %ebx
	jnz	nmi_restore
nmi_swapgs:
	SWAPGS_UNSAFE_STACK
nmi_restore:
	RESTORE_EXTRA_REGS
	RESTORE_C_REGS
	/* Pop the extra iret frame at once */
	REMOVE_PT_GPREGS_FROM_STACK 6*8
	/* Clear the NMI executing stack variable */
	movq	$0, 5*8(%rsp)
	INTERRUPT_RETURN
```

`INTERRUPT_RETURN`은 [arch/x86/include/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/irqflags.h)에 정의되었으며 `iret`명령을 확장합니다. 이것이 전부입니다.

이제 이전 `NMI`인터럽트가 실행을 끝내지 않았을 때 다른 `NMI`인터럽트가 일어난 경우를 고려해봅시다. 이 파트의 도입부부터 우리는 사용자 공간에서 온 것을 확인하고 이 경우 `first_nmi`으로 점프한 것을 떠올릴 수 있습니다:

```assembly
cmpl	$__KERNEL_CS, 16(%rsp)
jne	first_nmi
```

이 경우 이것은 매번 첫 `NMI`입니다. 왜냐하면 첫 `NMI`가 페이지 결함, 브레이크 포인트 또는 다른 예외를 잡은 것으로 인해 커널 모드가 실행되기 때문입니다. 사용자 공간에서 오지 않은 경우, 우선 임시 변수를 테스트하십시오:

```assembly
cmpl	$1, -8(%rsp)
je	nested_nmi
```

그것이 `1`로 설정됐으면 우리는 `nested_nmi`레이블로 점프합니다. `1`이 아니면 `IST`스택을 테스트합니다. 중첩된 `NMIs`의 경우 `repeat_nmi` 위에 있는지 확인합니다. 무시하는 경우, 다른 방법으로 `end_repeat_nmi`보다 위인지 확인하고 `nested_nmi_out`레이블로 점프합니다.

이제 `do_nmi`예외 처리기를 살펴보겠습니다. 이 함수는 [arch/x86/kernel/nmi.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/nmi.c)소스 코드 파일에 정의되었으며 모든 예외처리기는 두 매개변수를 가집니다:

* `pt_regs`의 주소;
* 에러 코드.

`do_nmi`는 `nmi_nesting_preprocess`함수의 호출로 시작하며 `nmi_nesting_postprocess`의 호출과 함께 끝납니다. `nmi_nesting_preprocess`함수는 디버그 스택에서 작동하지 않는지 확인하고 디버그 스택에서 `update_debug_stack` [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)변수가 `1`로 설정됐으면 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c)의 `debug_stack_set_zero`함수를 호출합니다 이 함수는 CPU당 `debug_stack_use_ctr`변수를 증가시키고 새로운 `Interrupt Descriptor Table`을 로드합니다:
 

```C
static inline void nmi_nesting_preprocess(struct pt_regs *regs)
{
        if (unlikely(is_debug_stack(regs->sp))) {
                debug_stack_set_zero();
                this_cpu_write(update_debug_stack, 1);
        }
}
```

`nmi_nesting_postprocess`함수는 `nmi_nesting_preprocess`에서 설정한 CPU당 `update_debug_stack`변수를 확인하고 디버그 스택을 리셋하거나 다른 말로 본래의 `Interrupt Descriptor Table`을 로드합니다. `nmi_nesting_preprocess`함수의 호출 이후, `do_nmi`에서 `nmi_enter`의 호출을 볼 수 있습니다. `nmi_enter`는 인터럽티드 프로세스의 `lockdep_recursion`필드를 증가시키고, 점유한 카운터를 업데이트 하며 `NMI`에 관한 [RCU](https://en.wikipedia.org/wiki/Read-copy-update)서브시스템을 알립니다. 또한 `nmi_exit`함수와 비슷한 `nmi_enter`도 있지만 그 반대도 마찬가지입니다. `nmi_enter` 이후 `irq_stat`구조체의 `__nmi_count`를 증가시키고 `default_do_nmi`함수를 호출합니다. 먼저 모든 `default_do_nmi`에서 이전 nmi의 주소를 확인하고 마지막 nmi의 주소를 실제 주소로 업데이트 합니다:

```C
if (regs->ip == __this_cpu_read(last_nmi_rip))
    b2b = true;
else
    __this_cpu_write(swallow_nmi, false);

__this_cpu_write(last_nmi_rip, regs->ip);
```

그 다음 CPU특정 `NMIs`를 처리해야 합니다:

```C
handled = nmi_handle(NMI_LOCAL, regs, b2b);
__this_cpu_add(nmi_stats.normal, handled);
```

그리고 비특정 `NMIs`는 reason에 따릅니다:

```C
reason = x86_platform.get_nmi_reason();
if (reason & NMI_REASON_MASK) {
	if (reason & NMI_REASON_SERR)
		pci_serr_error(reason, regs);
	else if (reason & NMI_REASON_IOCHK)
		io_check_error(reason, regs);

	__this_cpu_add(nmi_stats.external, 1);
	return;
}
```

이것이 전부입니다.

범위 초과 예외
--------------------------------------------------------------------------------

다음 예외는 `BOUND`범위를 초과한 예외입니다. 이 `BOUND`명령은 첫 번째 피연산자(배열 인덱스)가 두 번째 피연산자(바운드 피연산자)에 지정된 배열의 범위 내에 있는지 확인합니다. 인덱스가 bound 내에 없으면 `BOUND`범위는 예외가 초과됐거나 `#BR`이 발생한 것입니다. `#BR`예외의 처리기는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)에 정의된 `do_bounds`함수입니다. `do_bounds`처리기는 `exception_enter`함수의 호출로 시작하며 `exception_exit`의 호출로 끝납니다:

```C
prev_state = exception_enter();

if (notify_die(DIE_TRAP, "bounds", regs, error_code,
	           X86_TRAP_BR, SIGSEGV) == NOTIFY_STOP)
    goto exit;
...
...
...
exception_exit(prev_state);
return;
```

이전 컨텍스트의 상태를 얻은 후에, 예외를 `notify_die`체인에 추가하고 `NOTIFY_STOP`을 반환하면 예외에서 돌아옵니다. notify체인 및 `context tracking`함수에 대한 것은 [이전 파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-5.html)에서 읽을 수 있습니다. 다음 단계에서 `IF`플래그를 확인하는 `contidional_sti`함수가 비활성화됐으면 인터럽트를 활성화하고 그 값에 따르는 `local_irq_enable`을 호출합니다:

```C
conditional_sti(regs);

if (!user_mode(regs))
	die("bounds", regs, error_code);
```

그리고 사용자 모드에서 오지 않았으면 `die`함수와 함께 `SIGSEGV`신호를 보내는지 확인하십시오. 다음으로 우리는 [MPX](https://en.wikipedia.org/wiki/Intel_MPX)의 활성화 여부를 확인하고 이 기능이 비활성화 됐으면 `exit_trap`레이블로 점프합니다:

```C
if (!cpu_feature_enabled(X86_FEATURE_MPX)) {
	goto exit_trap;
}

여기서 우리는 `do_trap`함수를 실행합니다(자세한 내용은 이전 파트에서 찾을 수 있습니다):

```C
exit_trap:
	do_trap(X86_TRAP_BR, SIGSEGV, "bounds", regs, error_code, NULL);
	exception_exit(prev_state);
```

`MPX`기능이 활성화된 경우 `get_xsave_field_ptr`함수로 `BNDSTATUS`을 확인하고 이것이 0이면, 그것은`MPX`가 이 예외의 원인이 아닌 것을 의미합니다:

```C
bndcsr = get_xsave_field_ptr(XSTATE_BNDCSR);
if (!bndcsr)
		goto exit_trap;
```

이 모든 것 이후에도 `MPX`가 이 예외의 원인일 때 한 가지 방법이 있습니다. 이 파트에서는 인텔 메모리 보호 확장에 대한 자세한 내용은 다루지 않지만 다른 챕터에서 살펴볼 것입니다.

보조프로세서 예외 및 SIMD예외
--------------------------------------------------------------------------------

다음 두 가지 예외는 [x87 FPU](https://en.wikipedia.org/wiki/X87) 부동 소수점 오류 또는 `#MF`와 [SIMD](https://en.wikipedia.org/wiki/SIMD)부동 소수점 예외 또는 `#XF`입니다. 첫 번째 예외는 `x87 FPU`이 부동 소수점 오류를 감지했을 때 일어납니다. 예를 들어 0으로 나누기, 숫자 오버플로 등이 있습니다. 두 번째 예외는 프로세서가 [SSE/SSE2/SSE3](https://en.wikipedia.org/wiki/SSE3) `SIMD` 부동 소수점 예외를 감지했을 때 일어납니다. 이것은 `x87 FPU`와 비슷합니다. 이 예외를 위한 처리기는 `do_coprocessor_error`와 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)에서 정의된 `do_simd_coprocessor_error`가 있으며 서로 매우 유사합니다. 둘 다 동일한 소스 코드 파일에서 `math_error`함수를 호출하지만 다른 벡터 번호를 전달합니다. `do_coprocessor_error`는 `math_error`로 `X86_TRAP_MF`벡터 번호를 전달합니다:

```C
dotraplinkage void do_coprocessor_error(struct pt_regs *regs, long error_code)
{
	enum ctx_state prev_state;

	prev_state = exception_enter();
	math_error(regs, error_code, X86_TRAP_MF);
	exception_exit(prev_state);
}
```

그리고 `do_simd_coprocessor_error`는 `math_error`함수로 `X86_TRAP_XF`를 전달합니다:

```C
dotraplinkage void
do_simd_coprocessor_error(struct pt_regs *regs, long error_code)
{
	enum ctx_state prev_state;

	prev_state = exception_enter();
	math_error(regs, error_code, X86_TRAP_XF);
	exception_exit(prev_state);
}
```

먼저 모든 `math_error`함수는 현재 중단된 작업, fpu의 주소, 예외를 설명하는 문자열을 정의하고, 그것을 `notify_die`체인에 더하며 `NOTIFY_STOP`이 반환되면 예외 처리기에서 반환합니다:

```C
	struct task_struct *task = current;
	struct fpu *fpu = &task->thread.fpu;
	siginfo_t info;
	char *str = (trapnr == X86_TRAP_MF) ? "fpu exception" :
						"simd exception";

	if (notify_die(DIE_TRAP, str, regs, error_code, trapnr, SIGFPE) == NOTIFY_STOP)
		return;
```

그 후에 커널 모드에서 왔는지 확인하고 그렇다면 `fixup_exception`함수로 예외를 고치려 노력할 것입니다. 예외의 오류 코드와 벡터 번호로 작업을 채울 수 없거나 죽은 경우:

```C
if (!user_mode(regs)) {
	if (!fixup_exception(regs)) {
		task->thread.error_code = error_code;
		task->thread.trap_nr = trapnr;
		die(str, regs, error_code);
	}
	return;
}
```

사용자 모드에서 온 경우 `fpu`상태를 저장하고 예외의 벡터 번호로 작업 구조체를 채우고 신호의 숫자, `errno`, 예외가 발생한 곳의 주소, 신호 코드로 `siginfo_t`을 채웁니다:

```C
fpu__save(fpu);

task->thread.trap_nr	= trapnr;
task->thread.error_code = error_code;
info.si_signo		= SIGFPE;
info.si_errno		= 0;
info.si_addr		= (void __user *)uprobe_get_trap_addr(regs);
info.si_code = fpu__exception_code(fpu, trapnr);
```

그런 다음 신호 코드를 확인하고 0이 아닌 경우 반환합니다:

```C
if (!info.si_code)
	return;
```
		
또는 마지막에 `SIGFPE`신호를 보냅니다:

```C
force_sig_info(SIGFPE, &info, task);
```

이것이 전부입니다.

결론
--------------------------------------------------------------------------------

[인터럽트 및 인터럽트 처리](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)챕터의 6번째 파트의 끝으로 우리는 이 파트에서 `non-maskable`인터럽트, [SIMD](https://en.wikipedia.org/wiki/SIMD) 및 [x87 FPU](https://en.wikipedia.org/wiki/X87)부동 소수점 예외와 같은 몇몇 예외 처리기의 구현을 봤습니다. 결과적으로 우리는 이 파트에서 `trap_init`함수를 끝냈고 다음 파트로 향할 것입니다. 다음은 외부 인터럽트와 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)의 `early_irq_init`함수입니다.

질문이나 제안사항이 있다면 코멘트를 남기거나 [트위터](https://twitter.com/0xAX)로 보내주십시오.

**영어는 모국어가 아니어서 모든 불편함 점은 정말 죄송합니다. 실수를 발견하면 [linux-insides](https://github.com/0xAX/linux-insides)에서 수정 사항이 포함된 PR을 보내주십시오.**

링크
--------------------------------------------------------------------------------

* [일반 보호 결함](https://en.wikipedia.org/wiki/General_protection_fault)
* [opcode](https://en.wikipedia.org/wiki/Opcode)
* [마스크 불가능](https://en.wikipedia.org/wiki/Non-maskable_interrupt) 
* [BOUND 명령](http://pdos.csail.mit.edu/6.828/2005/readings/i386/BOUND.htm)
* [CPU 소켓](https://en.wikipedia.org/wiki/CPU_socket)
* [인터럽트 디스크립터 테이블](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [인터럽트 스택 테이블](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/kernel-stacks)
* [반가상화](https://en.wikipedia.org/wiki/Paravirtualization)
* [.rept](http://tigcc.ticalc.org/doc/gnuasm.html#SEC116)
* [SIMD](https://en.wikipedia.org/wiki/SIMD)
* [보조프로세서](https://en.wikipedia.org/wiki/Coprocessor)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [iret](http://faydoc.tripod.com/cpu/iret.htm)
* [페이지 결함](https://en.wikipedia.org/wiki/Page_fault)
* [breakpoint](https://en.wikipedia.org/wiki/Breakpoint)
* [글로벌 디스크립터 테이블](https://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [스택 프레임](https://en.wikipedia.org/wiki/Call_stack)
* [모델 특정 레지스터](https://en.wikipedia.org/wiki/Model-specific_register)
* [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [RCU](https://en.wikipedia.org/wiki/Read-copy-update) 
* [MPX](https://en.wikipedia.org/wiki/Intel_MPX)
* [x87 FPU](https://en.wikipedia.org/wiki/X87)
* [이전 파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-5.html)
