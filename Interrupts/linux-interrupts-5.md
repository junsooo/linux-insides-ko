인터럽트 및 인터럽트 처리. Part 5.
================================================================================

예외 처리기의 구현
--------------------------------------------------------------------------------

이것은 리눅스 커널의 인터럽트와 예외 처리에 관한 다섯 번째 파트로, 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-4.html)에서는 [인터럽트 디스크립터 테이블](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)에 대한 인터럽트 게이트를 설정하고 끝났습니다. 우리는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)소스코드 파일에서 `trap_init`함수를 실행했었습니다. 이전 파트에서는 이러한 인터럽트 게이트 설정만 봤고 현재 파트에서는 이러한 게이트에 대한 예외 처리기의 구현을 볼 수 있습니다. 예외 처리기가 실행되기 전 준비는 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S)어셈블리 파일에 있으며 예외 진입 포인트를 정의하는 [idtentry](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S#L820)매크로에 의해 일어납니다:

```assembly
idtentry divide_error			        do_divide_error			       has_error_code=0
idtentry overflow			            do_overflow			           has_error_code=0
idtentry invalid_op			            do_invalid_op			       has_error_code=0
idtentry bounds				            do_bounds			           has_error_code=0
idtentry device_not_available		    do_device_not_available		   has_error_code=0
idtentry coprocessor_segment_overrun	do_coprocessor_segment_overrun has_error_code=0
idtentry invalid_TSS			        do_invalid_TSS			       has_error_code=1
idtentry segment_not_present		    do_segment_not_present		   has_error_code=1
idtentry spurious_interrupt_bug		    do_spurious_interrupt_bug	   has_error_code=0
idtentry coprocessor_error		        do_coprocessor_error		   has_error_code=0
idtentry alignment_check		        do_alignment_check		       has_error_code=1
idtentry simd_coprocessor_error		    do_simd_coprocessor_error	   has_error_code=0
```

`idtentry`매크로는 실제 예외 처리기 이전의 준비(`divide_error`를 위한 `do_divide_error`, `overflow`를 위한 `do_overflow` 등등)에 대한 제어를 얻습니다. 다시 말해 `idtentry`매크로는 스택의 레지스터([pt_regs](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/ptrace.h#L43)구조체)를 위한 위치를 할당하고, 인터럽트/예외에 오류 코드가 없는 경우 stack consistency에 대한 더미 오류 코드를 넣고, `cs`세그먼트 레지스터에서 세그먼트 셀렉터를 확인하고 이전 상태(사용자 공간 또는 커널 공간)에 따라 전환합니다. 이러한 모든 준비가 끝나면 실제 인터럽트/예외 처리기를 호출합니다:

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
	...
	...
	...
	call	\do_sym
	...
	...
	...
END(\sym)
.endm
```

예외 처리기가 작업을 완료한 다음 `idtentry`매크로는 중단된 작업의 스택 및 범용 레지스터를 복구하고 [iret](http://x86.renejeschke.de/html/file_module_x86_id_145.html)명령을 실행합니다:

```assembly
ENTRY(paranoid_exit)
	...
	...
	...
	RESTORE_EXTRA_REGS
	RESTORE_C_REGS
	REMOVE_PT_GPREGS_FROM_STACK 8
	INTERRUPT_RETURN
END(paranoid_exit)
```

`INTERRUPT_RETURN`의 위치:

```assembly
#define INTERRUPT_RETURN	jmp native_iret
...
ENTRY(native_iret)
.global native_irq_return_iret
native_irq_return_iret:
iretq
```

`idtentry`에 대한 자세한 내용은 [https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-3.html)의 세번째 부분에서 읽을 수 있습니다. 예외 처리기가 실행되기 전에 준비해야 할 것을 보았으며, 이제 처리기를 살펴보겠습니다. 우선 다음 처리기를 살펴보겠습니다:

* 분할 오류(divide error)
* 오버플로우(overflow)
* 무효한 op(invalid op)
* 보조프로세서 세그먼트 오버런(coprocessor segment overrun)
* 무효한 TSS(invalid TSS)
* 세그먼트 없음(segment not present)
* 스택 세그먼트(stack segment)
* 정렬 확인(alignment check)

이러한 모든 처리기는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)소스 코드 파일에 `DO_ERROR`매크로와 함께 정의되어 있습니다:

```C
DO_ERROR(X86_TRAP_DE,     SIGFPE,  "divide error",                divide_error)
DO_ERROR(X86_TRAP_OF,     SIGSEGV, "overflow",                    overflow)
DO_ERROR(X86_TRAP_UD,     SIGILL,  "invalid opcode",              invalid_op)
DO_ERROR(X86_TRAP_OLD_MF, SIGFPE,  "coprocessor segment overrun", coprocessor_segment_overrun)
DO_ERROR(X86_TRAP_TS,     SIGSEGV, "invalid TSS",                 invalid_TSS)
DO_ERROR(X86_TRAP_NP,     SIGBUS,  "segment not present",         segment_not_present)
DO_ERROR(X86_TRAP_SS,     SIGBUS,  "stack segment",               stack_segment)
DO_ERROR(X86_TRAP_AC,     SIGBUS,  "alignment check",             alignment_check)
``` 

보다시피 `DO_ERROR`매크로는 4개의 매개변수를 사용합니다:

* 인터럽트의 벡터 번호;
* 중단 된 프로세스로 전송 될 신호의 번호;
* 예외를 설명하는 문자열;
* 예외 처리기 진입 포인트.

이 매크로는 동일한 소스 코드 파일에 정의되어 있으며 `do_handler`라고 명명된 함수로 확장됩니다:

```C
#define DO_ERROR(trapnr, signr, str, name)                              \
dotraplinkage void do_##name(struct pt_regs *regs, long error_code)     \
{                                                                       \
        do_error_trap(regs, error_code, str, trapnr, signr);            \
}
```

`##`토큰을 주의하십시오. 이것은 특별한 함수입니다. [GCC 매크로 연결](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html#Concatenation)은 주어진 두 개의 문자열을 연결합니다. 예를 들어, 첫째로 우리의 예시에서 `DO_ERROR`는 다음과 같이 확장됩니다:

```C
dotraplinkage void do_divide_error(struct pt_regs *regs, long error_code)     \
{
	...
}
```

우리는 `DO_ERROR`매크로에 의해 생성된 모든 함수가 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에서 `do_error_trap`함수의 호출을 일으키는 것을 볼 수 있습니다. `do_error_trap`함수의 구현을 살펴봅시다:

트랩 처리기
--------------------------------------------------------------------------------

`do_error_trap`함수는 [include/linux/context_tracking.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking.h)에 있는 다음의 두 가지 g함수로 시작하고 끝납니다:
```C
enum ctx_state prev_state = exception_enter();
...
...
...
exception_exit(prev_state);
```

리눅스 커널 서브 시스템의 컨텍스트 추적은 두 가지 기본 초기 컨텍스트 `user` 또는 `kernel`을 통해 레벨 컨테스트 사이에서 전환을 추적하기 위해 커널 경계 프로브를 제공합니다. `exception_enter`함수는 컨텍스트 추적이 활성화됐는지 확인합니다. 활성화가 됐으면 `exception_enter`는 이전의 컨텍스트를 읽고 `CONTEXT_KERNEL`과 비교합니다. 이전 컨텍스트가 `user`인 경우 [kernel/context_tracking.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/context_tracking.c)에서 `context_tracking_exit`함수를 호출해 컨텍스트 추적 서브시스템에 프로세서가 사용자 모드를 종료하고 커널 모드로 들어가고 있음을 알립니다:

```C
if (!context_tracking_is_enabled())
	return 0;

prev_ctx = this_cpu_read(context_tracking.state);
if (prev_ctx != CONTEXT_KERNEL)
	context_tracking_exit(prev_ctx);

return prev_ctx;
```

이전의 컨텍스트가 `user`가 아니라면 그것을 반환합니다. `pre_ctx`은 [include/linux/context_tracking_state.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking_state.h)에서 정의된 `enum ctx_state`타입을 가집니다. 다음을 보십시오:

```C
enum ctx_state {
	CONTEXT_KERNEL = 0,
	CONTEXT_USER,
	CONTEXT_GUEST,
} state;
```

두 번째 함수는 동일한 [include/linux/context_tracking.h](https://github.com/torvalds/linux/tree/master/include/linux/context_tracking.h)파일에서 정의된 `exception_exit`으로 컨텍스트 추적이 사용 가능한지 확인하고 이전 컨텍스트가 `user`면 `contert_tracking_enter`함수를 호출합니다:

```C
static inline void exception_exit(enum ctx_state prev_ctx)
{
	if (context_tracking_is_enabled()) {
		if (prev_ctx != CONTEXT_KERNEL)
			context_tracking_enter(prev_ctx);
	}
}
```

`context_tracking_enter`함수는 컨텍스트 추적 서브 시스템에 프로세서가 커널 모드에서 사용자 모드로 진입한다는 것을 알려줍니다. `exception_enter`과 `exception_exit` 사이에서 다음 코드를 볼 수 있습니다:

```C
if (notify_die(DIE_TRAP, str, regs, error_code, trapnr, signr) !=
		NOTIFY_STOP) {
	conditional_sti(regs);
	do_trap(trapnr, signr, str, regs, error_code,
		fill_trap_info(regs, signr, trapnr, &info));
}
```

먼저 [kernel/notifier.c](https://github.com/torvalds/linux/tree/master/kernel/notifier.c)에서 정의된 `notify_die`함수를 호출합니다. 호출자가 자신을 `notify_die`체인에 삽입해야하는 [커널 패닉](https://en.wikipedia.org/wiki/Kernel_panic), [커널 oops](https://en.wikipedia.org/wiki/Linux_kernel_oops), [마스크 불가능 인터럽트](https://en.wikipedia.org/wiki/Non-maskable_interrupt) 또는 다른 이벤트에 대해 알림을 받으려면  `notify_die`함수가 이를 수행해야 합니다. 리눅스 커널에는 무언가 일어났을 때 커널에 묻는 것을 허락하는 특별한 메커니즘이 있으며 이는 `notifiers` 또는 `notifier chains`라고 불립니다. 이 매커니즘을 사용하는 예시는 `USB`핫플러그인 이벤트([drivers/usb/core/notify.c](https://github.com/torvalds/linux/tree/master/drivers/usb/core/notify.c)을 보십시오), 메모리 [핫플러그](https://en.wikipedia.org/wiki/Hot_swapping)([include/linux/memory.h](https://github.com/torvalds/linux/tree/master/include/linux/memory.h)을 보십시오, `hotplug_memory_notifier`매크로 등등), 시스템 리부트 등이 있습니다. notifier체인은 단순하고 단일 링크된 리스트입니다. 리눅스 커널 하위시스템에 특정 이벤트를 알리려는 때에 이 체인은 특별한 `notifier_block`구조체를 채우고 이 구조체를 `notifier_chain_register`함수에 전달합니다. `notifier_call_chain`함수의 호출과 함께 이벤트를 보낼 수 있습니다. 먼저 모든 `notify_die`함수는 `die_args`구조체를  트랩 넘버, 트랩 문자열, 레지스터와 다른 값들로 채웁니다:

```C
struct die_args args = {
       .regs   = regs,
       .str    = str,
       .err    = err,
       .trapnr = trap,
       .signr  = sig,
}
```

`die_chain`과 함께 `atomic_notifier_call_chain`함수의 결과를 다음과 같이 반환합니다:
```C
static ATOMIC_NOTIFIER_HEAD(die_chain);
return atomic_notifier_call_chain(&die_chain, val, &args);
```

잠금과 `notifier_block`을 포함한 `atomic_notifier_head`구조체가 확장됩니다:

```C
struct atomic_notifier_head {
        spinlock_t lock;
        struct notifier_block __rcu *head;
};
```

`atomic_notifier_call_chain`함수는 notifier체인에서 각 함수를 차례대로 호출하고 마지막으로 호출된 notifier 함수의 값을 반환합니다. `do_error_trap`의 `notify_die`가 `NOTIFY_STOP`을 반환하지 않은 경우 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)의 `conditional_sti`함수를 실행해 [인터럽트 플래그](https://en.wikipedia.org/wiki/Interrupt_flag)의 값을 확인하고 이것에 의존하는 인터럽트를 활성화합니다:

```C
static inline void conditional_sti(struct pt_regs *regs)
{
        if (regs->flags & X86_EFLAGS_IF)
                local_irq_enable();
}
```

`local_irq_enable`매크로의 자세한 정보는 이 챕터의 두 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-2.html)에서 읽을 수 있습니다. 다음이자 마지막 호출 `do_error_trap`은 `do_trap`함수입니다. 먼저 모든 `do_trap`함수는 `task_struct`타입을 가진 `tsk`변수로 정의되며 현재 중단된 프로세스를 나타냅니다. 다음으로 `tsk`의 정의는 `do_trap_no_signal`함수의 호출을 통해 볼 수 있습니다:

```C
struct task_struct *tsk = current;

if (!do_trap_no_signal(tsk, trapnr, str, regs, error_code))
	return;
```

`do_trap_no_signal`함수는 두 가지 검사를 수행합니다:

* [가상 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode)모드에서 나왔는가?
* 커널 공간에서 나왔는가?

```C
if (v8086_mode(regs)) {
	...
}

if (!user_mode(regs)) {
	...
}

return -1;
```

[long 모드](https://en.wikipedia.org/wiki/Long_mode)는 [가상 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode)모드를 지원하지 않기 때문에 첫번째 경우는 고려하지 않아도 됩니다. 두번째 경우로 결함을 복구하려고 하는 `fixup_exception`함수와 수행이 불가능한 경우의 `die`가 있습니다:

```C
if (!fixup_exception(regs)) {
	tsk->thread.error_code = error_code;
	tsk->thread.trap_nr = trapnr;
	die(str, regs, error_code);
}
```

`die`함수는 [arch/x86/kernel/dumpstack.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/dumpstack.c)소스 코드 파일에 정의됐으며, 스택, 레지스터, 커널 모듈에 대해 유용한 정보를 출력하며 커널 [oops](https://en.wikipedia.org/wiki/Linux_kernel_oops)의 원인이 됩니다. `do_trap_no_signal`함수가 사용자 공간에서 온 경우 `-1`을 반환할 것이고 `do_trap`함수의 실행이 계속될 것입니다. `do_trap_no_signal`함수를 통과 했지만 `do_trap`이후에 종료되지 않았다면, 이는 이전의 컨텍스트가 `user`임을 의미합니다. 프로세서로 인한 대부분의 예외는 리눅스에서 오류 조건(0으로 나누기, 무효한 opcode 등등)으로 해석됩니다. 예외가 발생하면 리눅스 커널은 예외로 인해 잘못된 상태를 알리는 중단된 프로세스에 [신호](https://en.wikipedia.org/wiki/Unix_signal)를 보냅니다. 따라서 `do_trap`함수에서 주어진 숫자(분리 오류를 위한 `SIGFPE`, 오버플로우 예외를 위한 `SIGILL` 등등)의 신호를 보내야합니다. 우선 `thread.error_code`와 `thread_trap_nr`로 채운 현재 인터럽트 프로세스의 에러 코드와 벡터 번호를 저장합니다:

```C
tsk->thread.error_code = error_code;
tsk->thread.trap_nr = trapnr;
```

이후에 중단된 프로세스를 위해 처리되지 않은 신호에 대한 정보를 출력하기 위한 검사를 합니다. `show_unhandled_signals`변수가 설정됐는지, [kernel/signal.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/signal.c)의 `unhandled_signal`함수가 처리되지 않은 신호 및 [printk](https://en.wikipedia.org/wiki/Printk)레이트 제한을 반환하는지 확인합니다: 

```C
#ifdef CONFIG_X86_64
	if (show_unhandled_signals && unhandled_signal(tsk, signr) &&
	    printk_ratelimit()) {
		pr_info("%s[%d] trap %s ip:%lx sp:%lx error:%lx",
			tsk->comm, tsk->pid, str,
			regs->ip, regs->sp, error_code);
		print_vma_addr(" in ", regs->ip);
		pr_cont("\n");
	}
#endif
```

그리고 주어진 신호를 중단된 프로세스로 보냅니다:

```C
force_sig_info(signr, info ?: SEND_SIG_PRIV, tsk);
```

이것이 `do_trap`의 끝입니다. 우리는 `DO_ERROR`매크로로 정의된 8가지 예외에 대한 일반적인 구현을 봤습니다. 이제 다른 예외 처리기를 보겠습니다.

이중 결함
--------------------------------------------------------------------------------

다음 예외는 `#DF` 또는 `Double fault`입니다. 이 예외는 프로세서가 이전 예외에 대한 예외 처리기를 호출하는 동안 두 번째 예외를 감지한 경우 일어납니다. 우리는 이전 파트에서 이 예외를 위한 트랩 게이트를 설정했습니다:

```C
set_intr_gate_ist(X86_TRAP_DF, &double_fault, DOUBLEFAULT_STACK);
```

참고로 이 예외는 `1` 인덱스를 가진 `DOUBLEFAULT_STACK`[인터럽트 스택 테이블](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)에서 실행됩니다.

```C
#define DOUBLEFAULT_STACK 1
```

`double_fault`는 이 예외를 위한 처리기로 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에서 정의됐습니다. `double_fault`처리기는 두 변수(예외와 중단된 프로세스를 설명하는 문자열, 다른 예외 처리)기의 정의를 통해 시작됩니다:

```C
static const char str[] = "double fault";
struct task_struct *tsk = current;
```

이중 결함 예외의 처리기는 두 부분으로 나누어 집니다. 첫번째 부분은 결함이 `espfix64`스택의 `non-IST`결함인지 확인하는 검사입니다. 실레조 `iret`명령은 `16`비트 세그먼트로 돌아갈 때 맨 아래 `16`비트만을 복구합니다. `espfix`피쳐는 이 문제를 해결합니다. 따라서 espfix64스택의 `non-IST`결함이라면 스택을 `General Protection Fault`처럼 수정합니다:

```C
struct pt_regs *normal_regs = task_pt_regs(current);

memmove(&normal_regs->ip, (void *)regs->sp, 5*8);
ormal_regs->orig_ax = 0;
regs->ip = (unsigned long)general_protection;
regs->sp = (unsigned long)&normal_regs->orig_ax;
return;
```

두 번째 경우에는 이전의 예외 처리기와 거의 동일한 작업을 수행합니다. 첫번째는 이전의 컨텍스트(우리의 경우 `user`)를 버리는 `ist_enter`함수의 호출입니다:

```C
ist_enter(regs);
```

다음으로 이전 처리기에서와 같이 `Double fault`예외 및 에러 코드의 벡터 번호로 중단된 프로세스를 채웁니다:

```C
tsk->thread.error_code = error_code;
tsk->thread.trap_nr = X86_TRAP_DF;
```

다음으로 이중 결함([PID](https://en.wikipedia.org/wiki/Process_identifier)번호, 레지스터 콘텐트)에 관한 유용한 정보를 출력합니다:

```C
#ifdef CONFIG_DOUBLEFAULT
	df_debug(regs, error_code);
#endif
```

그리고 죽습니다:

```
	for (;;)
		die(str, regs, error_code);
```

이것이 전부입니다.

사용할 수 없는 예외 처리기 장치
--------------------------------------------------------------------------------

다음 예외는 `#NM` 또는 `Device not available`입니다. `Device not available`예외는 다음 상황에 따라 발생할 수 있습니다:

* 프로세서는 [컨트롤 레지스터](https://en.wikipedia.org/wiki/Control_register) `cr0`의 EM플래그가 설정되어있는 동안 [x87 FPU](https://en.wikipedia.org/wiki/X87)부동 소수점 명령이 실행됩니다; 
* 프로세서는 레지스터 `cr0`의 `MP` 및 `TS`플래그가 설정되어있는 동안 `wait` 또는 `fwait` 명령이 실행됩니다;
* 프로세서는 컨트롤 레지스터 `cr0`의 `TS`플래그가 설정되고 `EM`플래그가 해제된 상태에서 [x87 FPU](https://en.wikipedia.org/wiki/X87), [MMX](https://en.wikipedia.org/wiki/MMX_%28instruction_set%29) 또는 [SSE](https://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)명령이 실행됩니다.

`Device not available`예외 처리기는 `do_device_not_available`함수이고 이것은 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)소스 코드 파일에도 정의되어 있습니다. 이 파트의 도입부에서 본 다른 트랩과 마찬가지로 이전 컨텍스트의 획득으로 시작하고 끝납니다:

```C
enum ctx_state prev_state;
prev_state = exception_enter();
...
...
...
exception_exit(prev_state);
```

다음 단계에서 `FPU`가 eager이 아닌지 확인합니다:

```C
BUG_ON(use_eager_fpu());
```

작업을 전환하거나 인터럽트할 때 `FPU`상태의 로딩을 피할 수 있습니다. 작업에서 사용할 경우, `Device not Available exception`예외가 발생합니다. 작업을 전환하는 중에 `FPU`상태를 로딩하면 `FPU`는  eager입니다. 다음 단계에서 `x87`부동 소수점 유닛이 있는지(플래그 클리어) 없는지(플래그 설정)를 보여줄 수 있는 `EM`플래그의 `cr0` 컨트롤 레지스터를 확인합니다:

```C
#ifdef CONFIG_MATH_EMULATION
	if (read_cr0() & X86_CR0_EM) {
		struct math_emu_info info = { };

		conditional_sti(regs);

		info.regs = regs;
		math_emulate(&info);
		exception_exit(prev_state);
		return;
	}
#endif
```

`x87`부동 소수점 유닛이 없으면, 인터럽트를 `conditional_sti`로 활성화하고, `math_emu_info`([arch/x86/include/asm/math_emu.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/math_emu.h)에서 정의됨)구조체를 인터럽트 작업의 레지스터로 채우고 [arch/x86/math-emu/fpu_entry.c](https://github.com/torvalds/linux/tree/master/arch/x86/math-emu/fpu_entry.c)에서 `math_emulate`함수를 호출합니다. 함수 이름에서 알 수 있듯이 `X87 FPU`유닛(`x87`에 관한 것은 특별 챕터에서 알 수 있습니다)을 모방합니다. 다른 방법으로 `X86_CR0_EM`플래그가 지워지면 `x87 FPU`유닛이 표시된다는 의미로, `fpustate`에서 `FPU`레지스터를 라이브 하드웨어 레지스터로 복사해 [arch/x86/kernel/fpu/core.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/fpu/core.c)에서 `fpu__restore`함수를 호출합니다. 그 다음 `FPU`명령이 사용될 수 있습니다:

```C
fpu__restore(&current->thread.fpu);
```

일반 보호 결함 예외 처리기
--------------------------------------------------------------------------------

다음 예외는 `#GP` 또는 `General protection fault`입니다. 이 예외는 프로세서가 `general-protection violations`라고 하는 보호 위반 클래스 중 하나를 감지했을 때 발생합니다:

* `cs`, `ds`, `es`, `fs` , `gs`세그먼트에 액세스할 때 세그먼트의 한계를 초과;
* 시스템 세그먼트를 위한 세그먼트 셀렉터로 `cs`, `ds`, `es`, `fs` , `gs`레지스터를 로드할 때;
* 권한 규칙을 위반하는 행위;
* 및 기타...

이 예외를 위한 예외 처리기는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에 있는 `do_general_protection`입니다. `do_general_protection`함수는 이전 컨텍스트를 가져오는 다른 예외처리기로 시작하고 끝납니다:

```C
prev_state = exception_enter();
...
exception_exit(prev_state);
```

그 다음 인터럽트과 비활성화되면 인터럽트를 활성화하고 [가상 8086](https://en.wikipedia.org/wiki/Virtual_8086_mode)모드에서 왔는지 확인합니다:

```C
conditional_sti(regs);

if (v8086_mode(regs)) {
	local_irq_enable();
	handle_vm86_fault((struct kernel_vm86_regs *) regs, error_code);
	goto exit;
}
```

long 모드는 이 모드를 지원하지 않으므로 이 경우에는 예외 처리를 고려하지 않습니다. 다음 단계에서 이전 모드가 커널 모드인지 확인하고 트랩을 고치려 시도합니다. 현재 일반 보호 결함 예외를 수리할 수 없는 경우 예외의 벡터 넘버와 에러 코드로 중단된 프로세스를 채우고 `notify_die`체인에 추가합니다;

```C
if (!user_mode(regs)) {
	if (fixup_exception(regs))
		goto exit;

	tsk->thread.error_code = error_code;
	tsk->thread.trap_nr = X86_TRAP_GP;
	if (notify_die(DIE_GPF, "general protection fault", regs, error_code,
		       X86_TRAP_GP, SIGSEGV) != NOTIFY_STOP)
		die("general protection fault", regs, error_code);
	goto exit;
}
```

예외를 고칠 수 있다면 예외 상태를 벗어나는 `exit`레이블로 이동합니다:

```C
exit:
	exception_exit(prev_state);
```

사용자 모드에서 온 경우 `do_trap`함수에서 수행한 것처럼 사용자 모드에서 중단된 프로세스로 `SIGSEGV`신호를 보냅니다:

```C
if (show_unhandled_signals && unhandled_signal(tsk, SIGSEGV) &&
		printk_ratelimit()) {
	pr_info("%s[%d] general protection ip:%lx sp:%lx error:%lx",
		tsk->comm, task_pid_nr(tsk),
		regs->ip, regs->sp, error_code);
	print_vma_addr(" in ", regs->ip);
	pr_cont("\n");
}

force_sig_info(SIGSEGV, SEND_SIG_PRIV, tsk);
```

이것이 전부입니다.

결론
--------------------------------------------------------------------------------

이것이 [인터럽트 및 인터럽트 처리기](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)챕터 다섯 번째 파트의 끝으로 이 파트에서는 몇개의 인터럽트 처리기의 구현을 봤습니다. 다음 파트에서는 인터럽트 및 예외 처리기를 계속하고 [마스크 불가능 인터럽트](https://en.wikipedia.org/wiki/Non-maskable_interrupt), 수학[보조 프로세서](https://en.wikipedia.org/wiki/Coprocessor)처리기 및 [SIMD](https://en.wikipedia.org/wiki/SIMD)보조프로세서 예외 처리 등을 볼 것입니다.

질문이나 제안 사항이 있다면 코멘트를 남기거나 [트위터](https://twitter.com/0xAX)로 보내주십시오.

**영어는 모국어가 아니어서 모든 불편한 점은 정말 죄송합니다. 실수를 발견하면 [linux-insides](https://github.com/0xAX/linux-insides)에서 수정사항이 포함된 PR을 보내주십시오.**


링크
--------------------------------------------------------------------------------

* [인터럽트 디스크립터 테이블](https://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [iret 명령](http://x86.renejeschke.de/html/file_module_x86_id_145.html)
* [GCC 매크로 연결](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html#Concatenation)
* [커널 패닉](https://en.wikipedia.org/wiki/Kernel_panic)
* [커널 oops](https://en.wikipedia.org/wiki/Linux_kernel_oops)
* [마스크 불가능 인터럽트](https://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [핫 플러그](https://en.wikipedia.org/wiki/Hot_swapping)
* [인터럽트 플래그](https://en.wikipedia.org/wiki/Interrupt_flag)
* [long 모드](https://en.wikipedia.org/wiki/Long_mode)
* [signal](https://en.wikipedia.org/wiki/Unix_signal)
* [printk](https://en.wikipedia.org/wiki/Printk)
* [보조프로세서](https://en.wikipedia.org/wiki/Coprocessor)
* [SIMD](https://en.wikipedia.org/wiki/SIMD)
* [인터럽트 스택 테이블](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [x87 FPU](https://en.wikipedia.org/wiki/X87)
* [컨트롤 레지스터](https://en.wikipedia.org/wiki/Control_register)
* [MMX](https://en.wikipedia.org/wiki/MMX_%28instruction_set%29)
* [이전 파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-4.html)
