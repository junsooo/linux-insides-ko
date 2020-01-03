커널 초기화. Part 5.
================================================================================

아키텍처 별 초기화 이어서
================================================================================

[이전 부분](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-4)에서 [setup_arch](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L856) 함수의 아키텍처 별 항목의 초기화를 중단했으며, 이어서 계속 진행하겠습니다. [initrd](http://en.wikipedia.org/wiki/Initrd)에 대한 메모리를 예약 했으므로 다음 단계는 `olpc_ofw_detect`로 [One Laptop Per Child support](http://wiki.laptop.org/go/OFW_FAQ)를 감지하는 것입니다. 이 책 안의 항목과 관련된 플랫폼을 고려하지 않을 것이며, 그와 관련 된 함수들은 생략하겠습니다. 다음 단계는 `early_trap_init` 함수입니다. 이 함수는 디버그 (`# DB`-rflags의 `TF` 플래그가 설정되면 발생)를 초기화하고 `int3`(`# BP`)은 게이트를 중단합니다. 인터럽트에 대해 모른다면 [초기 인터럽트와 예외 처리](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-2)에서 읽을 수 있습니다. `x86` 아키텍처 `INT`에서 `INTO` 및 `INT3`은 작업이 인터럽트 핸들러를 명시적으로 호출 할 수있게하는 특수 명령입니다. `INT3` 명령어는 브레이크포인트인 (`# BP`) 핸들러를 호출합니다. 우리는 이미 인터럽트와 예외에 대해 [이 부분](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-2) 에서 보았습니다. :

```
----------------------------------------------------------------------------------------------
|Vector|Mnemonic|Description         |Type |Error Code|Source                   |
----------------------------------------------------------------------------------------------
|3     | #BP    |Breakpoint          |Trap |NO        |INT 3                    |
----------------------------------------------------------------------------------------------
```

디버그 인터럽트 `# DB` 는 디버거를 호출하는 기본 방법입니다. `early_trap_init`는 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)에 정의되어 있습니다. 이 함수는 `# DB` 및 `# BP` 핸들러를 설정하고 [IDT](https://ko.wikipedia.org/wiki/인터럽트_서술자_테이블)를 다시로드합니다. :

```C
void __init early_trap_init(void)
{
        set_intr_gate_ist(X86_TRAP_DB, &debug, DEBUG_STACK);
        set_system_intr_gate_ist(X86_TRAP_BP, &int3, DEBUG_STACK);
        load_idt(&idt_descr);
}
```

우리는 이미 이전 부분에서 인터럽트에 대해 `set_intr_gate`의 구현을 보았습니다. 다음으로 볼 것은 두 개의 유사한 함수인 `set_intr_gate_ist`와 `set_system_intr_gate_ist`입니다. 이 두 기능은 모두 세 가지 매개 변수를 갖습니다. :

* 인터럽트 번호;
* 인터럽트/예외 핸들러의 기준 주소;
* 세번째 매개 변수 - `인터럽트 스택 테이블`. `IST`는`x86_64`의 새로운 메커니즘이며 [TSS](http://en.wikipedia.org/wiki/Task_state_segment)의 일부입니다. 커널 모드의 모든 활성 스레드에는 16KB의 커널 스택이 있습니다. 사용자 공간에 스레드가 있는 동안 이 커널 스택은 비어 있습니다.

스레드 별 스택 외에도 각 CPU와 관련된 두 개의 특수한 스택이 있습니다. 이 스택에 관한 모든 것은 리눅스 커널 문서-[Kernel stacks](https://www.kernel.org/doc/Documentation/x86/kernel-stacks)에서 읽을 수 있습니다. `x86_64`는 마스크할 수 없는 인터럽트 등과 같은 어떠한 사건이 진행되는 동안 새로운 `특수` 스택으로 전환 할 수있는 기능을 제공합니다. 이 기능의 이름은 '인터럽트 스택 테이블'입니다. CPU 당 최대 7 개의 IST 항목이있을 수 있으며 모든 항목은 전용 스택을 가리킵니다. 우리의 경우 이것은 `DEBUG_STACK`입니다.

`set_intr_gate_ist`와`set_system_intr_gate_ist`는 단 하나의 차이점만 제외하고 `set_intr_gate`와 동일한 원리로 작동합니다. 이 두 함수 모두 `set_intr_gate`처럼 인터럽트 번호를 체크하고 내부에서 `_set_gate`를 호출합니다. :

```C
BUG_ON((unsigned)n > 0xFF);
_set_gate(n, GATE_INTERRUPT, addr, 0, ist, __KERNEL_CS);
```
___

`set_intr_gate`는 [dpl](http://en.wikipedia.org/wiki/Privilege_level) - 0 와 ist - 0 으로 `_set_gate`를 호출하지만, `set_intr_gate_ist`와 `set_system_intr_gate_ist`는 `ist`를 `DEBUG_STACK`으로 설정하고, `dpl`을 가장 낮은 권한인 `0x3`으로 설정합니다. 인터럽트가 발생하고 하드웨어가 이러한 디스크립터를 로드하면 하드웨어는 IST 값을 기반으로 새 스택 포인터를 자동으로 설정 한 다음 인터럽트 핸들러를 호출합니다. 모든 특수 커널 스택은 `cpu_init` 함수에서 설정됩니다 (나중에 살펴보겠습니다).

`idt_descr`에 쓰여진 `# DB`와 `# BP`게이트에 따라, `ldtr` 명령을 계산하는 `load_idt`를 이용하여 `IDT` 테이블을 재로드합니다. 이제 인터럽트 처리기를 살펴보고 작동 방식을 이해하려고합니다. 물론, 저는 이 책에서 모든 인터럽트 처리기를 다룰 수는 없으며 이것의 요점을 살펴보지는 않을겁니다. 리눅스 커널 소스 코드를 살펴 보는 것은 매우 흥미 롭기 때문에 여러분은 이제 `debug` 핸들러가 어떻게 구현되고 다른 인터럽트 핸들러는 어떻게 구현되는지 이해하게 될 것입니다.

#DB 핸들러
--------------------------------------------------------------------------------

위에서 읽을 수 있듯이, `set_intr_gate_ist`에서 `# DB` 핸들러의 주소를 `&debug`로 전달했습니다. [lxr.free-electrons.com](http://lxr.free-electrons.com/ident)은 리눅스 커널 소스 코드에서 식별자를 검색하기위한 훌륭한 자료이지만 불행히도 `debug` 핸들러는 찾지 못할 것입니다. 여러분이 알아낼 수 있는 것은 모두 [arch/x86/include/asm/traps.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/traps.h)에 있는 `debug` 정의입니다. :

```C
asmlinkage void debug(void);
```

`asmlinkage` 속성을 보면 `debug`가 [어셈블리](https://ko.wikipedia.org/wiki/어셈블리어)로 작성된 함수라는 것을 알 수 있습니다. 그렇습니다, 또 다시 어셈블리입니다 :). 다른 핸들러로서의 `# DB` 핸들러 구현은 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S)에서 볼 수 있으며, `idtentry` 어셈블리 매크로로 정의됩니다 :

```assembly
idtentry debug do_debug has_error_code=0 paranoid=1 shift_ist=DEBUG_STACK
```

`idtentry` 는 인터럽트/예외 엔트리 포인트 매크로입니다. 보시다시피 이것은 다섯개의 인수를 가집니다. :

* 인터럽트 엔트리 포인트의 이름;
* 인터럽트 핸들러의 이름;
* 인터럽트 에러 코드의 유무;
* paranoid  - 이 매개 변수 = 1 인 경우 특수 스택으로 전환 (위에서 다룬 내용);
* shift_ist - 인터럽트동안 전환 할 스택.


이제 'idtentry' 매크로 구현을 살펴 봅시다. 이 매크로는 동일한 어셈블리 파일에 정의되어 있으며 `ENTRY` 매크로로 `debug` 기능을 정의합니다. 처음으로 특수 스택으로 전환해야하는 경우에 `idtentry` 매크로는 주어진 매개 변수가 올바른지 확인합니다. 다음 단계에서는 인터럽트가 오류 코드를 반환하는지 확인합니다. 만약 인터럽트가 에러 코드를 리턴하지 않으면 (우리의 경우 `# DB`는 에러 코드를 리턴하지 않습니다.) `INTR_FRAME`를 호출하고, 인터럽트에 에러 코드가 있으면 `XCPT_FRAME`을 호출합니다. 이러한 매크로 'XCPT_FRAME'및 'INTR_FRAME'은 인터럽트를 위한 초기 프레임 상태를 구축하는 데에만 필요하며 다른 것은 아무 것도 수행하지 않습니다. 이들은 CFI 지시어를 사용하며, 디버깅에 사용됩니다. 더 많은 정보는 [CFI 지시문](https://sourceware.org/binutils/docs/as/CFI-directives.html)에서 찾을 수 있습니다. [arch/x86/kernel/entry_64.S]의 코멘트에 따르면 : `CFI 매크로는 더 나은 역 추적을 위해 dwarf2 unwind 정보를 생성하는 데 사용됩니다. 그들은 어떤 코드도 변경하지 않습니다.` 이에 따라 무시하겠습니다.

```assembly
.macro idtentry sym do_sym has_error_code:req paranoid=0 shift_ist=-1
ENTRY(\sym)
	/* Sanity check */
	.if \shift_ist != -1 && \paranoid == 0
	.error "using shift_ist requires paranoid=1"
	.endif

	.if \has_error_code
	XCPT_FRAME
	.else
	INTR_FRAME
	.endif
	...
	...
	...
```

초기 인터럽트/예외 다루기를 다뤘던 이전 파트에서 인터럽트 발생 후에 현재 스택은 다음 형식과 같다고 했던 것을 기억할 것입니다. :

```
    +-----------------------+
    |                       |
+40 |         SS            |
+32 |         RSP           |
+24 |        RFLAGS         |
+16 |         CS            |
+8  |         RIP           |
 0  |       Error Code      | <---- rsp
    |                       |
    +-----------------------+
```

`idtentry` 구현에서 온 다음 두 매크로를 살펴봅시다:

```assembly
	ASM_CLAC
	PARAVIRT_ADJUST_EXCEPTION_FRAME
```

첫 번째 `ASM_CLAC` 매크로는 `CONFIG_X86_SMAP` 구성 옵션에 의존하며 보안상의 이유가 필요합니다. 자세한 내용은 [여기](https://lwn.net/Articles/517475/)를 참조하세요. 두 번째 `PARAVIRT_ADJUST_EXCEPTION_FRAME` 매크로는 Xen 타입의 예외를 처리하기위한 것입니다(이 장은 커널 초기화에 대한 내용이므로 가상화에 대해서는 다루지 않습니다).

다음 코드는 인터럽트에 오류 코드가 있는지 확인하고, 없다면 스택의 `x86_64`에서 `0xffffffffffffffff` 인 `$-1`을 푸시합니다. :

```assembly
	.ifeq \has_error_code
	pushq_cfi $-1
	.endif
```

우리는 모든 인터럽트에 대한 스택 일관성을 위해 `모조` 에러 코드로 수행할 필요가 있습니다. 다음 단계에서 스택 포인터 `$ORIG_RAX-R15`:

```assembly
	subq $ORIG_RAX-R15, %rsp
```

에서 빼면, `ORIRG_RAX`와 `R15`, [arch/x86/include/asm/calling.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/calling.h)와 `ORIG_RAX-R15`에 정의된 다른 매크로들은 120 바이트입니다. 인터럽트 처리 중에 스택에 모든 레지스터를 저장해야하기 때문에 범용 레지스터는 120 바이트를 차지하는 것입니다. 범용 레지스터에 스택을 설정 한 후 다음 단계에서 인터럽트가 사용자 공간에서 발생했는지 확인합니다. :

```assembly
testl $3, CS(%rsp)
jnz 1f
```

여기서 우리는`CS`에서 첫 번째와 두 번째 비트를 확인합니다. `CS` 레지스터에는 처음 두 비트가 `RPL`인 세그먼트 선택기가 포함되어 있음을 기억할 것 입니다. 모든 권한 수준은 0–3 범위의 정수이며, 가장 낮은 숫자가 가장 높은 권한에 해당합니다. 따라서 커널 모드에서 인터럽트가 발생하면 `save_paranoid`를 호출하고, 그렇지 않은 경우 레이블 `1`로 이동합니다. `save_paranoid`에서 모든 범용 레지스터를 스택에 저장하고 필요한 경우 커널 `gs`에서 사용자 `gs`를 전환합니다. :

```assembly
	movl $1,%ebx
	movl $MSR_GS_BASE,%ecx
	rdmsr
	testl %edx,%edx
	js 1f
	SWAPGS
	xorl %ebx,%ebx
1:	ret
```

다음 단계에서 우리는 `pt_regs` 포인터를 `rdi`에 놓고, 만약 인터럽트 핸들러(우리의 경우 [arch/x86/kernel/traps.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/traps.c)에서 나오는 `do_debug`)를 가지고 있고, 호출한다면 `rsi` 에 에러코드를 저장합니다. 다른 핸들러들과 같이 `do_debug`는 두 매개변수를 가지고 있습니다. :

* pt_regs - 프로세스 메모리 영역에 저장되는 일련의 CPU 레지스터들을 보여주는 구조;
* error code - 인터럽트의 에러 코드.

인터럽트 핸들러가 작업을 끝낸 후, 스택을 재저장하는 `paranoid_exit`를 호출하고, 만약 인터럽트가 이곳에서 왔다면 유저환경을 활성화하고, `iret`를 호출합니다. 끝났습니다. 물론 모두는 아니고, 우리는 인터럽트에 대해 나눠진 다른 챕터에서 더 깊게 살펴볼 것입니다.

이것은 `# DB` 인터럽트에 대한 `idtentry` 매크로의 일반적인 모습입니다. 모든 인터럽트는 이 구현과 유사하며 idtentry로도 정의됩니다. `early_trap_init`가 작업을 마치면 다음 함수는 `early_cpu_init`입니다. 이 함수는 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c)에 정의되어 있으며 CPU 및 공급 업체에 대한 정보를 수집합니다.

초기 iorermap 초기화
--------------------------------------------------------------------------------

다음 단계에서는 초기의 `ioremap` 초기화에 대해 다룹니다. 일반적으로 장치들과 소통하는 데에는 두가지 방법이 있습니다. :

* I/O Ports;
* Device memory.

우리는 이미 첫번째 방법 (`outb/inb` 명령어들)을 리눅스 커널 부팅 [과정](Booting/linux-bootstrap-3.md) 파트에서 보았습니다. 두 번째 방법은 I/O 물리적 주소를 가상 주소에 매핑하는 것입니다. 물리적 주소가 CPU에 의해 액세스 될 때, 이는 I/O 장치의 메모리에 매핑 될 수있는 물리적 RAM의 일부를 가리킬 수 있습니다. 따라서 `ioremap`은 장치 메모리를 커널 주소 공간에 매핑하는 데 사용됩니다.

위에서 작성한 것처럼 다음 함수는 I/O 메모리를 커널 주소 공간에 다시 매핑하여 액세스 할 수있는 `early_ioremap_init` 입니다.  `ioremap`과 같은 일반 매핑 함수를 사용하려면, I/O 또는 메모리 영역을 임시로 매핑할 필요가 있는 초기 초기화 코드를 위해 초기 ioremap을 초기화해야합니다. 이 함수의 구현은 [arch/x86/mm/ioremap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/ioremap.c)에 있습니다. `early_ioremap_init`의 시작 부분에서 우리는 `pmd_t`타입의 `pmd` 포인트의 정의를 볼 수 있으며(이것은 페이지 중간 디렉토리 엔트리 `typedef struct {pmdval_t pmd;} pmd_t;`를 보여줍니다. 여기서 `pmdval_t`는 `unsigned long`타입입니다.), `fixmap`이 올바르게 정렬되었는지 확인할 수 있습니다. :

```C
pmd_t *pmd;
BUILD_BUG_ON((fix_to_virt(0) + PAGE_SIZE) & ((1 << PMD_SHIFT) - 1));
```

`fixmap`-`FIXADDR_START`에서부터 `FIXADDR_TOP`까지 확장하는 고정 가상 주소 매핑입니다. 고정 가상 주소는 컴파일 타임에 가상 주소를 알아야하는 서브 시스템에 필요합니다. `early_ioremap_init` 점검 후 [mm/early_ioremap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/early_ioremap.c)에서 `early_ioremap_setup` 함수를 호출합니다. `early_ioremap_setup`은 `unsigned long` 타입의 `slot_virt` 배열을 512 개의 임시 부팅 시간 고정 매핑한 가상 주소로 채웁니다. ** :


```C
for (i = 0; i < FIX_BTMAPS_SLOTS; i++)
    slot_virt[i] = __fix_to_virt(FIX_BTMAP_BEGIN - NR_FIX_BTMAPS*i);
```

그런 다음 `FIX_BTMAP_BEGIN`에 대한 페이지 중간 디렉토리 엔트리를 가져 와서 `pmd` 변수에 넣고, 부팅 타임 페이지 테이블인 0들로 `bm_pte`를 채운 뒤, 주어진 페이지 중간 디렉토리에서 주어진 페이지 테이블 엔트리를 설정하기 위해 `pmd_populate_kernel` 함수를 호출합니다. :

```C
pmd = early_ioremap_pmd(fix_to_virt(FIX_BTMAP_BEGIN));
memset(bm_pte, 0, sizeof(bm_pte));
pmd_populate_kernel(&init_mm, pmd, bm_pte);
```

위 내용은 이것으로 끝났습니다. 만약 아직 혼란스럽더라도 걱정하지 마세요. `ioremap` 과 `fixmaps` 에 대해 다루는 특별한 파트가 [리눅스 커널 메모리 관리. Part 2](https://junsoolee.gitbook.io/linux-insides-ko/summary/mm/linux-mm-2) 챕터에 있습니다.

루트 장치의 주 번호와 부 번호 얻기
--------------------------------------------------------------------------------

초기 `ioremap` 초기화 후에, 다음 코드를 볼 수 있습니다. :

```C
ROOT_DEV = old_decode_dev(boot_params.hdr.root_dev);
```
이 코드는 `do_mount_root` 함수 안에서 `initrd`가 나중에 마운트 될 루트 장치의 주 번호와 부 번호를 얻을 수 있습니다. 장치의 주 번호는 장치와 관련된 드라이버를 식별합니다. 부 번호는 드라이버에 의해 컨트롤 되는 장치를 가리킵니다. `old_decode_dev` 가 `boot_params_structure`로 부터 한 매개변수를 가져온다는 것을 알아두세요. x86 리눅스 커널 부팅 프로토콜에서 읽을 수 있습니다. :


```
Field name:	root_dev
Type:		modify (optional)
Offset/size:	0x1fc/2
Protocol:	ALL

  The default root device device number.  The use of this field is
  deprecated, use the "root=" option on the command line instead.
```

이제 `old_decode_dev`가 하는 것을 이해하려 해봅시다. 실제로 그것은 단지  `dev_t`를 발생시키는 곳 안에서 주어진 주 번호와 부 번호들로부터 `MKDEV`를 호출합니다. 구현은 매우 간단합니다. :

```C
static inline dev_t old_decode_dev(u16 val)
{
         return MKDEV((val >> 8) & 255, val & 255);
}
```

여기서 `dev_t`는 주/부 번호 쌍을 나타내는 커널 데이터 타입입니다.  하지만 이상한 접두사 `old_` 는 무엇일까요? 역사적인 이유로, 장치의 주/부 번호를 관리하는 두 가지 방법이 있습니다. 첫 번째 방법에서 주/부 번호는 합쳐서 2 바이트를 차지했습니다. 이전 코드에서 볼 수 있듯이, 주 번호가 8 비트, 부 번호가 8 비트를 차지합니다. 그러나 이 방법에는 문제가 있었습니다.  256 개의 주 번호와 256 개의 부 번호만 가능하다는 것 입니다. 따라서 16 비트 정수는 32 비트 정수로 대체되었으며, 12 비트는 주 번호로, 20 비트는 부 번호로 예약되었습니다. `new_decode_dev` 구현에서 이것을 볼 수 있습니다 :

```C
static inline dev_t new_decode_dev(u32 dev)
{
         unsigned major = (dev & 0xfff00) >> 8;
         unsigned minor = (dev & 0xff) | ((dev >> 12) & 0xfff00);
         return MKDEV(major, minor);
}
```

계산 후에 우리는 그 결과가 `0xffffffff` 인 경우에는 `0xfff`를, `0xfffff`인 경우에는 `주`번호에 대한 12비트를, 그리고 `부` 번호에 대한 20비트를 얻을 수 있습니다. 따라서 `old_decode_dev`의 실행이 끝날 때 `ROOT_DEV`에서 루트 디바이스의 주 번호와 부 번호를 얻을 수 있습니다.


메모리 맵 설정
--------------------------------------------------------------------------------

다음 다룰 것은 `setup_memory_map` 함수의 호출을 통해 메모리 맵을 설정하는 것입니다. 하지만 이것 전에 우리는 스크린 (현재 행과 열, 비디오 페이지 등(이것에 대해 [비디오 모드의 초기화와 보호 모드로 전환하기](https://junsoolee.gitbook.io/linux-insides-ko/summary/booting/linux-bootstrap-3)에서 살펴볼 수 있습니다.)), 확장된 디스플레이 초기화 데이터, 비디오 모드, 부트로더 타입, 등 에 대한 정보를 위해 다른 매개변수들을 설정합니다. :

```C
	screen_info = boot_params.screen_info;
	edid_info = boot_params.edid_info;
	saved_video_mode = boot_params.hdr.vid_mode;
	bootloader_type = boot_params.hdr.type_of_loader;
	if ((bootloader_type >> 4) == 0xe) {
		bootloader_type &= 0xf;
		bootloader_type |= (boot_params.hdr.ext_loader_type+0x10) << 4;
	}
	bootloader_version  = bootloader_type & 0xf;
	bootloader_version |= boot_params.hdr.ext_loader_ver << 4;
```

부팅 시간 동안 얻은 모든 매개 변수는 `boot_params` 구조체에 저장되었습니다. 그런 다음 I/O 메모리의 마지막을 설정해야합니다. 커널의 주요 목적 중 하나는 리소스 관리입니다. 그리고 그 리소스 중 하나는 메모리입니다. 이미 알고 있듯이 장치와 통신하는 두 가지 방법이 I/O 포트와 장치 메모리입니다. 등록 된 리소스에 대한 모든 정보는 다음을 통해 제공됩니다.

* /proc/ioports - 장치와의 입력 또는 출력 통신에 사용되는 현재 등록 된 포트 영역 목록을 제공합니다.
* /proc/iomem - 각 물리적 장치에 대한 시스템 메모리의 현재 맵을 제공합니다.

현재 우리의 관심은 `/proc/iomem`에 있습니다. :

```
cat /proc/iomem
00000000-00000fff : reserved
00001000-0009d7ff : System RAM
0009d800-0009ffff : reserved
000a0000-000bffff : PCI Bus 0000:00
000c0000-000cffff : Video ROM
000d0000-000d3fff : PCI Bus 0000:00
000d4000-000d7fff : PCI Bus 0000:00
000d8000-000dbfff : PCI Bus 0000:00
000dc000-000dffff : PCI Bus 0000:00
000e0000-000fffff : reserved
  000e0000-000e3fff : PCI Bus 0000:00
  000e4000-000e7fff : PCI Bus 0000:00
  000f0000-000fffff : System ROM
```

보시다시피 주소 범위는 소유자와 16 진수 표기법으로 표시됩니다. Linux 커널은 일반적인 방법으로 모든 리소스를 관리하기위한 API를 제공합니다. 전역 리소스 (예 : PIC 또는 I/O 포트)는 하드웨어 버스 슬롯과 관련된 서브셋들로 나눌 수 있습니다. 주요 구조체 `resource`:

```C
struct resource {
        resource_size_t start;
        resource_size_t end;
        const char *name;
        unsigned long flags;
        struct resource *parent, *sibling, *child;
};
```

는 시스템 리소스의 트리와 유사한 서브셋에 대한 추상화를 나타냅니다. 이 구조체는 `start`부터 `end`까지의 리소스가 커버하는 주소 범위, (`resource_size_t`는 `phys_addr_t`, 또는 `x86_64`의 경우에는 u64`). 리소스의 `name`(`/proc/iomem` 출력에서 이 이름들을 볼 수 있습니다.) 및 리소스의 플래그 ([include/linux/ioport.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/ioport.h)에 정의 된 모든 리소스 플래그)를 제공합니다. 마지막은 '리소스' 구조체에 대한 세 가지 포인터입니다. 이 포인터는 트리와 유사한 구조체를 가능하게합니다.

```
+-------------+      +-------------+
|             |      |             |
|    parent   |------|    sibling  |
|             |      |             |
+-------------+      +-------------+
       |
       |
+-------------+
|             |
|    child    |
|             |
+-------------+
```

모든 리소스의 서브셋에는 루트 범위 리소스가 있습니다. `iomem`의 경우, `iomem_resource`는 다음과 같이 정의됩니다. :

```C
struct resource iomem_resource = {
        .name   = "PCI mem",
        .start  = 0,
        .end    = -1,
        .flags  = IORESOURCE_MEM,
};
EXPORT_SYMBOL(iomem_resource);
```

TODO EXPORT_SYMBOL

`iomem_resource`는 `PCI mem` 이름과 `IORESOURCE_MEM`(`0x00000200`)을 플래그로하여 io 메모리의 루트 주소 범위를 정의합니다. 위에서 쓴 것처럼 현재 요점은 `iomem`의 끝 주소의 설정입니다. 우리는 다음을 이용하여 설정할 것입니다. :


```C
iomem_resource.end = (1ULL << boot_cpu_data.x86_phys_bits) - 1;
```

여기서 우리는 `boot_cpu_data.x86_phys_bits`를 `1` 쉬프트 합니다. `boot_cpu_data`는 `early_cpu_init`을 실행 하면서 채운 cpuinfo_x86 구조체입니다. 'x86_phys_bits'는 필드의 이름에서 알 수 있듯이 시스템의 최대 물리적 주소의 최대 비트 량을 나타냅니다. `iomem_resource`는 `EXPORT_SYMBOL` 매크로로 전달됩니다. 이 매크로는 주어진 심볼 (이 경우 `iomem_resource`)을 동적 링크를 위해 내보내거나, 다른 말로하면 심볼이 동적으로 로드 된 모듈에 액세스 할 수있게합니다.

루트 `iomem` 리소스 주소 범위의 끝 주소를 설정 마친 후, 위에서 작성한 것처럼 다음 단계는 메모리 맵을 설정하는 것입니다. 이는 `setup_ memory_map` 함수를 호출하여 생성됩니다 :


```C
void __init setup_memory_map(void)
{
        char *who;

        who = x86_init.resources.memory_setup();
        memcpy(&e820_saved, &e820, sizeof(struct e820map));
        printk(KERN_INFO "e820: BIOS-provided physical RAM map:\n");
        e820_print_map(who);
}
```


우선 여기에서 `x86_init.resources.memory_setup`의 호출을 살펴 봅시다. `x86_init`는 플랫폼 초기화 기능을 리소스 초기화, pci 초기화 등으로 나타내는 `x86_init_ops` 구조체입니다.`x86_init`의 초기화는 [arch/x86/kernel/x86_init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/x86_init.c)에 있습니다. 여기서는 매우 길기 때문에 전체 설명을 하지는 않고, 현재 우리에게 관심이 있는 부분만 살펴보겠습니다. :


```C
struct x86_init_ops x86_init __initdata = {
	.resources = {
            .probe_roms             = probe_roms,
            .reserve_resources      = reserve_standard_io_resources,
            .memory_setup           = default_machine_specific_memory_setup,
    },
    ...
    ...
    ...
}
```

여기서 볼 수 있듯이 `memry_setup` 필드는 [부트 시간](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)에 수집 한 [e820](http://en.wikipedia.org/wiki/E820) 엔트리의 수를 얻는 `default_machine_specific_memory_setup`이며, BIOS e820 맵을 삭제하고 `e820map` 구조체를 메모리 영역으로 채웁니다. 모든 영역이 수집되면 printk를 사용하여 모든 영역을 출력합니다. `dmesg` 명령을 실행하면 다음과 같은 것을 볼 수 있습니다. :

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009d7ff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009d800-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000e0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x00000000be825fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000be826000-0x00000000be82cfff] ACPI NVS
[    0.000000] BIOS-e820: [mem 0x00000000be82d000-0x00000000bf744fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000bf745000-0x00000000bfff4fff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000bfff5000-0x00000000dc041fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000dc042000-0x00000000dc0d2fff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000dc0d3000-0x00000000dc138fff] usable
[    0.000000] BIOS-e820: [mem 0x00000000dc139000-0x00000000dc27dfff] ACPI NVS
[    0.000000] BIOS-e820: [mem 0x00000000dc27e000-0x00000000deffefff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000defff000-0x00000000deffffff] usable
...
...
...
```


BIOS Enhanced Disk Device 정보 복사
--------------------------------------------------------------------------------


다음 다룰 두 단계는 `parse_setup_data` 함수를 이용하여 `setup_data`를 분석하고, BIOS EDD를 안전한 곳에 복사하는 것입니다. `setup_data`는 커널 부트 헤더의 필드이며 `x86` 부트 프로토콜에서 읽을 수 있습니다 :

```
Field name:	setup_data
Type:		write (special)
Offset/size:	0x250/8
Protocol:	2.09+

  The 64-bit physical pointer to NULL terminated single linked list of
  struct setup_data. This is used to define a more extensible boot
  parameters passing mechanism.
```


이는 장치 트리 블롭, EFI 설정 데이터 등 다양한 유형의 설정 정보를 저장하는 데 사용됩니다. 두 번째 단계에서는 BIOS EDD 정보를 [arch/x86/boot/edd.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/edd.c) 에서 수집 한 `boot_params`구조체에서 `edd`구조체로 복사합니다. :

```C
static inline void __init copy_edd(void)
{
     memcpy(edd.mbr_signature, boot_params.edd_mbr_sig_buffer,
            sizeof(edd.mbr_signature));
     memcpy(edd.edd_info, boot_params.eddbuf, sizeof(edd.edd_info));
     edd.mbr_signature_nr = boot_params.edd_mbr_sig_buf_entries;
     edd.edd_info_nr = boot_params.eddbuf_entries;
}
```

메모리 디스크립터 초기화
--------------------------------------------------------------------------------

다음 단계는 init 프로세스의 메모리 디스크립터를 초기화하는 것입니다. 이미 알고 있듯이 모든 프로세스에는 자체 주소 공간이 있습니다. 이 주소 공간에는 '메모리 디스크립터 (memory descriptor)'라는 특수한 데이터 구조가 있습니다. 리눅스 커널 소스 코드 메모리 디스크립터 안 에서 직접적으로 `mm_struct` 구조체를 보여줍니다. `mm_struct`는 커널 코드/데이터의 시작/끝 주소, brk의 시작/끝, 메모리 영역 수, 메모리 영역 목록 등 프로세스 주소 공간과 관련된 여러 가지 필드를 포함합니다. 이 구조체는 [include/linux/mm_types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/mm_types.h)에 정의되어 있습니다. 모든 프로세스에는 자체 메모리 디스크립터가 있는데, `task_struct` 구조체에서는 `mm` 과 `active_mm` 필드에 포함되어있습니다. 그리고 우리의 첫 번째 `init` 프로세스도 마찬가지입니다. 이전 [부분](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-4)에서 INIT_TASK` 매크로를 이용한 init인 `task_struct`의 초기화부분을 보았습니다. :


```C
#define INIT_TASK(tsk)  \
{
    ...
	...
	...
	.mm = NULL,         \
    .active_mm  = &init_mm, \
	...
}
```

`mm`은 프로세스 주소 공간을 가리키고, `active_mm`은 프로세스에 커널 스레드와 같은 주소 공간이 없는 경우에 활성 주소 공간을 가리킵니다 (자세한 내용은 [문서](https://www.kernel.org/doc/Documentation/vm/active_mm.txt)를 참조하세요). 이제 초기 프로세스의 메모리 디스크립터를 커널의 텍스트, 데이터, brk로 채웁니다. :

```C
	init_mm.start_code = (unsigned long) _text;
	init_mm.end_code = (unsigned long) _etext;
	init_mm.end_data = (unsigned long) _edata;
	init_mm.brk = _brk_end;
```

`init_mm` 는 초기 프로세스의 메모리 디스크립터 이며, 다음과 같이 정의되어 있습니다. :

```C
struct mm_struct init_mm = {
    .mm_rb          = RB_ROOT,
    .pgd            = swapper_pg_dir,
    .mm_users       = ATOMIC_INIT(2),
    .mm_count       = ATOMIC_INIT(1),
    .mmap_sem       = __RWSEM_INITIALIZER(init_mm.mmap_sem),
    .page_table_lock =  __SPIN_LOCK_UNLOCKED(init_mm.page_table_lock),
    .mmlist         = LIST_HEAD_INIT(init_mm.mmlist),
    INIT_MM_CONTEXT(init_mm)
};
```
여기서 `mm_rb`는 가상 메모리 영역의 레드-블랙 트리이고,`pgd`는 페이지 글로벌 디렉토리에 대한 포인터이며,`mm_users`는 주소 공간 사용자,`mm_count`는 기본 사용 카운터,`mmap_sem`은 메모리 영역 세마포어입니다. 초기 프로세스의 메모리 설명자를 설정 한 후, 다음 단계는 'mpx_mm_init'를 사용하여 Intel Memory Protection Extensions를 초기화하는 것입니다. 다음 단계는 다음을 사용하여 코드/데이터/bss 리소스를 초기화하는 것입니다.


```C
	code_resource.start = __pa_symbol(_text);
	code_resource.end = __pa_symbol(_etext)-1;
	data_resource.start = __pa_symbol(_etext);
	data_resource.end = __pa_symbol(_edata)-1;
	bss_resource.start = __pa_symbol(__bss_start);
	bss_resource.end = __pa_symbol(__bss_stop)-1;
```


우리는 이미 'resource'구조체에 대해 약간은 알고 있습니다(위 참조). 여기에서는 코드/데이터/bss 리소스를 그들의 물리적 주소로 채 웁니다. 이를 `/proc/iomem`에서 볼 수 있습니다 :

```C
00100000-be825fff : System RAM
  01000000-015bb392 : Kernel code
  015bb393-01930c3f : Kernel data
  01a11000-01ac3fff : Kernel bss
```

이 모든 구조체는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c)에 정의되어 있으며 일반적인 리소스 초기화와 유사하게 생겼습니다. :

```C
static struct resource code_resource = {
	.name	= "Kernel code",
	.start	= 0,
	.end	= 0,
	.flags	= IORESOURCE_BUSY | IORESOURCE_MEM
};
```

이 파트에서 다루는 내용의 마지막 단계는 `NX` 설정입니다. `NX-비트`또는 실행 비트 없음은 페이지 디렉토리 항목에서 63 비트이며, 테이블 엔트리에 의해 매핑 된 모든 물리적 페이지에서부터 코드를 실행하는 기능을 제어합니다. 이 비트는 `no-execute` 페이지 보호 메커니즘이 `EFER.NXE`를 1로 설정하여 활성화 한 경우에만 사용/설정할 수 있습니다. `x86_configure_nx` 함수에서 CPU가 `NX-비트 '를 지원하는지, 비활성화되지 않는지 확인합니다. 검사 후 우리는 `__supported_pte_mask`를 채웁니다. :

```C
void x86_configure_nx(void)
{
        if (cpu_has_nx && !disable_nx)
                __supported_pte_mask |= _PAGE_NX;
        else
                __supported_pte_mask &= ~_PAGE_NX;
}
```

결론
--------------------------------------------------------------------------------

리눅스 커널 초기화 과정에 대한 다섯 번째 부분의 마지막입니다. 이 부분에서 우리는 아키텍처 고유의 것들을 초기화하는 `setup_arch` 함수에 계속해서 뛰어 들었습니다. 긴 파트였지만 끝내지 못했습니다. 앞서 작성한 것처럼 `setup_arch`는 큰 함수이므로 다음 파트에서도 모든 내용을 다룰 지 확실하지 않습니다. 이 파트에서  'Fix-mapped'주소, ioremap 등과 같은 새롭고 흥미로운 개념들이 있었습니다. 아직 확실히 이해하지 못했더라도 걱정하지 마세요.이 개념들에 대한 특별한 파트가 있습니다.-[리눅스 커널 메모리 관리 Part 2.](https://junsoolee.gitbook.io/linux-insides-ko/summary/mm/linux-mm-2). 다음 파트에서는 아키텍처 별 초기화를 계속헤사 디루고, 초기 커널 매개 변수, pci 장치의 초기 덤프, 데스크탑 관리 인터페이스 스캔과 기타 여러 가지 분석을 보게됩니다.


질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견이나 핑을 남겨주세요.

**영어는 제 모국어가 아닙니다. 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수를 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한글 번역](https://github.com/junsooo/linux-insides-ko)으로 PR을 보내주세요.**

Links
--------------------------------------------------------------------------------

* [mm vs active_mm](https://www.kernel.org/doc/Documentation/vm/active_mm.txt)
* [e820](http://en.wikipedia.org/wiki/E820)
* [Supervisor mode access prevention](https://lwn.net/Articles/517475/)
* [Kernel stacks](https://www.kernel.org/doc/Documentation/x86/x86_64/kernel-stacks)
* [TSS](http://en.wikipedia.org/wiki/Task_state_segment)
* [IDT](http://en.wikipedia.org/wiki/Interrupt_descriptor_table)
* [Memory mapped I/O](http://en.wikipedia.org/wiki/Memory-mapped_I/O)
* [CFI directives](https://sourceware.org/binutils/docs/as/CFI-directives.html)
* [PDF. dwarf4 specification](http://dwarfstd.org/doc/DWARF4.pdf)
* [Call stack](http://en.wikipedia.org/wiki/Call_stack)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)
