인터럽트 및 인터럽트 처리. Part 3.
================================================================================

예외
--------------------------------------------------------------------------------

이것은 리눅스 커널에서 인터럽트 처리를 다루는 세 번째 파트입니다 [chapter](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)  그리고 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)에서  우리는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blame/master/arch/x86/kernel/setup.c) 소스코드의  `setup_arch` 함수에서 멈췄습니다. 

우리는 이미이 함수가 아키텍처 고유의 초기화를 실행한다는 것을 알고 있습니다. 우리의 경우`setup_arch` 함수는 [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처 관련 초기화를 합니다. `setup_arch`는 큰 기능이며, 이전 부분에서는 다음 두 가지 예외에 대한 두 가지 예외 처리기 설정을 중단했습니다.

* `# DB`-디버그 예외, 인터럽트 된 프로세스에서 디버그 핸들러로 제어를 전송합니다.
* `# BP`-`int 3` 명령으로 인한 중단 점 예외.

이러한 예외는`x86_64` 아키텍처가 [kgdb](https://en.wikipedia.org/wiki/KGDB)를 통한 디버깅을 위해 조기 예외 처리를 할 수 있도록합니다.

아시다시피`early_trap_init` 함수에서 이러한 예외 처리기를 설정했습니다.


```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```

[arch / x86 / kernel / traps.c](https://github.com/torvalds/linux/tree/master/arch/x86/kernel/traps.c)에서. 우리는 이미 앞부분에서`set_intr_gate_ist`와`set_system_intr_gate_ist` 함수의 구현을 보았고 이제이 두 예외 핸들러의 구현에 대해 살펴볼 것입니다.

디버그 및 중단 점 예외
--------------------------------------------------------------------------------

자, 우리는`# DB`와`# BP` 예외에 대해`early_trap_init` 함수에 예외 핸들러를 설정했으며 이제는 구현을 고려할 차례입니다. 그러나이 작업을 수행하기 전에 먼저 이러한 예외에 대한 세부 정보를 살펴 보겠습니다.

첫 번째 예외 인`# DB` 또는`debug` 예외는 디버그 이벤트가 발생할 때 발생합니다. 예를 들어 [debug register](http://en.wikipedia.org/wiki/X86_debug_register)의 내용을 변경해보십시오. 디버그 레지스터는 [Intel 80386](http://en.wikipedia.org/wiki/Intel_80386) 프로세서에서 시작하여`x86` 프로세서에 제공되는 특수 레지스터이며,이 CPU 확장의 이름에서 알 수 있듯이 이 레지스터의 주 목적은 디버깅입니다.

이 레지스터를 사용하면 코드에서 중단 점을 설정하고 추적하기 위해 데이터를 읽거나 쓸 수 있습니다. 디버그 레지스터는 권한 모드에서만 액세스 할 수 있으며 다른 권한 수준에서 실행할 때 디버그 레지스터를 읽거나 쓰려고하면 [일반 보호 오류](https://en.wikipedia.org/wiki/General_protection_fault) 예외가 발생합니다. . 그래서 우리는`# DB` 예외에 `set_intr_gate_ist`를 사용했지만 `set_system_intr_gate_ist`는 사용하지 않았습니다.

`# DB` 예외의 verctor 수는`1 '(우리는`X86_TRAP_DB`로 전달)이며 사양에서 읽을 수 있듯이이 예외에는 오류 코드가 없습니다.


```
+-----------------------------------------------------+
|Vector|Mnemonic|Description         |Type |Error Code|
+-----------------------------------------------------+
|1     | #DB    |Reserved            |F/T  |NO        |
+-----------------------------------------------------+
```

두 번째 예외는 프로세서가 [int 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3) 명령을 실행할 때 발생하는 `# BP` 또는 `breakpoint` 예외입니다. `DB` 예외와 달리 `# BP` 예외는 사용자 공간에서 발생할 수 있습니다. 코드의 어느 곳에 나 추가 할 수 있습니다. 예를 들어 간단한 프로그램을 살펴 보겠습니다.

```C
// breakpoint.c
#include <stdio.h>

int main() {
    int i;
    while (i < 6){
	    printf("i equal to: %d\n", i);
	    __asm__("int3");
		++i;
    }
}
```

이 프로그램을 컴파일하고 실행하면 다음과 같은 결과가 나타납니다.

```
$ gcc breakpoint.c -o breakpoint
i equal to: 0
Trace/breakpoint trap
```

그러나 gdb로 실행하면 중단 점이 표시되고 프로그램을 계속 실행할 수 있습니다.

```
$ gdb breakpoint
...
...
...
(gdb) run
Starting program: /home/alex/breakpoints 
i equal to: 0

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
(gdb) c
Continuing.
i equal to: 1

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
(gdb) c
Continuing.
i equal to: 2

Program received signal SIGTRAP, Trace/breakpoint trap.
0x0000000000400585 in main ()
=> 0x0000000000400585 <main+31>:	83 45 fc 01	add    DWORD PTR [rbp-0x4],0x1
...
...
...
```

이 순간부터 우리는이 두 가지 예외에 대해 약간 알고 있으며 처리기를 고려할 수 있습니다.

예외 처리기 전 준비
--------------------------------------------------------------------------------

앞에서 언급했듯이 `set_intr_gate_ist` 및 `set_system_intr_gate_ist` 함수는 두 번째 매개 변수에서 예외 처리기의 주소를 사용합니다. 두 가지 예외 처리기는 다음과 같습니다.

* `debug`;
* `int3`.

C 코드에는 이러한 기능이 없습니다. 이 모든 것은 커널의`* .c / *. h` 파일에서 찾을 수 있습니다. [arch / x86 / include / asm / traps.h](https://github.com/torvalds/linux/tree/master/arch/x86/include/asm/traps.h) 커널 헤더 파일 :
```C
asmlinkage void debug(void);
```

그리고

```C
asmlinkage void int3(void);
```

이 함수들의 정의에서`asmlinkage` 지시어에 주목할 수 있습니다. 지시문은 [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection)의 특수 지정자입니다. 실제로 어셈블리에서 호출되는 'C'함수의 경우 함수 호출 규칙을 명시 적으로 선언해야합니다. 우리의 경우,`asmlinkage` 서술자로 만들어진 함수라면,`gcc`는 함수를 컴파일하여 스택에서 파라미터를 가져옵니다.

따라서 두 처리기 모두 [arch / x86 / entry / entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) 어셈블리 소스 코드 파일에 정의되어 있습니다. `idtentry` 매크로로 :

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

그리고

```assembly
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

각 예외 처리기는 두 부분으로 구성 될 수 있습니다. 첫 번째 부분은 일반 부분이며 모든 예외 처리기에서 동일합니다. 예외 처리기는 스택에 [범용 레지스터](https://en.wikipedia.org/wiki/Processor_register)를 저장하고 사용자 공간에서 예외가 발생한 경우 커널 스택으로 전환하고 예외의 두 번째 부분으로 제어를 전송해야합니다. 매니저. 예외 처리기의 두 번째 부분은 특정 예외에 따라 특정 작업을 수행합니다. 예를 들어 페이지 오류 예외 처리기는 지정된 주소에 대한 가상 페이지를 찾아야하고 잘못된 opcode 예외 처리기는`SIGILL` [signal](https://en.wikipedia.org/wiki/Unix_signal) 등을 보내야합니다.

방금 봤 듯이, 예외 처리기는 [arch / x86 / kernel / entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d93/arch/x86/kernel/entry_64.S)의 `idtentry` 매크로 정의에서 시작합니다. 어셈블리 소스 코드 파일이므로이 매크로의 구현을 살펴 보겠습니다. 보시다시피, 'idtentry'매크로는 다섯 가지 인수를 취합니다.

* `sym`-예외 처리기의 엔트리가 될`.globl name`으로 전역 기호를 정의합니다.
* `do_sym`-예외 핸들러의 2 차 엔트리를 나타내는 심볼 이름;
* `has_error_code`-예외 오류 코드의 존재에 관한 정보.

마지막 두 매개 변수는 선택 사항입니다.

* `paranoid`-현재 모드를 확인하는 방법을 보여줍니다 (나중에 자세히 설명 할 것입니다).
* `shift_ist`-`Interrupt Stack Table`에서 실행되는 예외임을 보여줍니다.

`.idtentry` 매크로의 정의는 다음과 같습니다.

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
...
...
...
END(\sym)
.endm
```

`idtentry`매크로의 내부를 고려하기 전에 예외가 발생할 때 스택 상태를 알아야합니다. [Intel® 64 및 IA-32 아키텍처 소프트웨어 개발자 매뉴얼 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html) 에서 예외 발생시 스택 상태는 다음과 같습니다.

```
    +------------+
+40 | %SS        |
+32 | %RSP       |
+24 | %RFLAGS    |
+16 | %CS        |
 +8 | %RIP       |
  0 | ERROR CODE | <-- %RSP
    +------------+
```

이제 우리는`idtmacro`의 구현을 고려할 수 있습니다. `# DB` 및`BP` 예외 핸들러는 모두 다음과 같이 정의됩니다.

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
idtentry int3 do_int3 has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

이러한 정의를 살펴보면 컴파일러가 `debug`와 `int3` 이름을 가진 두 개의 루틴을 생성 할 것이고, 이들 예외 핸들러는 일부 준비 후에 `do_debug`와 `do_int3` 보조 핸들러를 호출 할 것입니다. 세 번째 매개 변수는 오류 코드의 존재를 정의하며 예외가 없는 것처럼 볼 수 있습니다. 위의 다이어그램에서 볼 수 있듯이 프로세서는 예외가 제공하는 경우 오류 코드를 스택에 푸시합니다. 이 경우 `debug` 및`int3` 예외에는 오류 코드가 없습니다. 스택은 오류 코드를 제공하는 예외와 그렇지 않은 예외를 다르게 볼 수 있기 때문에 약간의 어려움이 발생할 수 있습니다. 그렇기 때문에 예외가 제공하지 않으면 'idtentry'매크로의 구현이 가짜 오류 코드를 스택에 넣는 것부터 시작합니다.

```assembly
.ifeq \has_error_code
    pushq	$-1
.endif
```

그러나 가짜 오류 코드 일뿐입니다. 또한 `-1`은 유효하지 않은 시스템 호출 번호를 나타내므로 시스템 호출 재시작 로직이 트리거되지 않습니다.

`idtentry` 매크로 `shift_ist`와 `paranoid`의 마지막 두 매개 변수는 `Interrupt Stack Table`의 스택에서 실행되는 예외 처리기를 알 수 있습니다. 이미 시스템의 각 커널 스레드에 자체 스택이 있다는 것을 알고있을 것입니다. 이러한 스택 외에도 시스템의 각 프로세서와 관련된 일부 특수 스택이 있습니다. 이러한 스택 중 하나는 예외 스택입니다. [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처는 '인터럽트 스택 테이블'이라는 특별한 기능을 제공합니다. 이 기능을 사용하면 `double fault` 등과 같은 원자 예외와 같은 지정된 이벤트에 대해 새 스택으로 전환 할 수 있습니다. `shift_ist` 매개 변수를 사용하면 예외 처리기를 위해 IST 스택을 켜야하는지 알 수 있습니다.

두 번째 매개 변수 인 `paranoid`는 사용자 공간에서 예외 처리기가 아닌지 여부를 알 수있는 방법을 정의합니다. 이를 결정하는 가장 쉬운 방법은 CS 세그먼트 레지스터의 `CPL` 또는 `Current Privilege Level`을 통하는 것입니다. `3`과 같으면 사용자 공간에서 왔고, 0이면 커널 공간에서 왔습니다.

```
testl $3,CS(%rsp)
jnz userspace
...
...
...
// we are from the kernel space
```

그러나 불행히도이 방법은 100 % 보증하지 않습니다. 커널 문서에 설명 된대로 :

> 우리가 NMI / MCE / DEBUG / 슈퍼 아토믹 엔트리 컨텍스트에 있다면,
> 일반 항목에 CS를 쓴 직후에 트리거되었을 수 있습니다.
> 스택이지만 SWAPGS를 실행하기 전에 확인하는 유일한 안전한 방법
> GS의 경우 느린 방법 인 RDMSR입니다.

다시 말해 `NMI`는 [swapgs](http://www.felixcloutier.com/x86/SWAPGS.html) 명령의 중요 섹션에서 발생할 수 있습니다. 이런 식으로 CPU 별 영역의 시작에 대한 포인터를 저장하는 `MSR_GS_BASE` [모델 특정 레지스터](https://en.wikipedia.org/wiki/Model-specific_register)의 값을 확인해야합니다. 따라서 사용자 공간에서 왔는지 여부를 확인하려면 MSR_GS_BASE 모델 특정 레지스터의 값을 확인해야하며 음수이면 커널 공간에서 왔으며 다른 방법으로 사용자 공간에서 나왔습니다.

```assembly
movl $MSR_GS_BASE,%ecx
rdmsr
testl %edx,%edx
js 1f
```

처음 두 줄의 코드에서 우리는 `MSR_GS_BASE` 모델 특정 레지스터의 값을 `edx : eax` 쌍으로 읽습니다. 사용자 공간에서 음의 값을 `gs`로 설정할 수 없습니다. 그러나 우리는 물리 메모리의 직접 매핑이`0xffff880000000000` 가상 주소에서 시작한다는 것을 알고 있습니다. 이런 방식으로 `MSR_GS_BASE`는 `0xffff880000000000`부터 `0xffffc7ffffffffff`까지의 주소를 포함합니다. `rdmsr` 명령어가 실행 된 후, `% edx` 레지스터에서 가능한 가장 작은 값은 -0xffff8800이며, 부호없는 4 바이트에서 -30720입니다. 이것이 per-cpu 영역의 시작을 가리키는 커널 공간 `gs`가 음의 값을 포함하는 이유입니다.

스택에서 가짜 오류 코드를 푸시 한 후 다음과 같이 범용 레지스터를 위한 공간을 할당해야합니다.

```assembly
ALLOC_PT_GPREGS_ON_STACK
```

[arch / x86 / entry / calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) 헤더 파일에 정의 된 매크로는 스택에 15 * 8 바이트의 공간을 할당하여 범용 레지스터를 유지합니다.

```assembly
.macro ALLOC_PT_GPREGS_ON_STACK addskip=0
    addq	$-(15*8+\addskip), %rsp
.endm
```

따라서 `ALLOC_PT_GPREGS_ON_STACK`을 실행 한 후 스택은 다음과 같습니다.

```
     +------------+
+160 | %SS        |
+152 | %RSP       |
+144 | %RFLAGS    |
+136 | %CS        |
+128 | %RIP       |
+120 | ERROR CODE |
     |------------|
+112 |            |
+104 |            |
 +96 |            |
 +88 |            |
 +80 |            |
 +72 |            |
 +64 |            |
 +56 |            |
 +48 |            |
 +40 |            |
 +32 |            |
 +24 |            |
 +16 |            |
  +8 |            |
  +0 |            | <- %RSP
     +------------+
```

범용 레지스터를위한 공간을 할당 한 후 예외가 사용자 공간에서 발생했는지 여부를 이해하기 위해 몇 가지 검사를 수행하고, 그렇다면, 중단 된 프로세스 스택으로 돌아가거나 예외 스택을 유지해야합니다.

```assembly
.if \paranoid
    .if \paranoid == 1
	    testb	$3, CS(%rsp)
	    jnz	1f
	.endif
	call	paranoid_entry
.else
	call	error_entry
.endif
```

물론 이 모든 경우를 고려해 봅시다.

사용자 공간에서의 예외 발생
--------------------------------------------------------------------------------

첫 번째로 예외에 `debug`및 `int3` 예외와 같이 예외가 `paranoid = 1`인 경우를 생각해 봅시다. 이 경우 우리는 `CS` 세그먼트 레지스터에서 셀렉터를 확인하고 사용자 공간에서 왔거나 `paranoid_entry`가 다른 방식으로 호출되면 `1f`레이블로 점프합니다.

사용자 공간에서 예외 처리기로 온 첫 번째 경우를 고려해 봅시다. 위에서 설명한 바와 같이 우리는`1` 레이블로 점프해야합니다. `1` 라벨은

```assembly
call	error_entry
```

모든 범용 레지스터를 스택의 이전에 할당 된 영역에 저장하는 루틴 :

```assembly
SAVE_C_REGS 8
SAVE_EXTRA_REGS 8
```

이 두 매크로는 [arch / x86 / entry / calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/calling.h) 헤더 파일에 정의되어 있으며 이동 만합니다. 범용의 값은 스택의 특정 위치에 등록됩니다.

```assembly
.macro SAVE_EXTRA_REGS offset=0
	movq %r15, 0*8+\offset(%rsp)
	movq %r14, 1*8+\offset(%rsp)
	movq %r13, 2*8+\offset(%rsp)
	movq %r12, 3*8+\offset(%rsp)
	movq %rbp, 4*8+\offset(%rsp)
	movq %rbx, 5*8+\offset(%rsp)
.endm
```

`SAVE_C_REGS`와`SAVE_EXTRA_REGS`를 실행하면 스택은 다음과 같이 보입니다.

```
     +------------+
+160 | %SS        |
+152 | %RSP       |
+144 | %RFLAGS    |
+136 | %CS        |
+128 | %RIP       |
+120 | ERROR CODE |
     |------------|
+112 | %RDI       |
+104 | %RSI       |
 +96 | %RDX       |
 +88 | %RCX       |
 +80 | %RAX       |
 +72 | %R8        |
 +64 | %R9        |
 +56 | %R10       |
 +48 | %R11       |
 +40 | %RBX       |
 +32 | %RBP       |
 +24 | %R12       |
 +16 | %R13       |
  +8 | %R14       |
  +0 | %R15       | <- %RSP
     +------------+
```

커널이 범용 레지스터를 스택에 저장 한 후 다음을 사용하여 사용자 공간에서 다시 왔는지 확인해야합니다.

```assembly
testb	$3, CS+8(%rsp)
jz	.Lerror_kernelspace
```

문서에 설명 된 것처럼 `% RIP`가 잘린 것으로보고 된 경우 오류가 발생할 수 있기 때문입니다. 어쨌든 두 경우 모두 [SWAPGS](http://www.felixcloutier.com/x86/SWAPGS.html) 명령이 실행되고 `MSR_KERNEL_GS_BASE` 및 `MSR_GS_BASE`의 값이 교환됩니다. 이 시점부터 `% gs` 레지스터는 커널 구조의 기본 주소를 가리 킵니다. 따라서 `SWAPGS`명령이 호출되었으며 `error_entry`라우팅의 주요 지점이었습니다.

이제 우리는 `idtentry` 매크로로 돌아갈 수 있습니다. `error_entry` 호출 후 다음과 같은 어셈블러 코드가 표시 될 수 있습니다.

```assembly
movq	%rsp, %rdi
call	sync_regs
```

여기에 우리는`sync_regs`의 첫 번째 인자가 될 스택 포인터`% rdi` 레지스터의 기본 주소를 넣습니다 ([x86_64 ABI](https://www.uclibc.org/docs/psABI-x86_64.pdf)) [arch / x86 / kernel / traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c) 소스 코드 파일에 정의 된이 함수를 호출하고 호출하십시오.

```C
asmlinkage __visible notrace struct pt_regs *sync_regs(struct pt_regs *eregs)
{
	struct pt_regs *regs = task_pt_regs(current);
	*regs = *eregs;
	return regs;
}
```

이 함수는 [arch/x86/include/asm/processor.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d93/arch/x86/include/asm/processor.h)에 정의 된 `task_ptr_regs` 매크로의 결과를 가져옵니다. 헤더 파일을 스택 포인터에 저장하고 반환하고, `task_ptr_regs` 매크로는 일반 커널 스택에 대한 포인터를 나타내는 `thread.sp0`의 주소로 확장됩니다.

```C
#define task_pt_regs(tsk)       ((struct pt_regs *)(tsk)->thread.sp0 - 1)
```

우리가 사용자 공간에서 왔을 때, 이것은 예외 처리기가 실제 프로세스 컨텍스트에서 실행될 것임을 의미합니다. `sync_regs`에서 스택 포인터를 얻은 후 스택을 전환합니다.

```assembly
movq	%rax, %rsp
```

The last two steps before an exception handler will call secondary handler are:

1. Passing pointer to `pt_regs` structure which contains preserved general purpose registers to the `%rdi` register:

```assembly
movq	%rsp, %rdi
```

as it will be passed as first parameter of secondary exception handler.

2. Pass error code to the `%rsi` register as it will be second argument of an exception handler and set it to `-1` on the stack for the same purpose as we did it before - to prevent restart of a system call:

```
.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```

Additionally you may see that we zeroed the `%esi` register above in a case if an exception does not provide error code. 

In the end we just call secondary exception handler:

```assembly
call	\do_sym
```

which:

```C
dotraplinkage void do_debug(struct pt_regs *regs, long error_code);
```

will be for `debug` exception and:

```C
dotraplinkage void notrace do_int3(struct pt_regs *regs, long error_code);
```

will be for `int 3` exception. In this part we will not see implementations of secondary handlers, because of they are very specific, but will see some of them in one of next parts.

We just considered first case when an exception occurred in userspace. Let's consider last two.

An exception with paranoid > 0 occurred in kernelspace
--------------------------------------------------------------------------------

In this case an exception was occurred in kernelspace and `idtentry` macro is defined with `paranoid=1` for this exception. This value of `paranoid` means that we should use slower way that we saw in the beginning of this part to check do we really came from kernelspace or not. The `paranoid_entry` routing allows us to know this:

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

As you may see, this function represents the same that we covered before. We use second (slow) method to get information about previous state of an interrupted task. As we checked this and executed `SWAPGS` in a case if we came from userspace, we should to do the same that we did before: We need to put pointer to a structure which holds general purpose registers to the `%rdi` (which will be first parameter of a secondary handler) and put error code if an exception provides it to the `%rsi` (which will be second parameter of a secondary handler):

```assembly
movq	%rsp, %rdi

.if \has_error_code
	movq	ORIG_RAX(%rsp), %rsi
	movq	$-1, ORIG_RAX(%rsp)
.else
	xorl	%esi, %esi
.endif
```

The last step before a secondary handler of an exception will be called is cleanup of new `IST` stack fram:

```assembly
.if \shift_ist != -1
	subq	$EXCEPTION_STKSZ, CPU_TSS_IST(\shift_ist)
.endif
```

You may remember that we passed the `shift_ist` as argument of the `idtentry` macro. Here we check its value and if its not equal to `-1`, we get pointer to a stack from `Interrupt Stack Table` by `shift_ist` index and setup it.

In the end of this second way we just call secondary exception handler as we did it before:

```assembly
call	\do_sym
```

The last method is similar to previous both, but an exception occured with `paranoid=0` and we may use fast method determination of where we are from.

Exit from an exception handler
--------------------------------------------------------------------------------

After secondary handler will finish its works, we will return to the `idtentry` macro and the next step will be jump to the `error_exit`:

```assembly
jmp	error_exit
```

routine. The `error_exit` function defined in the same [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S) assembly source code file and the main goal of this function is to know where we are from (from userspace or kernelspace) and execute `SWPAGS` depends on this. Restore registers to previous state and execute `iret` instruction to transfer control to an interrupted task.

That's all.

Conclusion
--------------------------------------------------------------------------------

It is the end of the third part about interrupts and interrupt handling in the Linux kernel. We saw the initialization of the [Interrupt descriptor table](https://en.wikipedia.org/wiki/Interrupt_descriptor_table) in the previous part with the `#DB` and `#BP` gates and started to dive into preparation before control will be transferred to an exception handler and implementation of some interrupt handlers in this part. In the next part we will continue to dive into this theme and will go next by the `setup_arch` function and will try to understand interrupts handling related stuff.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

Links
--------------------------------------------------------------------------------

* [Debug registers](http://en.wikipedia.org/wiki/X86_debug_register)
* [Intel 80385](http://en.wikipedia.org/wiki/Intel_80386)
* [INT 3](http://en.wikipedia.org/wiki/INT_%28x86_instruction%29#INT_3)
* [gcc](http://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [TSS](http://en.wikipedia.org/wiki/Task_state_segment)
* [GNU assembly .error directive](https://sourceware.org/binutils/docs/as/Error.html#Error)
* [dwarf2](http://en.wikipedia.org/wiki/DWARF)
* [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html)
* [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [system call](http://en.wikipedia.org/wiki/System_call)
* [swapgs](http://www.felixcloutier.com/x86/SWAPGS.html)
* [SIGTRAP](https://en.wikipedia.org/wiki/Unix_signal#SIGTRAP)
* [Per-CPU variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [kgdb](https://en.wikipedia.org/wiki/KGDB)
* [ACPI](https://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)
