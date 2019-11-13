커널 부팅 과정. Part 4.
================================================================================

64 bit 모드로 전환
--------------------------------------------------------------------------------

 커널 부팅 프로세스의 네 번째 부분으로 우리는 [보호 모드](http://en.wikipedia.org/wiki/Protected_mode) 첫 단계를 배울 것 입니다. CPU가 [롱 모드](http://en.wikipedia.org/wiki/Long_mode) 와 [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)를 지원하는지 확인하고, [페이징](http://en.wikipedia.org/wiki/Paging), 페이지 테이블을 초기화하기 등 결국 우리는 [롱 모드](https://en.wikipedia.org/wiki/Long_mode)로의 전환에 대해 논의합니다.

**NOTE: there will be much assembly code in this part, so if you are not familiar with that, you might want to consult a book about it**

이전 [부분](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)에서 우리는 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S)의 `32-bit` 진입 점으로 점프하는 것을 멈췄습니다:

```assembly
jmpl	*%eax
```

우리는 eax 레지스터에는 32 비트 진입 점의 주소가 포함되어 있다는 것을 알 수 있었습니다. [리눅스 커널 x86 부팅 프로토콜](https://www.kernel.org/doc/Documentation/x86/boot.txt)에서 이것에 대해 읽을 수 있습니다:

```
bzImage를 사용할 때, 보호 모드 커널이 0x100000로 재배치 되었습니다.
```

32 비트 진입 점에서 레지스터 값을 확인하여 사실인지 확인해 봅시다:

```
eax            0x100000	1048576
ecx            0x0	    0
edx            0x0	    0
ebx            0x0	    0
esp            0x1ff5c	0x1ff5c
ebp            0x0	    0x0
esi            0x14470	83056
edi            0x0	    0
eip            0x100000	0x100000
eflags         0x46	    [ PF ZF ]
cs             0x10	16
ss             0x18	24
ds             0x18	24
es             0x18	24
fs             0x18	24
gs             0x18	24
```

 여기서 cs 레즈스터에 - `0x10` ([이전 부분](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)에서 기억할 수 있듯이, `Global Descriptor Table`의 두 번째 인덱스 인 것을 알 수 있습니다.)이 포함되어 있음을 알 수 있고, `eip` 레지스터에 `0x100000`이 포함되어 있으며 코드 세그먼트를 포함한 모든 세그먼트의 기본 주소는 0입니다.

따라서 실제 주소를 얻을 수 있습니다. 부팅 프로토콜에서 지정한대로, `0:0x100000` 또는 `0x100000` 입니다. 이제 32-bit 진입점부터 시작하겠습니다.

32-bit 진입점
--------------------------------------------------------------------------------

[arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) 어셈블리 소스 코드 파일에서 `32-bit` 진입 점의 정의를 찾을 수 있습니다:

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```

우선, 디렉토리 이름이 `compressed`인 이유는 무엇일까요? 실제로 `bzimage`는 압축 된 `vmlinux + 헤더 + 커널 설정 코드`입니다. 이전 내용에서 커널 설정 코드를 보았습니다. 따라서 `head_64.S`의 주요 목표는 롱 모드로 들어가서 준비한 다음 커널을 압축 해제하는 것입니다. 이 부분에서 커널 압축 해제까지의 모든 단계를 볼 수 있습니다.

`arch/x86/boot/compressed` 디렉터리에서 두개의 파일을 찾을 수 있습니다:

* [head_32.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_32.S)
* [head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)

이 책은 `x86_64`에만 관련되어 있기 때문에 `head_64.S` 소스 코드 파일 만 고려할 것입니다. [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/Makefile)을 봅시다. `make` 타깃을 찾을 수 있습니다:

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

`$(obj)/head_$(BITS).o`를 살펴봅시다.

이것은 `$(BITS)`가 `head_32.o` 또는 `head_64.o`로 설정된 것을 기반으로 연결할 파일을 선택한다는 의미입니다. `$(BITS)` 변수는 커널 구성에 따라 [arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)의 다른 곳에 정의됩니다:

```Makefile
ifeq ($(CONFIG_X86_32),y)
        BITS := 32
        ...
        ...
else
        BITS := 64
        ...
        ...
endif
```

이제 시작 위치를 알았으니 해보겠습니다.

필요시 세그먼트를 다시 로드
--------------------------------------------------------------------------------

위에서 설명한 것처럼, [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S)
어셈블리 소스 코드 파일에서 시작합니다. 먼저 startup_32 정의 전에 특수 섹션 속성의 정의를 봅니다:

```assembly
    __HEAD
    .code32
ENTRY(startup_32)
```

`__HEAD`는 [include / linux / init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) 헤더 파일에 정의 된 매크로이며 다음 섹션의 정의를 확장한다:

```C
#define __HEAD		.section	".head.text","ax"
```

`.head.text` 이름과 `ax` 플래그와 함께. 이 경우이 플래그는이 섹션이 [실행 가능](https://en.wikipedia.org/wiki/Executable)이거나 다른 말로 코드를 포함하고 있음을 나타냅니다. 이 섹션의 정의는 [arch / x86 / boot / compressed / vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/vmlinux에서 찾을 수 있습니다. .lds.S) 링커 스크립트:

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
     }
     ...
     ...
     ...
}
```

`GNU LD` 링커 스크립팅 언어의 구문에 익숙하지 않은 경우 다음 [문서](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts)에서 자세한 정보를 찾을 수 있습니다. 위치 카운터 - 즉,`.` 기호는 링커의 특별한 변수입니다. 여기에 할당 된 값은 세그먼트와 관련된 오프셋입니다. 이 경우 위치 카운터에 0을 할당합니다. 이것은 우리 코드가 메모리의`0` 오프셋에서 실행되도록 연결되어 있음을 의미합니다. 또한 이 정보를 주석에서 찾을 수 있습니다:

```
head_64.S의 일부분은 startup_32가 주소 0에 있다고 가정한다는 거에 주의하세요.

```

자, 이제 우리는 현재 위치를 알고 있으며, 이제`startup_32` 함수를 살펴볼 시간이다.

`startup_32` 함수의 시작 부분에서 우리는 [flags](https://en.wikipedia.org/wiki/FLAGS_register) 레지스터에서 `DF` 비트를 지우는 `cld` 명령을 볼 수 있습니다. 방향 플래그가 지워지면 [stos](http://x86.renejeschke.de/html/file_module_x86_id_306.html), [scas](http://x86.renejeschke.de/html/file_module_x86_id_287.html) 등과 같은 모든 문자열 연``은 인덱스 레지스터 `esi` 또는`edi`를 증가시킵니다. 나중에 페이지 테이블 등의 공간을 비우기 위해 문자열 연산을 사용하므로 방향 플래그를 지워야합니다.

`DF` 비트를 클리어 한 후 다음 단계는 `loadflags` 커널 설정 헤더 필드에서 `KEEP_SEGMENTS` 플래그를 점검하는 것입니다. 기억할 지 모르겠지만 우리는 이미 책의 맨 처음 [부분](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html) 에서 'loadflags'를 보았습니다. 거기에서 힙을 사용하기 위해 `CAN_USE_HEAP` 플래그를 확인했습니다. 이제 'KEEP_SEGMENTS'플래그를 확인해야합니다. 이 플래그는 리눅스 [부팅 프로토콜](https://www.kernel.org/doc/Documentation/x86/boot.txt) 문서에 설명되어 있습니다:

```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - If 0, reload the segment registers in the 32bit entry point.
  - If 1, do not reload the segment registers in the 32bit entry point.
    Assume that %cs %ds %ss %es are all set to flat segments with
		a base of 0 (or the equivalent for their environment).

```

따라서 만약 `KEEP_SEGMENTS` 비트가 `loadflags`에 설정되어 있지 않다면, `ds`,`ss` 및 `es` 세그먼트 레지스터를 기준이 `0` 인 데이터 세그먼트의 인덱스로 설정해야합니다:

```C
	testb $KEEP_SEGMENTS, BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss

```

`__BOOT_DS`는 `0x18` ([글로벌 디스크립터 테이블](https://en.wikipedia.org/wiki/Global_Descriptor_Table)의 데이터 세그먼트 색인)인것을 기억합시다. 만약 `KEEP_SEGMENTS`가 설정되면 가장 가까운 `1f` 레이블로 이동하거나 설정되지 않은 경우 `__BOOT_DS`로 세그먼트 레지스터를 업데이트합니다. 꽤 쉽지만 여기서 흥미로운 점이 있습니다. 이전 [부분](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)을 읽었다면 [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux)에서 [보호 모드](https://en.wikipedia.org/wiki/Protected_mode)로 전환 한 직후 세그먼트 레지스터들을 이미 업데이트 한 것을 알 수 있습니다. 그렇다면 왜 세그먼트 레지스터의 값을 다시 신경 써야합니까? 대답은 쉽습니다. 리눅스 커널은 32-bit 부트 프로토콜을 가지고 있으며 부트 로더가 이를 사용하여 리눅스 커널을 로드한다면 `startup_32` 전의 모든 코드가 누락됩니다.이 경우,`startup_32`는 부트 로더 바로 다음에 Linux 커널의 첫 번째 진입 점이 될 것이며 세그먼트 레지스터가 알려진 상태에 있다고 보장할 수 없습니다.

`KEEP_SEGMENTS` 플래그를 확인하고 세그먼트 레지스터에 올바른 값을 넣은 후 다음 단계는 로드한 위치와 실행하기 위해 컴파일한 위치의 차이를 계산하는 것입니다. `setup.ld.S`에는 다음 정의가 포함되어 있습니다: `.head.text` 섹션의 첫 부분에 `. = 0` 이것은 이 섹션의 코드가 `0` 주소에서 실행되기 위해 컴파일되었음을 의미합니다. 우리는 `objdump` 출력에서 ​​이것을 볼 수 있습니다:

```
arch/x86/boot/compressed/vmlinux:     파일 포맷 elf64-x86-64


.head.text 섹션의 분해 :

0000000000000000 <startup_32>:
   0:   fc                      cld
   1:   f6 86 11 02 00 00 40    testb  $0x40,0x211(%rsi)
```

`objdump` 유틸리티는 `startup_32`의 주소가 `0`이라고 알려주지만 실제로는 그렇지 않습니다. 우리의 현재 목표는 실제로 우리가 어디에 있는지 아는 것입니다. `rip` 상대 주소 지정을 지원하기 때문에 [롱 모드](https://en.wikipedia.org/wiki/Long_mode)에서 하는 것이 매우 간단하지만 현재는 [보호 모드](https://en.wikipedia.org/wiki/Protected_mode). `startup_32`의 주소를 알기 위해 공통 패턴을 사용할 것입니다. 레이블을 정의하고 이 레이블에 호출하고 스택의 상단을 레지스터로 팝해야합니다.

```assembly
call label
label: pop %reg
```

그 후,`% reg` 레지스터는 레이블의 주소를 포함 할 것입니다. 리눅스 커널에서 `startup_32`의 주소를 검색하는 비슷한 코드를 봅시다:

```assembly
        leal	(BP_scratch+4)(%esi), %esp
        call	1f
1:      popl	%ebp
        subl	$1b, %ebp
```

이전 부분에서 보았듯 `esi` 레지스터에는 우리가 보호 모드로 이동하기 전에 채워진 구조인 [boot_params](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L113) 구조가 있다. `boot_params` 구조는 오프셋이 0x1e4 인 특수 필드 `scratch`를 포함합니다. 이 4 바이트 필드는 `call` 명령을 위한 임시 스택입니다. `scratch` 필드 +`4` 바이트의 주소를 가져 와서`esp` 레지스터에 넣습니다. 방금 설명한 것처럼 임시 스택이고 스택은`x86_64` 아키텍처에서 위에서 아래로 커지기 때문에 'BP_scratch'필드의 베이스에 `4` 바이트를 추가합니다. 따라서 스택 포인터는 스택의 상단을 가리 킵니다. 다음으로 위에서 설명한 패턴을 볼 수 있습니다. `call` 명령어가 실행 된 후 스택 맨 위에 리턴 주소가 있으므로`1f` 레이블을 호출하고 이 레이블의 주소를`ebp` 레지스터에 넣습니다. 이제 우리는`1f` 레이블의 주소를 가지게되었고 이제는`startup_32`의 주소를 얻는 것이 쉽습니다. 스택에서 얻은 주소에서 레이블 주소를 빼면됩니다:

```
startup_32 (0x0)     +-----------------------+
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
1f (0x0 + 1f offset) +-----------------------+ %ebp - real physical address
                     |                       |
                     |                       |
                     +-----------------------+
```

`startup_32`는 `0x0` 주소에서 실행되도록 연결되어 있으며,`1f`는 `0x0 + 1f`로 오프셋 된 주소, 약 `0x21` 바이트를 의미합니다. `ebp` 레지스터는`1f` 라벨의 실제 물리적 주소를 포함합니다. 따라서 `ebp`에서`1f`를 빼면 `startup_32`의 실제 물리적 주소를 얻게됩니다. 리눅스 커널 [부트 프로토콜](https://www.kernel.org/doc/Documentation/x86/boot.txt)은 보호 모드 커널의 베이스가`0x100000`이라고 설명합니다. [gdb](https://en.wikipedia.org/wiki/GNU_Debugger)에서 이를 확인할 수 있습니다. 디버거를 시작하고 중단 점을 `1f` 주소 `0x100021`에 넣습니다. 이것이 맞다면 `ebp` 레지스터에 `0x100021`이 보일 것입니다:

```
$ gdb
(gdb)$ target remote :1234
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb)$ br *0x100022
Breakpoint 1 at 0x100022
(gdb)$ c
Continuing.

Breakpoint 1, 0x00100022 in ?? ()
(gdb)$ i r
eax            0x18	0x18
ecx            0x0	0x0
edx            0x0	0x0
ebx            0x0	0x0
esp            0x144a8	0x144a8
ebp            0x100021	0x100021
esi            0x142c0	0x142c0
edi            0x0	0x0
eip            0x100022	0x100022
eflags         0x46	[ PF ZF ]
cs             0x10	0x10
ss             0x18	0x18
ds             0x18	0x18
es             0x18	0x18
fs             0x18	0x18
gs             0x18	0x18
```

다음 명령 `subl $ 1b, % ebp`를 실행하면:

```
(gdb) nexti
...
...
...
ebp            0x100000	0x100000
...
...
...
```

맞습니다. `startup_32`의 주소는`0x100000`입니다. `startup_32` 레이블의 주소를 알고 나면 [롱 모드](https://en.wikipedia.org/wiki/Long_mode)로의 전환을 준비 할 수 있습니다. 다음 목표는 스택을 설정하고 CPU가 롱 모드와 [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)를 지원하는지 확인하는 것입니다.

스택 설정 및 CPU 검증
--------------------------------------------------------------------------------

`startup_32` 레이블의 주소를 모르는 동안 스택을 설정할 수 없습니다. 스택을 배열로 생각할 수 있으며 스택 포인터 레지스터 'esp'는 이 배열의 끝을 가리켜야 합니다. 물론 코드에서 배열을 정의 할 수 있지만 올바른 방법으로 스택 포인터를 구성하려면 실제 주소를 알아야합니다. 코드를 봅시다:

```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

동일한 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)어셈블리 소스 코드 파일에 정의 된 `boot_stack_end`는 [.bss](https://en.wikipedia.org/wiki/.bss) 섹션에 있습니다:

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

먼저, 우리는 `boot_stack_end`의 주소를 `eax` 레지스터에 넣습니다. 그래서 `eax` 레지스터는 그것이 연결된 `boot_stack_end`의 주소, 즉 `0x0 + boot_stack_end`를 포함합니다. `boot_stack_end`의 실제 주소를 얻으려면`startup_32`의 실제 주소를 추가해야합니다. 알고 있듯이, 우리는 이 주소를 찾아서 `ebp` 레지스터에 넣었습니다. 결국, 레지스터 `eax`는 `boot_stack_end`의 실제 주소를 포함 할 것이고 우리는 그저 스택 포인터에 넣으면 됩니다.

스택을 설정 한 후 다음 단계는 CPU 확인입니다. `롱 모드`로의 전환을 수행 할 때 CPU가 `롱 모드`및 'SSE'를 지원하는지 확인해야합니다. 우리는 `verify_cpu` 함수의 호출을 통해 확인 할 것입니다:

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

이 기능은 [arch/x86/kernel/verify_cpu.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/kernel/verify_cpu.S) 어셈블리 파일에 정의되어 있으며 [cpuid](https://en.wikipedia.org/wiki/CPUID)명령어에 대한 몇 번의 호출만을 포함합니다. 이 명령어는 프로세서에 대한 정보를 얻는 데 사용됩니다. 이 경우에는 `롱 모드` 및 `SSE` 지원을 확인하고 `eax`레지스터에서 성공하면 `0`, 실패하면 `1`을 반환합니다.

`eax`의 값이 0이 아닌 경우 하드웨어 인터럽트가 발생하지 않으면서 `hlt` 명령을 호출하여 CPU를 중지시키는 `no_longmode` 레이블로 점프합니다:

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

`eax` 레지스터의 값이 0이면 모든 것이 정상이며 계속할 수 있습니다.

재배치 주소 계산
--------------------------------------------------------------------------------

다음 단계는 필요한 경우 압축 해제를 위한 재배치 주소를 계산하는 것입니다. 먼저, 커널이 `재배치 가능하다`는 것이 무엇을 의미하는지 알아야합니다. 우리는 이미 리눅스 커널의 32 비트 진입 점의 기본 주소는`0x100000`인 것을 이미 알고 있습니다. 하지만 이것은 32 비트 진입 점입니다. 리눅스 커널의 기본 주소는 `CONFIG_PHYSICAL_START` 커널 설정 옵션의 값에 의해 결정됩니다. 기본값은 `0x1000000` 또는`16 MB`입니다. 여기서 가장 큰 문제는 리눅스 커널이 충돌하면 커널 개발자는 다른 주소에서 로드하도록 구성된 [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt)에 대한 `rescue kernel`이 있어야 한다는 것입니다. 리눅스 커널은 이 문제를 해결하기위한 특별한 설정 옵션을 제공합니다 :`CONFIG_RELOCATABLE`. 리눅스 커널의 문서에서 볼 수 있다:

```
재배치 정보를 유지하는 커널 이미지를 만듭니다.
1MB가 아닌 다른 곳에 로드 할 수 있습니다.

참고 : CONFIG_RELOCATABLE = y 인 경우 커널은 로드 된 주소와 컴파일 시간 실제 주소에서 실행됩니다.
(CONFIG_PHYSICAL_START)가 최소 위치로 사용됩니다.
```

간단히 말해, 구성이 동일한 Linux 커널을 다른 주소에서 부팅 할 수 있습니다. 기술적으로 이것은 decompressor를 [위치 독립적 코드] (https://en.wikipedia.org/wiki/Position-independent_code)로 컴파일하여 수행됩니다. [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/Makefile)을 보면, decompressor는 실제로`-fPIC` 플래그로 컴파일됩니다:

```Makefile
KBUILD_CFLAGS += -fno-strict-aliasing -fPIC
```

위치 독립적인 코드를 사용하는 경우 프로그램 카운터 값에 명령어의 주소 필드를 추가하여 주소를 얻습니다. 우리는 어떤 주소에서든 그러한 주소를 사용하는 코드를 로드 할 수 있습니다. 그래서 우리는 `startup_32`의 실제 물리적 주소를 얻어야 했습니다. 이제 리눅스 커널 코드로 돌아가 봅시다. 현재 목표는 압축 해제를 위해 커널을 재배치 할 수 있는 주소를 계산하는 것입니다. 이 주소의 계산은 `CONFIG_RELOCATABLE` 커널 구성 옵션에 따라 다릅니다. 코드를 봅시다:

```assembly
#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jge	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
```

`ebp` 레지스터의 값은 `startup_32` 레이블의 물리적 주소입니다. 커널 구성 중에 `CONFIG_RELOCATABLE` 커널 구성 옵션이 활성화 된 경우, 이 주소를 `ebx` 레지스터에 넣고 이를 `2MB`의 배수에 맞추고 `LOAD_PHYSICAL_ADDR` 값과 비교합니다. `LOAD_PHYSICAL_ADDR` 매크로는 [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/boot.h) 헤더 파일에 정의되어 있습니다. 다음과 같습니다:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

보시다시피 커널을 로드할 물리적 주소를 나타내는 정렬된 `CONFIG_PHYSICAL_ALIGN`값으로 확장됩니다. `LOAD_PHYSICAL_ADDR`과 `ebx` 레지스터의 값을 비교 한 후 압축 된 커널 이미지를 압축 해제 할 `startup_32`의 오프셋을 추가합니다. 커널 설정 중에`CONFIG_RELOCATABLE` 옵션이 활성화되어 있지 않다면 커널을 로드 할 기본 주소를 넣고 `z_extract_offset`를 추가하면됩니다.

이 모든 계산 후에, 우리는 그것을 로드 한 주소를 포함하는 `ebp`와 압축 해제 후 커널이 이동 될 주소로 설정된 'ebx'를 갖게됩니다. 그러나 이것이 끝이 아닙니다. 압축 된 커널 이미지는 압축 해제 버퍼의 끝으로 이동하여 커널이 나중에 위치 할 계산을 단순화해야합니다. 이를 위해:

```assembly
1:
    movl	BP_init_size(%esi), %eax
    subl	$_end, %eax
    addl	%eax, %ebx
```

우리는 `boot_params.BP_init_size`(또는 `hdr.init_size`의 커널 설정 헤더 값)에서 `eax` 레지스터에 값을 넣습니다. `BP_init_size`는 압축 및 비 압축 [vmlinux](https://en.wikipedia.org/wiki/Vmlinux) 중 더 큰 값을 포함합니다. 다음으로 이 값에서 `_end` 심볼의 주소를 빼고 빼기 결과를 커널 압축 해제를 위한 기본 주소를 저장하는`ebx` 레지스터에 추가합니다.

롱 모드로 들어가기 전에 준비
--------------------------------------------------------------------------------

압축 된 커널 이미지를 재배치 할 기본 주소가 있으면 64 비트 모드로 전환하기 전에 마지막 단계를 수행해야합니다. 먼저 재배치 가능한 커널이 512G 미만의 주소에서 실행될 수 있으므로 64 비트 세그먼트로 [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table)을 업데이트해야합니다:

```assembly
	addl	%ebp, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

여기서 Global Descriptor 테이블의 기본 주소를 실제로 로드 한 주소로 조정하고 `lgdt` 명령으로 `Global Descriptor Table`을 로드합니다.

이해하기 위해서 `gdt` 오프셋으로 `Global Descriptor Table`의 정의를 살펴 봐야합니다. 동일한 소스 코드 [파일](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)에서 정의를 찾을 수 있습니다:

```assembly
	.data
gdt64:
	.word	gdt_end - gdt
	.long	0
	.word	0
	.quad   0
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x00cf9a000000ffff	/* __KERNEL32_CS */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
gdt_end:
```

우리는 `.data`섹션에 있으며 5 개의 디스크립터를 포함하고 있음을 알 수 있다. 첫 번째는 커널 코드 세그먼트를 위한 32 비트 디스크립터, 64 비트 커널 세그먼트, 커널 데이터 세그먼트 및 2 개의 태스크 디스크립터이다.

우리는 이미 이전 [부분](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)에서 `Global Descriptor Table`을 로드했으며, 거의 똑같지만 `64` 비트 모드에서 실행하기 위해 `CS.L = 1` 및 `CS.D = 0` 인 디스크립터이다. 보시다시피 `gdt`의 정의는 두 바이트부터 시작합니다: `gdt_end-gdt`는 `gdt` 테이블 또는 테이블 제한의 마지막 바이트를 나타냅니다. 다음 4 바이트는`gdt`의 기본 주소를 포함합니다.

`lgdt` 명령어로 `Global Descriptor Table`을로드 한 후,`cr4` 레지스터의 값을 `eax`에 넣어 [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension)를 활성화해야합니다. 5번째 비트를 설정하고 다시 `cr4`로 로드:

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

이제 64 비트 모드로 전환하기 전에 모든 준비가 거의 끝났습니다. 마지막 단계는 페이지 테이블을 작성하는 것이지만 그 전에 롱 모드에 대한 정보가 있습니다.

롱 모드
--------------------------------------------------------------------------------

[롱 모드](https://en.wikipedia.org/wiki/Long_mode)는 [x86_64](https://en.wikipedia.org/wiki/X86-64) 프로세서의 기본 모드입니다. 먼저 `x86_64`와 `x86`의 차이점을 살펴 보겠습니다.

`64 비트` 모드는 다음과 같은 기능을 제공합니다:

* `r8`에서 `r15`까지의 새로운 8 개의 범용 레지스터 + 모든 범용 레지스터는 64 비트입니다;
* 64 비트 명령어 포인터 - `RIP`;
* 새로운 작동 모드 - 롱 모드;
* 64 비트 주소 및 피연산자;
* RIP 상대 주소 지정 (다음 부분에서 예를 볼 것입니다).

긴 모드는 레거시 보호 모드의 확장입니다. 두 개의 하위 모드로 구성됩니다:

* 64-bit 모드;
* 호환 모드.

`64 비트` 모드로 전환하려면 다음을 수행해야합니다:

* [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension) 활성화;
* 페이지 테이블을 빌드하고 최상위 페이지 테이블의 주소를 `cr3` 레지스터에 로드;
* `EFER.LME` 활성화;
* 페이징 활성화.

`cr4` 제어 레지스터에서 `PAE`비트를 설정하여 이미 `PAE`를 활성화했습니다. 다음 목표는 [paging](https://en.wikipedia.org/wiki/Paging)의 구조를 구축하는 것입니다. 우리는 이것을 다음 단락에서 볼 것입니다.

초기 페이지 테이블 초기화
--------------------------------------------------------------------------------

우리는 이미 64 비트 모드로 이동하기 전에 페이지 테이블을 만들어야한다는 것을 알고 있으므로 초기 4G 부팅 페이지 테이블의 구축을 살펴 보자.

**참고 : 여기에서는 가상 메모리 이론을 설명하지 않습니다. 그것에 대해 더 자세히 알고 싶다면 이 부분의 끝에있는 링크를 참조하십시오.**

리눅스 커널은 4 단계 페이징을 사용하며 일반적으로 6 개의 페이지 테이블을 만듭니다:

* 하나의 `PML4` 또는 하나의 항목이있는 `페이지 맵 레벨 4` 테이블;
* 하나의 `PDP` 또는 네 개의 항목이있는 `Page Directory Pointer` 테이블;
* 총 2048 개의 항목이있는 4 개의 페이지 디렉토리 테이블.

구현해 봅시다. 우선, 메모리에서 페이지 테이블의 버퍼를 지웁니다. 모든 테이블은 `4096` 바이트이므로 정확히 `24` 킬로바이트 버퍼가 필요합니다:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$(BOOT_INIT_PGT_SIZE/4), %ecx
	rep	stosl
```

`edi` 레지스터에 `pgtable` + `ebx`의 주소 (`ebx`에는 압축 해제를 위해 커널을 재배치 할 주소가 들어 있음을 기억하십시오)를 넣고 `eax` 레지스터를 지우고 `ecx` 레지스터를 `6144`로 설정합니다.

`rep stosl` 명령어는 `eax`의 값을 `edi`에 기록하고 `edi` 레지스터의 값을 `4` 늘리고 `ecx` 레지스터의 값을 `1`줄입니다. 이 동작은 `ecx`레지스터의 값이 0보다 큰 동안 반복됩니다. 그래서 우리는 `6144` 또는 `BOOT_INIT_PGT_SIZE/4`를 `ecx`에 넣었습니다.

`pgtable`은 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) 어셈블리 파일의 끝에서 정의됩니다:

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill BOOT_PGT_SIZE, 1, 0
```

보다시피 `.pgtable` 섹션에 있으며 크기는 `CONFIG_X86_VERBOSE_BOOTUP` 커널 설정 옵션에 따라 다릅니다:

```C
#  ifdef CONFIG_X86_VERBOSE_BOOTUP
#   define BOOT_PGT_SIZE	(19*4096)
#  else /* !CONFIG_X86_VERBOSE_BOOTUP */
#   define BOOT_PGT_SIZE	(17*4096)
#  endif
# else /* !CONFIG_RANDOMIZE_BASE */
#  define BOOT_PGT_SIZE		BOOT_INIT_PGT_SIZE
# endif
```

`pgtable` 구조에 대한 버퍼를 확보 한 후에는 최상위 페이지 테이블 인 `PML4`를 만들 수 있습니다:

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

여기서 다시 우리는 `ebx`에 상대적인, 다시 말해 `startup_32`의 주소에 상대적인 `pgtable`의 주소를 `edi` 레지스터에 넣었다. 다음으로, 이 주소를 `eax` 레지스터에 오프셋 `0x1007`로 넣습니다. `0x1007`은 `PML4`+`7`의 크기 인 `4096` 바이트입니다. 여기서 `7`은 `PML4` 항목의 플래그를 나타냅니다. 이 경우 이 플래그는 `PRESENT+RW+USER` 입니다. 결국 첫 번째 `PDP` 항목의 주소를 먼저 `PML4`에 씁니다.

다음 단계에서 우리는 동일한 `PRESENT+RW+USE` 플래그를 가진 `Page Directory Pointer` 테이블에 4 개의 `Page Directory` 엔트리를 구축 할 것입니다:

```assembly
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:  movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

`pgtable` 테이블에서 `4096` 또는 `0x1000` 오프셋인 페이지 디렉토리 포인터의 기본 주소를 `edi`레시스터에 넣고 첫 번째 페이지 디렉토리 포인터 항목의 주소를 `eax` 레지스터에 넣습니다. `ecx` 레지스터에 `4`를 넣으면, 다음 루프의 카운터가되고 첫 페이지 디렉토리 포인터 테이블 엔트리의 주소를 `edi` 레지스터에 씁니다. 이 `edi`에는 `0x7` 플래그가 있는 첫 번째 페이지 디렉토리 포인터 항목의 주소가 포함됩니다. 다음으로 각 항목이 `8` 바이트 인 다음 페이지 디렉토리 포인터 항목의 주소를 계산하고 주소를 `eax`에 씁니다. 페이징 구조를 구축하는 마지막 단계는 `2-MByte` 페이지로 `2048` 페이지 테이블 항목을 구축하는 것입니다:

```assembly
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:  movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

여기서 우리는 앞에서와 거의 같은 일을합니다. 모든 항목에는 플래그 - `$ 0x00000183`-`PRESENT + WRITE + MBZ`가 있습니다. 결국, 우리는`2-MByte` 페이지를 가진 `2048` 페이지 또는:

```python
>>> 2048 * 0x00200000
4294967296
```

`4G` 페이지 테이블. `4` 기가 바이트의 메모리를 매핑하는 초기 페이지 테이블 구조 구축을 마쳤으며 이제는 `cr3` 제어 레지스터에 상위 레벨 페이지 테이블 - `PML4` - 의 주소를 넣을 수 있습니다:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

그게 다입니다. 모든 준비가 완료되었으며 이제 롱 모드로 전환되는 것을 볼 수 있습니다.

64 비트 모드로 전환
--------------------------------------------------------------------------------

우선 [MSR](http://en.wikipedia.org/wiki/Model-specific_register)의 `EFER.LME` 플래그를 `0xC0000080`으로 설정해야합니다:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

`ecx` 레지스터에 `MSR_EFER` 플래그를 넣습니다([arch/x86/include/asm/msr-index.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86에 정의되어 있음)) [MSR](http://en.wikipedia.org/wiki/Model-specific_register) 레지스터를 읽는 `rdmsr` 명령을 호출하십시오. `rdmsr`이 실행되면 `edx : eax`에 결과 데이터를 얻고 이는 `ecx` 값에 따라 달라집니다. `btsl` 명령어로 `EFER_LME` 비트를 확인하고 `wrmsr` 명령어로 `eax`에서 `MSR` 레지스터로 데이터를 씁니다.

다음 단계에서는 커널 세그먼트 코드의 주소를 스택으로 푸시하고 (GDT에 정의) `startup_64` 루틴의 주소를 `eax`에 넣습니다.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

그런 다음 이 주소를 스택으로 푸시하고 `cr0` 레지스터에서 `PG` 및`PE` 비트를 설정하여 페이징을 활성화합니다:

```assembly
	pushl	%eax
    movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

그리고 실행합니다:

```assembly
lret
```

설명.

이전 단계에서 `startup_64` 함수의 주소를 스택으로 푸시했고 `lret` 명령 후에 CPU가 주소를 추출하여 점프합니다.

이 모든 단계를 마친 후 마침내 64 비트 모드에 있습니다:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

이게 답니다!

결론
--------------------------------------------------------------------------------

이것이 리눅스 커널 부팅 과정의 네 번째 부분입니다. 질문이나 제안이 있으면 트위터 [0xAX](https://twitter.com/0xAX)나 [email](anotherworldofworld@gmail.com)을 보내거나 [issue](https://github.com/0xAX/linux-insides/issues/new)를 만드십시오.

다음 부분에서는 커널 압축 해제 등을 볼 수 있습니다.

**모국어가 영어가 아니면 죄송합니다. 실수를 발견하면 PR을 [linux-insides](https://github.com/0xAX/linux-internals)로 보내주십시오.**

링크
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [GNU linker](http://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf)
* [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
* [Paging](http://en.wikipedia.org/wiki/Paging)
* [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [.fill instruction](http://www.chemie.fu-berlin.de/chemnet/use/info/gas/gas_7.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)
* [Paging on osdev.org](http://wiki.osdev.org/Paging)
* [Paging Systems](https://www.cs.rutgers.edu/~pxk/416/notes/09a-paging.html)
* [x86 Paging Tutorial](http://www.cirosantilli.com/x86-paging/)
