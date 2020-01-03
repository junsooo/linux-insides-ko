커널 초기화. Part 3.
================================================================================

커널 진입 포인트 전에 마지막 준비
--------------------------------------------------------------------------------

이것은 리눅스 커널 초기화 프로세스 시리즈의 세번째 파트 입니다. 이전 [파트](https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-2.md)에서는 early 인터럽트 및 예외처리를 봤으며 현재 파트에서는 리눅스 커널 초기화 프로세스를 계속할 것입니다. 다음 포인트는 커널 진입 포인트입니다. `start_kernel`함수는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)소스 코드 파일에 있습니다. 예, 기술적으로는 커널 진입 포인트가 아니라 특정 아키텍처에 의존하지 않는 일반 커널 코드의 시작입니다. `start_kernel`함수를 호출하기 전에 준비해야 할 것이 있습니다. 계속합시다.

다시 boot_params으로
--------------------------------------------------------------------------------

이전 파트에서 인터럽트 디스크립터 테이블을 설정을 중지하고 `IDTR`레지스터를 불러왔습니다. 다음 단계에서 `copy_bootdata`함수를 호출합니다: 

```C
copy_bootdata(__va(real_mode_data));
```

이 함수는 하나의 인자-가상 주소`real_mode_data`를 사용합니다. [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/uapi/asm/bootparam.h#L114)의 `boot_params`구조체의 주소를 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S)의 첫번째 인자 `x86_64_start_kernel`함수에 넘긴것을 기억하십시오:

```
	/* rsi는 C에서 넘겨진 흥미로운 정보의 
        리얼 모드 구조체 포인터 입니다.*/
	movq	%rsi, %rdi
```

이제 `__va`매크로를 봅니다. 이 매크로는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에 정의되었습니다:

```C
#define __va(x)                 ((void *)((unsigned long)(x)+PAGE_OFFSET))
```

`PAGE_OFFSET`은 `0xffff880000000000`및 모든 물리 메모리의 직접 매핑 가상 주소의 기반인 `__PAGE_OFFSET`에 위치합니다. 그래서 `boot_params`구조의 가상주소를 얻고 그것을 [arch/x86/include/asm/setup.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/setup.h)에 선언된 `boot_params`의 `real_mod_data`를 복사한 곳의 `copy_bootdata`함수에 넘깁니다.


```C
extern struct boot_params boot_params;
```

`copy_boot_data`의 구현을 봅시다:

```C
static void __init copy_bootdata(char *real_mode_data)
{
	char * command_line;
	unsigned long cmd_line_ptr;

	memcpy(&boot_params, real_mode_data, sizeof boot_params);
	sanitize_boot_params(&boot_params);
	cmd_line_ptr = get_cmd_line_ptr();
	if (cmd_line_ptr) {
		command_line = __va(cmd_line_ptr);
		memcpy(boot_command_line, command_line, COMMAND_LINE_SIZE);
	}
}
```

우선 이 함수는 `__init`접두사로 선언됩니다. 이것은 이 함수가 초기화 중에만 사용되고 사용된 메모리가 해제되는 것을 의미합니다.

커널 커맨드 라인의 두 변수를 선언하고 `memcpy`함수의 `boot_params`에서 `real_mode_data`를 복사하는 것을 볼 수 있습니다. `boot_params`이 0이어서 알수없는 필드의 초기화에 실패한 부트로더의 경우 `boot_params`구조체와 같은 `ext_ramdisk_image`등의 영역이 약간 채워진 `sanitize_boot_params`함수를 호출합니다. 다음으로 `get_cmd_line_ptr`함수를 호출해 커맨드 라인의 주소를 얻습니다:

```C
unsigned long cmd_line_ptr = boot_params.hdr.cmd_line_ptr;
cmd_line_ptr |= (u64)boot_params.ext_cmd_line_ptr << 32;
return cmd_line_ptr;
```

커널 부트헤더에서 커맨드 라인의 64비트 주소를 얻고 이것을 반환합니다. 마지막으로 `cmd_line_ptr`을 확인하고 이것의 가상주소를 받아 바이트 배열 `boot_command_line`에 복사합니다:

```C
extern char __initdata boot_command_line[];
```

커널 커맨드라인과 `boot_params`구조체가 복사됩니다. 다음 단계에서는 `load_ucode_bsp` 프로세서 마이크로 코드를 호출하는 함수를 볼 수 있지만, 여기서는 아닙니다.

마이크로코드가 불려진 다음 `console_loglevel`과 `Kernel Alive` 문자열을 출력하는 `early_printk`함수를 확인하는 것을 볼 수 있습니다. 그러나 아직 `early_printk`가초기화되지 않아서 출력은 볼 수 없습니다. 이것은 커널의 사소한 버그로 패치를 보냈습니다 - [커밋](http://git.kernel.org/cgit/linux/kernel/git/tip/tip.git/commit/?id=91d8f0416f3989e248d3a3d3efb821eda10a85d2)하면 메인라인에서 볼 수 있습니다. 따라서 이 코드를 건너 뛸 수 있습니다.

초기화 페이지로 이동
--------------------------------------------------------------------------------

다음 단계에서는 `boot_params`구체조가 복사됬으니 초기화 프로세스를 위해 초기 페이지 테이블에서 페이지 테이블로 이동해야 합니다. 이미 전환을 위해 초기 페이지 테이블을 설정했습니다. 이전 [파트](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)에서 읽을 수 있으며 `reset_early_page_tables`함수에서 모두 떨어트릴 수 있고(이전 파트에서 볼 수 있음) 커널의 높은 매핑을 유지합니다. 그 다음 호출합니다:

```C
	clear_page(init_level4_pgt);
```

함수 및 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head_64.S)에 정의된 `init_level4_pgt`을 넘깁니다:

```assembly
NEXT_PAGE(init_level4_pgt)
	.quad   level3_ident_pgt - __START_KERNEL_map + _KERNPG_TABLE
	.org    init_level4_pgt + L4_PAGE_OFFSET*8, 0
	.quad   level3_ident_pgt - __START_KERNEL_map + _KERNPG_TABLE
	.org    init_level4_pgt + L4_START_KERNEL*8, 0
	.quad   level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE
```

커널 코드, 데이터, bss에 대해 처음에 2기가바이트와 512메가바이트를 매핑합니다. `clear_page`함수는 [arch/x86/lib/clear_page_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/lib/clear_page_64.S)에 정의됬습니다. 이 함수를 봅시다:

```assembly
ENTRY(clear_page)
	CFI_STARTPROC
	xorl %eax,%eax
	movl $4096/64,%ecx
	.p2align 4
	.Lloop:
    decl	%ecx
#define PUT(x) movq %rax,x*8(%rdi)
	movq %rax,(%rdi)
	PUT(1)
	PUT(2)
	PUT(3)
	PUT(4)
	PUT(5)
	PUT(6)
	PUT(7)
	leaq 64(%rdi),%rdi
	jnz	.Lloop
	nop
	ret
	CFI_ENDPROC
	.Lclear_page_end:
	ENDPROC(clear_page)
```

함수 이름에서 알 수 있듯이 0페이지 테이블을 지우거나 채웁니다. 우선 이 함수는 `CFI_STARTPROC` 및 GNU 어셈블리 명령어로 확장된 `CFI_ENDPROC`로 시작합니다.

```C
#define CFI_STARTPROC           .cfi_startproc
#define CFI_ENDPROC             .cfi_endproc
```

그리고 디버깅이 쓰입니다. `CFI_STARTPROC`매크로 후에 `eax`레지스터를 0으로 만들고 `ecx`(이것은 counter이 됩니다)에 64를 넣습니다. 다음으로 `.Lloop`레이블로 시작하는 루프를 볼 수 있고 이것은 `ecx`의 감소에서 시작됩니다. 다음으로 `init_level4_pgt`의 기본 주소를 포함하는 `rdi`의 `rax`레지스터에 0을 넣고 동일한 절차를 7번 수행합니다. 하지만 `rdi`오프셋은 매번 8로 이동합니다. 이후 `init_level4_pgt`의 첫 64바이트는 0으로 채워집니다. 다음 단계에서 `init_level4_pgt` 64-바이트 오프셋의 주소를 `rdi`에 넣고 `ecx`가 0이 될때까지 모든 작업을 반복합니다. 결국 `init_level4_pgt`은 0으로 채워집니다.

따라서 0으로 채워진 `init_level4_pgt`을 가집니다. 마지막 `init_level4_pgt`엔트리의 커널 하이 매핑을 다음으로 설정합니다.

```C
init_level4_pgt[511] = early_top_pgt[511];
```

`reset_early_page_table`함수의 모든 `early_top_pgt`엔트리를 지우고 커널 하이 매핑을 유지한 것을 기억하십시오. 

`x86_64_start_kernel`함수의 마지막 단계는 다음을 호출합니다:

```C
x86_64_start_reservations(real_mode_data);
```

`real_mode_data`같은 인자와 함수. `x86_64_start_reservations`함수는 같은 소스 코드 파일`x86_64_start_kernel` 함수에 정의됬습니다. 봅시다:

```C
void __init x86_64_start_reservations(char *real_mode_data)
{
	if (!boot_params.hdr.version)
		copy_bootdata(__va(real_mode_data));

	reserve_ebda_region();

	start_kernel();
}
```

`start_kernel`함수가 커널 진입 포인트에 들어가기전 마지막 함수임을 알 수 있습니다. 그것이 무엇을 하고 어떻게 작동하는지 알아봅시다.

커널 진입 포인트 전 마지막 단계
--------------------------------------------------------------------------------

우선 `x86_64_start_reservations`함수에서 `boot_params.hdr.version`을 확인하십시오:

```C
if (!boot_params.hdr.version)
	copy_bootdata(__va(real_mode_data));
```

그리고 그것이 0이라면 `real_mode_data`(구현에 대한 읽기)의 가상주소에서 `copy_bootdata`함수를 다시 호출하십시오.

다음 단계는 [arch/x86/kernel/head.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head.c)에 정의된 `reserve_ebda_region`함수를 호출합니다. 이 함수는 `EBDA` 또는 확장된 BIOS 데이터 영역에 대한 메모리 블록을 예약합니다. 확장된 BIOS 데이터 영역은 기존 메모리 상단에 있으며 포트, 디스크 매개변수 등의 데이터를 포함합니다. 

`reserve_ebda_region`함수를 봅시다. 반가상화가 활성화됐는지 확인합니다:

```C
if (paravirt_enabled())
	return;
```
In the next step we need to get the end of the low memory:
반가상화가 활성화됐을 경우 활성화된 확장 bios 데이터 영역이 없기때문에 `reserve_ebda_region`함수에서 빠져나옵니다. 다음 단계에서 low 메모리의 끝을 찾아야합니다:

```C
lowmem = *(unsigned short *)__va(BIOS_LOWMEM_KILOBYTES);
lowmem <<= 10;
```

BIOS의 low 메모리의 가상 주소를 킬로바이트 단위로 가져오고 그것을 10개(즉, 1024를 곱하기)씩 바이트로 변환합니다. 그 다음, 다음과 같이 확장된 BIOS 데이터 주소를 가져옵니다:

```C
ebda_addr = get_bios_ebda();
```

`get_bios_ebda`함수는 [arch/x86/include/asm/bios_ebda.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bios_ebda.h)에 정의됐으며 다음과 같습니다:

```C
static inline unsigned int get_bios_ebda(void)
{
	unsigned int address = *(unsigned short *)phys_to_virt(0x40E);
	address <<= 4;
	return address;
}
```

어떻게 작동하는지 이해해봅시다. 여기서 확장된 BIOS 데이터 영역의 기본 주소를 포함하는 세그먼트인 `0x0040:0x000e`에서  물리주소 `0x40E`를 가상으로 변환하는 것을 볼 수 있습니다. 물리주소를 가상주소로 변환하는 `phys_to_virt`함수를 사용한다고 걱정하지 마십시오. 이전에는 같은 점에 대해 `__va`매크로를 사용했지만 `phys_to_virt`는 같습니다:

```C
static inline void *phys_to_virt(phys_addr_t address)
{
         return __va(address);
}
```

한 가지 차이점이 있습니다: `CONFIG_PHYS_ADDR_T_64BIT`에 의존해 인자`phys_addr_t`를 전달합니다:

```C
#ifdef CONFIG_PHYS_ADDR_T_64BIT
	typedef u64 phys_addr_t;
#else
	typedef u32 phys_addr_t;
#endif
```

이 구성 옵션은 `CONFIG_PHYS_ADDR_T_64BIT`에 의해 활성화됩니다. 확장된 BIOS 데이터 영역의 기본 주소를 저장하는 세그먼트의 가상주소를 얻고 4만큼 이동해 반환합니다. 그 다음 `ebda_addr`변수는 확장된 BIOS 데이터 영역의 기본주소를 포함합니다.

다음 단계는 확장된 BIOS 데이터 영역과 low 메모리의 주소가 `INSANE_CUTOFF`매크로 이상인지 확인합니다:

```C
if (ebda_addr < INSANE_CUTOFF)
	ebda_addr = LOWMEM_CAP;

if (lowmem < INSANE_CUTOFF)
	lowmem = LOWMEM_CAP;
```

이것은:

```C
#define INSANE_CUTOFF		0x20000U
```

또는 128킬로바이트 입니다. 마지막 단계에서 low 메모리의 낮은 부분과 확장된 bios 데이터 영역을 얻고 low메모리와 1메가바이트 사이의 확장된 bios 데이터를 위한 메모리 영역을 예약하는 `memblock_reserve`함수를 호출합니다.

```C
lowmem = min(lowmem, ebda_addr);
lowmem = min(lowmem, LOWMEM_CAP);
memblock_reserve(lowmem, 0x100000 - lowmem);
```

`memblock_reserve`함수는 [mm/block.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/block.c)에 정의되었고 두 매개변수를 가집니다:

* 기본 물리적 주소;
* 영역 크기

그리고 주어진 기본 주소와 크기의 메모리 영역을 예약합니다. `memblock_reserve`는 리눅스 커널 메모리 관리자 프레임워크 책의 첫번째 함수입니다. 곧 메모리 관리자를 자세히 살펴볼 것이고 지금은 구현을 보겠습니다.

리눅스 커널 메모리 관리자 프레임워크의 첫 터치
--------------------------------------------------------------------------------

이전 단락에서 `memblock_reserve`함수의 호출에서 멈췄고 앞서 말했듯이 메모리 관리자 프레임워크의 첫번째 함수입니다. 이것이 어떻게 작동하는지 이해해봅시다. `memblock_reserve`함수 호출:

```C
memblock_reserve_region(base, size, MAX_NUMNODES, 0);
```

함수에 4개의 매개변수를 전달합니다:

* 메모리 영역의 물리 기본 주소;
* 메모리 영역의 크기;
* numa 노드의 최대 개수;
* 플래그;

`memblock_reserve_region`본체의 시작부분에서 `memblock_type`구조체의 정의를 볼 수 있습니다:

```C
struct memblock_type *_rgn = &memblock.reserved;
```

메모리 블록의 형식을 제시합니다. 보십시오:

```C
struct memblock_type {
         unsigned long cnt;
         unsigned long max;
         phys_addr_t total_size;
         struct memblock_region *regions;
};
```

확장된 bios 데이터 영역을 위한 메모리 블록을 예약해야하므로 현재 메모리 영역의 형식은 다음과 같은 `memblock`구조체로 예약됩니다:

```C
struct memblock {
         bool bottom_up;
         phys_addr_t current_limit;
         struct memblock_type memory;
         struct memblock_type reserved;
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
         struct memblock_type physmem;
#endif
};
```

그리고 일반 메모리 블록을 설명합니다. `memblock.reserved`의 주소에 할당하면 `_rgn`가 초기화되는 것을 볼 수 있습니다. `memblock`은 전역변수입니다:

```C
struct memblock memblock __initdata_memblock = {
	.memory.regions		= memblock_memory_init_regions,
	.memory.cnt		= 1,
	.memory.max		= INIT_MEMBLOCK_REGIONS,
	.reserved.regions	= memblock_reserved_init_regions,
	.reserved.cnt		= 1,
	.reserved.max		= INIT_MEMBLOCK_REGIONS,
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
	.physmem.regions	= memblock_physmem_init_regions,
	.physmem.cnt		= 1,
	.physmem.max		= INIT_PHYSMEM_REGIONS,
#endif
	.bottom_up		= false,
	.current_limit		= MEMBLOCK_ALLOC_ANYWHERE,
};
```

이 변수에 대해 자세히 설명하지는 않지만 메모리 관리자 파트에서 모든 세부 정보를 볼 수 있습니다. `memblock`변수는 `__initdata_memblock`로 정의되는 것을 참고하십시오:

```C
#define __initdata_memblock __meminitdata
```

`__meminit_data`는:

```C
#define __meminitdata    __section(.meminit.data)
```

이것으로 모든 메모리 블록이 `.meminit.data`섹션에 있다고 결론 내릴 수 있습니다. `_rgn`을 정의한 후 `memblock_dbg`매크로의 정보를 출력합니다. `memblock=debug`을 커널 커맨드 라인에 전달함으로써 활성화할 수 있습니다.

디버깅 라인이 출력된 후 다음 함수가 호출됩니다:

```C
memblock_add_range(_rgn, base, size, nid, flags);
```

`.meminit.data`섹션에 새로운 메모리 블록 영역을 추가합니다. 따라서 `_rgn`을 초기화하지는 않지만 이것은 `&memblock.reserved`을 포함합니다. 전달된 `_rgn`을 확장된 BIOS 데이터 지역의 기본주소, 이 지역의 크기와 플래그로 채웁니다: 

```C
if (type->regions[0].size == 0) {
    WARN_ON(type->cnt != 1 || type->total_size);
    type->regions[0].base = base;
    type->regions[0].size = size;
    type->regions[0].flags = flags;
    memblock_set_region_node(&type->regions[0], nid);
    type->total_size = size;
    return 0;
}
```

지역을 채운 후 두 매개변수로 `memblock_set_region_node`함수를 호출할 수 있습니다:

* 채워진 메모리 지역의 주소;
* NUMA 노드 아이디.

여기서 지역은 `memblock_region`구조체로 표현됩니다:

```C
struct memblock_region {
    phys_addr_t base;
	phys_addr_t size;
	unsigned long flags;
#ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
    int nid;
#endif
};
```

NUMA 노드 아이디는 [include/linux/numa.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/numa.h)에 정의된 `MAX_NUMNODES`매크로에 의존합니다:

```C
#define MAX_NUMNODES    (1 << NODES_SHIFT)
```

`NODES_SHIFT`는 구성 매개변수 `CONFIG_NODES_SHIFT`에 의존하며 다음과 같이 정의됩니다:

```C
#ifdef CONFIG_NODES_SHIFT
  #define NODES_SHIFT     CONFIG_NODES_SHIFT
#else
  #define NODES_SHIFT     0
#endif
```

`memblick_set_region_node`함수는 `memblock_region`에서 주어진 값으로 `nid`필드를 채웁니다: 

```C
static inline void memblock_set_region_node(struct memblock_region *r, int nid)
{
         r->nid = nid;
}
```

그 다음 먼저 `.meminit.data`섹션의 확장된 bios 데이터 영역의 `memblock`를 예약합니다. `reserve_ebda_region`함수는 이 단계에서 작업을 완료하고 [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/head64.c)로 돌아갈 수 있습니다.

커널 진입 포인트 전의 모든 준비가 끝났습니다! 마지막으로 `x86_64_start_reservations`함수를 호출합니다:

```C
start_kernel()
```

함수는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)파일에 있습니다.

이것이 이 파트의 전부입니다.

결론
--------------------------------------------------------------------------------

리눅스 커널 내부에 대한 세 번째 파트의 끝입니다. 다음 파트에서는 커널 진입 포인트 - `start_kernel`함수의 첫 번째 초기화 단계를 볼 것입니다. 이것은 첫 번째 `init`프로세스가 실행되기 전 첫 번째 단계가 될 것입니다.

질문이나 제안사항이 있다면 코멘트를 남기거나 [트위터](https://twitter.com/0xAX)로 보내주십시오.

**영어는 모국어가 아니며 모든 불편한 점은 정말 죄송합니다. 실수를 발견하면 [linux-insides](https://github.com/0xAX/linux-insides)에서 수정 사항이 포함된 PR을 보내주십시오.**

링크
--------------------------------------------------------------------------------

* [BIOS 데이터 영역](http://stanislavs.org/helppc/bios_data_area.html)
* [PC의 확장된 BIOS 데이터 영역에는 무엇이 있습니까?](http://www.kryslix.com/nsfaq/Q.6.html)
* [이전 파트](https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-2.md)
