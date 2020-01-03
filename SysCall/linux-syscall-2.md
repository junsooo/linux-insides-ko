리눅스 커널에서의 시스템 호출. Part 2.
================================================================================

리눅스 커널은 시스템 호출을 어떻게 처리합니까 
--------------------------------------------------------------------------------

이전 [파트](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html)는 리눅스 커널에서 [시스템 호출](https://en.wikipedia.org/wiki/System_call) 개념을 설명하는 이 챕터의 첫 번째 파트입니다.
이전 파트에서 우리는 리눅스 커널에서 시스템 호출이 무엇인지 배우고, 일반적인 시스템 작동을 배웠습니다. 이것은 사용자 공간 관점에서 소개되었으며, [write](http://man7.org/linux/man-pages/man2/write.2.html)시스템 호출 구현의 일부가 논의됩니다. 이 파트에서는 리눅스 커널 코드로 이동하기 전에 몇 가지 이론부터 시작해 시스템 호출을 계속 살펴봅니다.

사용자 응용 프로그램은 응용 프로그램에서 직접 시스템을 호출하지 않습니다. 우리는 다음과 같은 `Hello world!`프로그램을 작성하지 않았습니다:

```C
int main(int argc, char **argv)
{
	...
	...
	...
	sys_write(fd1, buf, strlen(buf));
	...
	...
}
```

우리는 [C 표준 라이브러리](https://en.wikipedia.org/wiki/GNU_C_Library)의 도움으로 비슷한 것을 사용할 수 있으며 다음과 같이 보일 것입니다:

```C
#include <unistd.h>

int main(int argc, char **argv)
{
	...
	...
	...
	write(fd1, buf, strlen(buf));
	...
	...
}
```

어쨌든, `write`는 직접적인 시스템 호출이 아니며 커널 함수가 아닙니다. 응용프로그램은 범용 레지스터를 올바른 순서, 올바른 값으로 채우고  `syscall`명령을 사용해 실제 시스템 호출을 작성해야합니다. 이 파트에서는 `syscall`명령이 프로세서와 만났을 때 리눅스 커널에서 어떤일이 일어나는지 살펴 볼 것입니다.

시스템 호출 테이블의 초기화
--------------------------------------------------------------------------------

이전 파트에서 우리는 시스템 호출 개념이 인터럽트와 매우 유사하다는 것을 알았습니다. 또한, 시스템 호출은 소프트웨어 인터럽트로 구현됩니다. 따라서 프로세서가 사용자 응용프로그램에서 `syscall`명령을 처리할 때, 이 명령은 예외처리기에 제어를 전달하는 예외를 초래합니다. 아시다시피, 모든 예외 처리기(또는 예외에 반응하는 커널 [C](https://en.wikipedia.org/wiki/C_%28programming_language%29)함수)는 커널 코드에 배치됩니다. 그런데 리눅스 커널은 관련된 시스템 호출에 필요한 시스템 호출 처리기의 주소를 어떻게 검색할까요? 리눅스 커널은 `system call table`이라 불리는 특별한 테이블을 포함합니다. 시스템 호출 테이블은 [arch/x86/entry/syscall_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscall_64.c)소스 코드 파일에서 정의된 리눅스 커널 내의 `sys_call_table`배열로 표시됩니다. 구현을 살펴봅시다:

```C
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
    #include <asm/syscalls_64.h>
};
```

보시다시피, `sys_call_table`은 `__NR_syscall_max` 매크로에서 주어진 [아키텍처](https://en.wikipedia.org/wiki/List_of_CPU_architectures)를 위한 시스템 호출의 최대 개수를 표시하는 `__NR_syscall_max + 1`크기의 배열입니다. 이 책은 [x86_64](https://en.wikipedia.org/wiki/X86-64)아키텍처에 관한 것이기 때문에 우리의 경우 `__NR_syscall_max`은 `322`이고 글을 쓰는 시점(현재 리눅스 커널 버전은 `4.2.0-rc8+`)에서 올바른 숫자입니다. 우리는 include/generated/asm-offsets.h 커널을 편집하는 동안 [Kbuild](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt)에서 생성된 헤더 파일의 매크로를 볼 것입니다:

```C
#define __NR_syscall_max 322
```

[arch/x86/entry/syscalls/syscall_64.tbl](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl#L331)에 `x86_64`을 위한 시스템 호출의 동일한 수가 있습니다. 여기에는 두 가지 중요한 주게가 있습니다; `sys_call_table`배열의 타입과, 배열 내의 요소의 초기화입니다. 우선, 타입입니다. `sys_call_ptr_t`은 포인터로 시스템 호출 테이블을 나타냅니다. 이것은 아무것도 반환하지 않고 인수를 사용하지 않는 함수 포인터에 대한 [typedef](https://en.wikipedia.org/wiki/Typedef)로 정의됩니다:

```C
typedef void (*sys_call_ptr_t)(void);
```

두 번째는 `sys_call_table`배열의 초기화 입니다. 위의 코드에서 볼 수 있듯이, 시스템 호출 처리기에 대한 포인터를 포함하는 배열의 모든 요소는 `sys_ni_syscall`을 가리킵니다. `sys_ni_syscall`함수는 구현되지 않은 시스템 호출을 나타냅니다. 우선, `sys_call_table`배열의 모든 요소가 구현되지 않은 시스템 호출을 가리킵니다. 이것은 나중에 덧붙여질 시스템 호출 처리기에 대한 포인터의 저장영역만을 초기화하므로 올바른 초기 동작입니다.  `sys_ni_syscall`의 구현은 매우 쉽습니다. 우리의 경우 이것은 [-errno](http://man7.org/linux/man-pages/man3/errno.3.html) 또는 `-ENOSYS`을 반환합니다:

```C
asmlinkage long sys_ni_syscall(void)
{
	return -ENOSYS;
}
```

`-ENOSYS`오류가 있음을 알려줍니다:

```
ENOSYS          Function not implemented (POSIX.1)
```

`sys_call_table`의 초기화에서 `...`의 참고사항입니다. 우리는 그것을 [Designated Initializers](https://gcc.gnu.org/onlinedocs/gcc/Designated-Inits.html)라는[GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)컴파일러의 확장으로 실행할 수 있습니다. 이 확장은 고정되지 않은 순서로 요소를 초기화 하게 해줍니다. 보시다시피, 배열의 끝에 `asm/syscalls_64.h`헤더를 포합시킵니다. 이 헤더 파일은 [arch/x86/entry/syscalls/syscalltbl.sh](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscalltbl.sh) 특수 스크립트에 의해 생성됐으며 [syscall table](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl)에서 헤더 파일을 생성합니다. `asm/syscalls_64.h`은 다음 매크로의 정의를 포함합니다:

```C
__SYSCALL_COMMON(0, sys_read, sys_read)
__SYSCALL_COMMON(1, sys_write, sys_write)
__SYSCALL_COMMON(2, sys_open, sys_open)
__SYSCALL_COMMON(3, sys_close, sys_close)
__SYSCALL_COMMON(5, sys_newfstat, sys_newfstat)
...
...
...
```

`__SYSCALL_COMMON`매크로는 동일한 소스 코드 [파일](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscall_64.c)에 정의되었고 함수의 정의로 확장된 `__SYSCALL_64`매크로를 확장합니다:

```C
#define __SYSCALL_COMMON(nr, sym, compat) __SYSCALL_64(nr, sym, compat)
#define __SYSCALL_64(nr, sym, compat) [nr] = sym,
```

이후, `sys_call_table`은 다음과 같은 형태를 취합니다:

```C
asmlinkage const sys_call_ptr_t sys_call_table[__NR_syscall_max+1] = {
	[0 ... __NR_syscall_max] = &sys_ni_syscall,
	[0] = sys_read,
	[1] = sys_write,
	[2] = sys_open,
	...
	...
	...
};
```

그 후에 구현되지 않은 시스템 호출을 가리키는 모든 요소는 위에서 본 것처럼 `-ENOSYS`을 반환하는 `sys_ni_syscall`함수의 주소를 포함할 것이며, 다른 요소는 `sys_syscall_name`함수를 가리킬 것입니다.

이 시점에서, 우리는 시스템 호출 테이블을 채웠으며 리눅스 커널은 각 시스템 호출 처리기의 위치를 알고 있습니다. 그러나 리눅스 커널은 사용자 공간 응용프로그램에서 시스템 호출을 처리하도록 지시받은 직후 `sys_syscall_name`함수를 호출하지 않습니다. 인터럽트 및 인터럽트 처리에 관한 [챕터](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)를 떠올려보십시오. 리눅스 커널이 인터럽트 처리의 제어를 얻었을 때, 사용자 공간 레지스터 저장, 새로운 스택으로 전환하기 및 인터럽트 처리기를 호출하기 전에 더 많은 작업과 같은 몇 가지 준비를 해야 했습니다. 이것은 시스템 호출 처리와 동일한 상황입니다. 시스템 호출을 처리하기 위한 준비가 첫 번째지만, 리눅스 커널이 이러한 준비를 시작하기 전에 시스템 호출의 엔트리 포인트가 초기화되어야 하며 리눅스 커널만이 이 준비를 하는 방법을 알아야합니다. 다음 단락에서는 리눅스 커널에서 시스템 호출 항목을 초기화하는 프로세스를 볼 수 있습니다.

시스템 호출 엔트리의 초기화
--------------------------------------------------------------------------------

시스템에서 시스템 호출이 일어났을 때, 이것의 처리를 시작하는 코드의 첫 바이트는 어디에 있습니까? 인텔 메뉴얼 [64-ia-32-architectures-software-developer-vol-2b-manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)에서 읽을 수 있습니다:

```
SYSCALL은 0레벨 특전 OS system-call 처리기를 부릅니다.
이는 IA32_LSTAR MSR에서 RIP을 로드합니다.
```

즉, 시스템 호출 엔트리를 `IA32_LSTAR`[모델 특정 레지스터](https://en.wikipedia.org/wiki/Model-specific_register)에 넣어야합니다. 이 작업은 리눅스 커널 초기화 프로세스 중에 발생한 장소를 얻습니다. 리눅스 커널에서 인터럽트 및 인터럽트 처리를 설명하는 챕터의 네 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-4.html)를 읽었다면 리눅스 커널이 프로세스를 초기화하는 동안 `trap_init`함수를 호출하는 것을 알 것입니다. 이 함수는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c)소스 코드 파일에서 정의되었으며 분리 오류, [보조프로세서](https://en.wikipedia.org/wiki/Coprocessor)오류 등과 같은 `non-early`예외 처리기의 초기화를 실행합니다. `non-early`예외 처리기의 초기화 이외에도, 이 함수는 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/blob/arch/x86/kernel/cpu/common.c) 소스 코드 파일에서 `per-cpu`상태의 초기화뿐 아니라, 동일한 소스 코드 파일에서 `syscall_init`함수를 호출하는 `cpu_init`함수를 호출합니다.

이 함수는 시스템 호출 엔트리 포인트의 초기화를 수행합니다. 이 함수의 구현을 살펴봅시다. 이 함수는 매개변수를 사용하지 않으며 먼저 두 가지 모델 특정 레지스터를 채웁니다:

```C
wrmsrl(MSR_STAR,  ((u64)__USER32_CS)<<48  | ((u64)__KERNEL_CS)<<32);
wrmsrl(MSR_LSTAR, entry_SYSCALL_64);
```

첫 번째 모델 특정 레지스터 `MSR_STAR`는 사용자 코드 세그먼트의 `63:48`비트를 포함합니다. 이 비트들은 시스템 호출에서 특권과 관련된 사용자 코드를 반환하는 기능을 제공하는 `sysret`명령을 위한 `CS` 및 `SS`세그먼트 레지스터를 부를 것입니다. 또한 `MSR_STAR`는 사용자 공간 응용프로그램에서 시스템 호출이 실행될 때 `CS` 및 `SS`세그먼트 레지스터를 위한 기본 선택자로 사용될 커널 코드에서 `47:32`비트를 포함합니다. 코드의 두 번째 줄에서 우리는 `MSR_LSTAR`레지스터를 시스템 호출 엔트리를 나타내는 `entry_SYSCALL_64`심볼로 채웁니다.`entry_SYSCALL_64`은 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S)어셈블리 파일에서 정의되었으며 시스템 호출 처리기가 실행(위를 보십시오. 이미 이 준비를 적었습니다)되기 전에 수행되는 준비와 관련된 코드를 포함합니다. 우리는 이제 `entry_SYSCALL_64`를 고려하지 않아도 되지만, 이 챕터의 끝에서 반환할 것입니다.

시스템 호출을 위한 엔트리 포인트를 설정한 다음, 다음 모델 특정 레지스터를 설정해야합니다:

* `MSR_CSTAR` - 호환성 모드 호출자를 위한 목표`rip`;
* `MSR_IA32_SYSENTER_CS` - `sysenter`명령을 위한 목표`cs`;
* `MSR_IA32_SYSENTER_ESP` - `sysenter`명령을 위한 목표`esp`;
* `MSR_IA32_SYSENTER_EIP` - `sysenter`명령을 위한 목표`eip`.

이 모델 특정 레지스터들의 값은 `CONFIG_IA32_EMULATION` 커널 구성 옵션에 의존합니다. 커널 구성 옵션이 활성화되면, 레거시 32비트 프로그램을 64비트 커널에서 실행할 수 있습니다. 첫 번째 경우, `CONFIG_IA32_EMULATION` 커널 구성 옵션이 활성화되면, 이 모델 특정 레지스터 들을 시스템이 호환 모드를 호출하기 위한 엔트리 포인트로 채웁니다:

```C
wrmsrl(MSR_CSTAR, entry_SYSCALL_compat);
```

그리고 커널 코드 세그먼트를 사용해 스택 포인터에 0을 넣고 `entry_SYSENTER_compat`심볼의 주소를 [명령 포인터](https://en.wikipedia.org/wiki/Program_counter)에 적습니다:

```C
wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)__KERNEL_CS);
wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
wrmsrl_safe(MSR_IA32_SYSENTER_EIP, (u64)entry_SYSENTER_compat);
```

다른 방법으로, `CONFIG_IA32_EMULATION`커널 구성 옵션이 비활성화되면 `ignore_sysret`심볼을 `MSR_CSTAR`에 적습니다:

```C
wrmsrl(MSR_CSTAR, ignore_sysret);
```

이것은 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S)어셈블리 파일에서 정의되었으며 `-ENOSYS`오류 코드를 반환합니다:

```assembly
ENTRY(ignore_sysret)
	mov	$-ENOSYS, %eax
	sysret
END(ignore_sysret)
```

이제 우리는 `CONFIG_IA32_EMULATION`커널 구성 옵션이 활성화 됐을 때 이전 코드에서 수행한 `MSR_IA32_SYSENTER_CS`, `MSR_IA32_SYSENTER_ESP`, `MSR_IA32_SYSENTER_EIP`모델 특정 레지스터를 채워야 합니다. 이 경우(`CONFIG_IA32_EMULATION`구성 옵션이 설정되지 않은 때) 우리는 `MSR_IA32_SYSENTER_ESP`와 `MSR_IA32_SYSENTER_EIP`를 0으로 채우고 [글로벌 디스크립터 터이블](https://en.wikipedia.org/wiki/Global_Descriptor_Table)의 무효한 세그먼트를 `MSR_IA32_SYSENTER_CS`모델 특정 레지스터에 넣습니다:

```C
wrmsrl_safe(MSR_IA32_SYSENTER_CS, (u64)GDT_ENTRY_INVALID_SEG);
wrmsrl_safe(MSR_IA32_SYSENTER_ESP, 0ULL);
wrmsrl_safe(MSR_IA32_SYSENTER_EIP, 0ULL);
```

리눅스 커널의 부팅 프로세스를 설명하는 챕터의 두 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)에서 `Global Descriptor Table`에 관한 내용을 읽을 수 있습니다.

`syscall_init`함수의 끝에서 플래그의 설정을 `MSR_SYSCALL_MASK`모델 특정 레지스터에 작성 함으로써 [플래그 레지스터](https://en.wikipedia.org/wiki/FLAGS_register)내의 플래그를 마스크합니다:

```C
wrmsrl(MSR_SYSCALL_MASK,
	   X86_EFLAGS_TF|X86_EFLAGS_DF|X86_EFLAGS_IF|
	   X86_EFLAGS_IOPL|X86_EFLAGS_AC|X86_EFLAGS_NT);
```

이 플래그들은 syscall의 초기화 중에 지워집니다. 이것이 전부입니다. 이것은 `syscall_init`함수의 끝이며 시스템 호출 엔트리의 작업이 준비되었음을 의미합니다. 이제 사용자 응용 프로그램이 `syscall`명령을 실행할 때 어떤 일이 일어나는지 볼 수 있습니다.

시스템 호출 처리기가 호출되기 전 준비
--------------------------------------------------------------------------------

이미 쓴 것처럼 리눅스 커널에서 시스템 호출 또는 인터럽트 처리기를 호출하기 전에 몇 가지 준비 작업을 수행해야합니다. `idtentry`매크로는 예외 처리기가 실행되기 전에 요청된 준비를 수행하고 `interrupt`매크로는 인터럽트 처리기가 호출되기 전에 요청된 준비를 수행하며 `entry_SYSCALL_64`은 시스템 호출 처리기가 실행되기 전에 요청된 준비를 수행할 것입니다.

`entry_SYSCALL_64`은 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S)어셈블리 파일에서 정의되었으며 다음 매크로에서 시작합니다:

```assembly
SWAPGS_UNSAFE_STACK
```

이 매크로는 [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irqflags.h)헤더 파일에서 정의되었으며 `swapgs`명령으로 확장됩니다:

```C
#define SWAPGS_UNSAFE_STACK	swapgs
```

현재 GS 기본 레지스터의 값을 `MSR_KERNEL_GS_BASE `모델 특정 레지스터에 포함된 값과 교환합니다. 다시 말해 커널 스택으로 옮깁니다. 그 다음 이전 스택 포인터로 `rsp_scratch` [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) 변수를 가리키고 스택 포인터를 현재 프로세서를 위한 스택의 맨 위를 가리키도록 설정합니다:

```assembly
movq	%rsp, PER_CPU_VAR(rsp_scratch)
movq	PER_CPU_VAR(cpu_current_top_of_stack), %rsp
```

다음 단계에서 스택 세그먼트와 이전 스택 포인터를 스택에 넣습니다:

```assembly
pushq	$__USER_DS
pushq	PER_CPU_VAR(rsp_scratch)
```

그 다음 인터럽트가 엔트리에서 `off`이고 미구현 시스템 호출 및 스텍의 코드 세그먼트 레지스터를 위한 범용 [레지스터](https://en.wikipedia.org/wiki/Processor_register)(이외에도 `bp`, `bx`및 `r12`에서 `r15`로), 플래그, `-ENOSYS`를 저장하기 때문에 인터럽트를 활성화합니다:

```assembly
ENABLE_INTERRUPTS(CLBR_NONE)

pushq	%r11
pushq	$__USER_CS
pushq	%rcx
pushq	%rax
pushq	%rdi
pushq	%rsi
pushq	%rdx
pushq	%rcx
pushq	$-ENOSYS
pushq	%r8
pushq	%r9
pushq	%r10
pushq	%r11
sub	$(6*8), %rsp
```

사용자 응용프로그램에서 시스템 호출이 일어났을 때, 범용 레지스터의 상태는 다음과 같습니다:

* `rax` - 시스템 호출 번호 포함; 
* `rcx` - 사용자 공간의 반환 주소 포함;
* `r11` - 레지스터 플래그 포함;
* `rdi` - 시스템 호출 처리기의 첫 번째 인자 포함;
* `rsi` - 시스템 호출 처리기의 두 번째 인자 포함;
* `rdx` - 시스템 호출 처리기의 세 번째 인자 포함;
* `r10` - 시스템 호출 처리기의 네 번째 인자 포함;
* `r8`  - 시스템 호출 처리기의 다섯 번째 인자 포함;
* `r9`  - 시스템 호출 처리기의 여섯 번째 인자 포함;

다른 범용 레지스터(`rbp`, `rbx` 및 `r12`에서 `r15`)는 [C ABI](http://www.x86-64.org/documentation/abi.pdf)의 수신자를 보존합니다. 따라서 스택 상단에 레지스터 플래그를 넣은 다음 사용자 코드 세그먼트, 사용자 공간으로의 주소 반환, 시스템 호출 번호, 처음 세 인자, 미구현 시스템 호출을 위한 덤프 오류 코드 및 스텍의 다른 인자를 넣습니다.

다음 단계에서 현재 `thread_info`의 `_TIF_WORK_SYSCALL_ENTRY`를 확인합니다:

```assembly
testl	$_TIF_WORK_SYSCALL_ENTRY, ASM_THREAD_INFO(TI_flags, %rsp, SIZEOF_PTREGS)
jnz	tracesys
```

`_TIF_WORK_SYSCALL_ENTRY`매크로는 [arch/x86/include/asm/thread_info.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/thread_info.h)헤더 파일에서 정의되었으며 추적 시스템 호출에 관련된 스레드 정보 플래그의 집합을 제공합니다:

```C
#define _TIF_WORK_SYSCALL_ENTRY \
    (_TIF_SYSCALL_TRACE | _TIF_SYSCALL_EMU | _TIF_SYSCALL_AUDIT |   \
    _TIF_SECCOMP | _TIF_SINGLESTEP | _TIF_SYSCALL_TRACEPOINT |     \
    _TIF_NOHZ)
```

이 챕터에서는 디버깅/추적과 관련된 것을 고려하지 않지만, 별도의 챕터에서 리눅스 커널의 디버깅 및 추적 기술에 대한 내용을 살펴볼 것입니다. `tracesys`레이블 다음 레이블은 `entry_SYSCALL_64_fastpath`입니다. `entry_SYSCALL_64_fastpath`에서 우리는 [arch/x86/include/asm/unistd.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/unistd.h)에서 정의된 `__SYSCALL_MASK`를 확인할 것입니다:

```C
# ifdef CONFIG_X86_X32_ABI
#  define __SYSCALL_MASK (~(__X32_SYSCALL_BIT))
# else
#  define __SYSCALL_MASK (~0)
# endif
```

`__X32_SYSCALL_BIT`의 위치:

```C
#define __X32_SYSCALL_BIT	0x40000000
```

보시다시피 `__SYSCALL_MASK`는 `CONFIG_X86_X32_ABI`커널 구성 옵션에 따르며 64비트 커널에서 32비트[ABI](https://en.wikipedia.org/wiki/Application_binary_interface)를 위한 마스크를 나타냅니다.

따라서 우리는 `__SYSCALL_MASK`의 값을 확인하고 `CONFIG_X86_X32_ABI`가 비활성화되면 `rax`레지스터의 값을 최대 시스템 번호(`__NR_syscall_max`)와 비교하거나 `CONFIG_X86_X32_ABI`가 활성화됐으면 `eax`레지스터를 `__X32_SYSCALL_BIT`로 마스크하고 동일한 비교를 수행합니다:

```assembly
#if __SYSCALL_MASK == ~0
	cmpq	$__NR_syscall_max, %rax
#else
	andl	$__SYSCALL_MASK, %eax
	cmpl	$__NR_syscall_max, %eax
#endif
```

그 다음 `CF`와 `ZF`플래그가 0이면 수행하는 `ja`명령으로 마지막 비교의 결과를 검사합니다:

```assembly
ja	1f
```

이에 대한 올바른 시스템 호출이 있는 경우, 우리는 네 번째 인자를 `r10`에서 [x86_64 C ABI](http://www.x86-64.org/documentation/abi.pdf)의 준수를 유지하는 `rcx`로 옮기고 시스템 호출 처리기의 주소로 `call`명령을 실행합니다:

```assembly
movq	%r10, %rcx
call	*sys_call_table(, %rax, 8)
```

참고로, `sys_call_table`은 우리가 이 파트의 위에서 본 배열입니다. 이미 알고 있듯이 `rax`범용 레지스터는 시스템 호출의 번호를 포함하고 `sys_call_table`의 각 요소는 8바이트입니다. 따라서 우리는 `*sys_call_table(, %rax, 8)`표기를 사용해 `sys_call_table`배열에서 주어진 시스템 호출 처리기를 위한 올바른 오프셋을 찾습니다.

그것이 전부입니다. 우리는 요구되는 모든 준비를 했고 시스템 호출 처리기는 주어진 인터럽트 처리기(예를 들어 `sys_read`, `sys_write` 또는 리눅스 커널 코드에서 `SYSCALL_DEFINE[N]`매크로로 정의된 다른 시스템 호출 처리기)를 위해 호출했습니다.

시스템 호출 종료
--------------------------------------------------------------------------------

시스템 호출 처리기가 작업을 마치면, 시스템 호출 처리기를 호출한 직후 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S)로 돌아갑니다:

```assembly
call	*sys_call_table(, %rax, 8)
```

시스템 호출 처리기에서 리턴한 다음 단계는 시스템 처리기의 반환 값을 스택에 넣는 것입니다. 우리는 시스템 호출이 범용 `rax`레지스터에서 사용자 프로그램에 결과를 반환하는 것을 알고 있으므로, 시스템 호출 처리기가 작업을 마친 다음 그 값을 스택으로 옮깁니다:

```C
movq	%rax, RAX(%rsp)
```

`RAX`의 위치에.

그런 다음 [arch/x86/include/asm/irqflags.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/irqflags.h)에서 `LOCKDEP_SYS_EXIT`매크로의 호출을 볼 수 있습니다:

```assembly
LOCKDEP_SYS_EXIT
```

이 매크로의 구현은 시스템 호출 종료 시 디버그 잠금을 허용하는 `CONFIG_DEBUG_LOCK_ALLOC`커널 구성 옵션에 의존합니다. 다시 우리는 이 챕터에서 그것을 고려하지 않고, 별도의 것으로 돌아올 것입니다. `entry_SYSCALL_64`함수가 끝나면 `rcx`레지스터는 시스템 호출을 호출한 응용프로그램의 리턴 주소를 포함해야 하고 `r11`레지스터가 이전 [플래그 레지스터]를 포함하므로  `rcx` 및 `r11`외에도 모든 범용 레지스터를 복구합니다. 모든 범용 레지스터가 복구된 후 리턴 주소로 `rcx`를 , 플래그로 `r11`을 그리고 이전 스택 포인터로 `rsp`를 채웁니다:

```assembly
RESTORE_C_REGS_EXCEPT_RCX_R11

movq	RIP(%rsp), %rcx
movq	EFLAGS(%rsp), %r11
movq	RSP(%rsp), %rsp

USERGS_SYSRET64
```

결국 사용자 `GS`와 커널 `GS`를 다시 교환하는 `swapgs`명령 및 시스템 호출 처리기의 종료를 실행하는 `sysretq`명령의 호출로 확장된 `USERGS_SYSRET64`매크로를 호출합니다: 

```C
#define USERGS_SYSRET64				\
	swapgs;	           				\
	sysretq;
```

이제 우리는 사용자 응용프로그램이 시스템 호출을 호출할 때 무엇이 일어나는지 압니다. 이 프로세스의 전체 경로는 다음과 같습니다:

* 사용자 응용프로그램은 범용 레지스터를 값(시스템 호출 번호 및 시스템 호출의 인자)으로 채우는 코드를 포함합니다;
* 프로세서가 사용자 모드에서 커널 모드로 전환하고 시스템 호출 엔트리 `entry_SYSCALL_64`의 수행을 시작합니다;
* `entry_SYSCALL_64`는 커널 스택으로 전환하고 몇 개의 범용 레지스터, 이전 스택 및 코드 세그먼트, 플래그 등을 스택에 저장합니다;
* `entry_SYSCALL_64`는 `rax`레지스터에서 시스템 호출 번호를 확인하고, 시스템 호출의 번호가 올바르면 `sys_call_table`에서 시스템 호출 처리기를 찾아 호출합니다;
* 시스템 호출이 올바르지 않은 경우, 시스템 호출 종료로 점프합니다;
* 시스템 호출 처리기는 작업을 완료한 후, 범용 레지스터, 이전 스택, 플래그 및 리턴주소를 복구하고 `entry_SYSCALL_64`에서 `sysretq`명령을 사용해 종료합니다.

그것이 전부입니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널에서 시스템 호출 개념에 대한 두 번째 파트의 끝입니다. 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html)에서 우리는 사용자 응용프로그램 뷰에서 이 개념에 대한 이론을 봤습니다. 이 파트에서 우리는 시스템 호출 개념과 관련된 것들을 계속해서 살펴봤고 시스템 호출이 일어났을 때 리눅스 커널이 무엇을 하는지 봤습니다.

질문이나 제안 사항이 있다면, 트위터[0xAX](https://twitter.com/0xAX)로 자유롭게 보내거나 제 [이메일](anotherworldofworld@gmail.com)에 넣거나 [이슈](https://github.com/0xAX/linux-insides/issues/new)를 만들어 주세요. 

**영어는 모국어가 아니어서 모든 불편한 점은 정말 죄송합니다. 실수를 발견하면 저에게 [linux-insides](https://github.com/0xAX/linux-insides)로 PR을 보내주십시오.**

링크
--------------------------------------------------------------------------------

* [시스템 호출](https://en.wikipedia.org/wiki/System_call)
* [write](http://man7.org/linux/man-pages/man2/write.2.html)
* [C 표준 라이브러리](https://en.wikipedia.org/wiki/GNU_C_Library)
* [cpu 아키텍처의 리스트](https://en.wikipedia.org/wiki/List_of_CPU_architectures)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [kbuild](https://www.kernel.org/doc/Documentation/kbuild/makefiles.txt)
* [typedef](https://en.wikipedia.org/wiki/Typedef)
* [errno](http://man7.org/linux/man-pages/man3/errno.3.html)
* [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [모델 특정 레지스터](https://en.wikipedia.org/wiki/Model-specific_register)
* [인텔 2b 메뉴얼](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [보조프로세서](https://en.wikipedia.org/wiki/Coprocessor)
* [지시 포인터](https://en.wikipedia.org/wiki/Program_counter)
* [플래그 레지스터](https://en.wikipedia.org/wiki/FLAGS_register)
* [글로벌 디스크립터 테이블](https://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [범용 레지스터](https://en.wikipedia.org/wiki/Processor_register)
* [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)
* [x86_64 C ABI](http://www.x86-64.org/documentation/abi.pdf)
* [이전 챕터](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-1.html)
