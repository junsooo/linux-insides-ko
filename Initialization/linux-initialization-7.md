커널 초기화. Part 7.
================================================================================

아키텍쳐 별 초기화의 끝, 거의...
================================================================================


이것은 Linux Kernel 초기화 프로세스의 일곱 번째 파트로 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c#L861)의 `setup_arch` 함수 내부를 다룹니다. 이전 [파트들](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization)에서 보았듯이, `setup_arch` 함수는 커널 코드/데이터/bss를위한 메모리 예약, [Desktop Management Interface](http://en.wikipedia.org/wiki/Desktop_Management_Interface), early dump of the [PCI](http://en.wikipedia.org/wiki/PCI) 의 초기 스캐닝, PCI의 덤프 초기화와 같은 많은 아키텍쳐 별(우리의 경우 [x86_64]) 초기화 과정을 수행합니다. 이전 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-6)를 읽었다면, 우리가 `setup_real_mode` 함수에서 끝났다는 것을 기억하실 것입니다. 다음 단계에서 [memblock](https://junsoolee.gitbook.io/linux-insides-ko/summary/mm/linux-mm-1)의 제한을 모든 매핑 된 페이지로 설정하면 [kernel/printk/printk.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/printk/printk.c)에서 `setup_log_buf` 함수의 호출을 볼 수 있습니다.

`setup_log_buf` 함수는 커널 순환 버퍼를 설정하고, 그 길이는 `CONFIG_LOG_BUF_SHIFT` 구성 옵션에 따라 다릅니다. `CONFIG_LOG_BUF_SHIFT`의 문서에서 읽을 수 있듯이 `12` 와 `21`사이에 있을 것입니다. 내부에서 버퍼는 char형 배열로 정의됩니다. :

```C
#define __LOG_BUF_LEN (1 << CONFIG_LOG_BUF_SHIFT)
static char __log_buf[__LOG_BUF_LEN] __aligned(LOG_ALIGN);
static char *log_buf = __log_buf;
```

이제 setup_log_buf 함수의 구현을 살펴 봅시다. 현재 버퍼가 비어 있는지 확인하고 (단지 설정만 했기 때문에 비어 있어야 함) 다른 설정은 초기 설정인지 확인합니다. 커널 로그 버퍼 설정이 초기가 아닌 경우, 우리는 `log_buf_add_cpu` 함수를 호출하여 모든 CPU의 버퍼 크기를 증가시킵니다. :

```C
if (log_buf != __log_buf)
    return;

if (!early && !new_log_buf_len)
    log_buf_add_cpu();
```

`setup_arch`에서 볼 수 있기 때문에 `log_buf_add_cpu` 함수는 알아보지 않고, `setup_log_buf`를 다음과 같이 호출합니다. :

```C
setup_log_buf(1);
```

여기서 `1`은 초기 설정임을 의미합니다. 다음 단계에서는 커널 로그 버퍼의 길이가 업데이트되는 `new_log_buf_len` 변수를 확인하고 `memblock_virt_alloc` 함수를 통해 버퍼에 새 공간을 할당하거나 리턴합니다.

커널 로그 버퍼가 준비되면 다음 함수는 'reserve_initrd'입니다. [커널 초기화. part 4.](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-4)에서 이미 `early_reserve_initrd` 함수를 호출했습니다. 이제 우리는 `init_mem_mapping` 함수에서 직접 메모리 매핑을 재구성함에 따라 [initrd](http://en.wikipedia.org/wiki/Initrd)를 직접 매핑 된 메모리로 옮겨야합니다. `reserve_initrd` 함수는 `initrd`의 베이스 주소와 종료 주소의 정의에서 시작하여 부트 로더가 `initrd`를 제공하는지도 확인합니다. 우리가 `early_reserve_initrd` 에서 본 것과 동일합니다. 그러나 `memblock_reserve` 함수를 호출하여 `memblock` 영역의 예약된 영역 대신에, 직접 메모리 영역의 매핑 된 크기를 가져 와서 'initrd'의 크기가 이 영역보다 크지 않은지 확인합니다. :

```C
mapped_size = memblock_mem_size(max_pfn_mapped);
if (ramdisk_size >= (mapped_size>>1))
    panic("initrd too large to handle, "
	      "disabling initrd (%lld needed, %lld available)\n",
	      ramdisk_size, mapped_size>>1);
```


여기서 우리는 `memblock_mem_size` 함수를 호출하고 `max_pfn_mapped`를 함수에 전달합니다. 여기서 max_pfn_mapped는 가장 높은 직접 매핑 된 페이지 프레임 번호를 포함합니다. `페이지 프레임 번호`가 무엇인지 기억하지 못할 수 있으니 간단히 설명합니다. 가상 주소의 첫 번째 12 비트는 실제 페이지 또는 페이지 프레임의 오프셋을 나타냅니다. 가상 주소의 `12` 비트를 오른쪽으로 쉬프트하면 오프셋 부분은 버리고 `Page Frame Number`를 얻게됩니다. `memblock_mem_size`에서 우리는 모든 memblock `mem`(예약되지 않음) 영역을 거쳐 매핑 된 페이지의 크기를 계산하여 `mapped_size` 변수로 반환합니다 (위 코드 참조). 직접 매핑 된 메모리의 크기를 얻었으므로 'initrd'의 크기가 매핑 된 페이지보다 크지 않은지 확인합니다. 더 큰 경우 시스템을 정지시키고 유명한 [Kernel panic](http://en.wikipedia.org/wiki/Kernel_panic) 메시지를 출력하는 `panic`을 호출합니다. 다음 단계에서는 'initrd' 크기에 대한 정보를 출력합니다. `dmesg` 출력에서 이 결과를 볼 수 있습니다. :

```C
[0.000000] RAMDISK: [mem 0x36d20000-0x37687fff]
```

그리고 `relocate_initrd` 함수를 통해 `initrd`를 직접 매핑 영역으로 재할당합니다. `relocate_initrd` 함수의 시작에서 우리는 `memblock_find_in_range` 함수를 이용하여 빈 영역을 찾습니다. :

```C
relocated_ramdisk = memblock_find_in_range(0, PFN_PHYS(max_pfn_mapped), area_size, PAGE_SIZE);

if (!relocated_ramdisk)
    panic("Cannot find place for new RAMDISK of size %lld\n",
	       ramdisk_size);
```

`memblock_find_in_range` 함수는 주어진 범위에서 사용 가능한 영역을 찾으려고 시도합니다. 이 경우 `0`부터 최대 매핑 된 물리적 주소와 크기까지는 `initrd`의 정렬 된 크기와 같아야합니다. 주어진 크기의 영역을 찾지 못하면 다시 `panic`을 호출합니다. 모든 것이 정상적이라면, 다음 단계에서 RAM 디스크를 직접 매핑 된 메모리의 밑으로 이동시키기 시작합니다.

`reserve_initrd` 함수의 끝에서, 우리는 다음을 호출하여 램 디스크가 차지했던 memblock 메모리를 해제합니다 :

```C
memblock_free(ramdisk_image, ramdisk_end - ramdisk_image);
```

램 디스크 이미지 `initrd` 를 재할당 한 후 다음 함수는 [arch/x86/kernel/vsmp_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vsmp_64.c)의 `vsmp_init`입니다. 이 함수는 'ScaleMP vSMP'의 서포트를 초기화합니다. 이전 부분에서 언급했듯이 이 챕터에서는 관련이없는 `x86_64` 초기화 부분 (예 : 지금의 부분 또는 `ACPI`등)을 다루지 않습니다. 따라서 지금은 이 구현을 건너 뛰고 병렬 컴퓨팅 기술을 다루는 부분으로 가겠습니다.

다음 함수는 [arch/x86/kernel/io_delay.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/io_delay.c)의 `io_delay_init`입니다. 이 함수를 사용하면 기본 I/O 지연 `0x80` 포트를 오버라이드할 수 있습니다. 우리는 이미 [보호 모드로 전환하기 전 마지막 준비](https://junsoolee.gitbook.io/linux-insides-ko/summary/booting/linux-bootstrap-3)에서 I/O 지연을 보았습니다. 이제 `io_delay_init` 구현을 살펴봅시다. :


```C
void __init io_delay_init(void)
{
    if (!io_delay_override)
        dmi_check_system(io_delay_0xed_port_dmi_table);
}
```

이 함수는 `io_delay_override` 변수를 확인하고 `io_delay_override`가 설정된 경우 I/O 지연 포트를 오버라이드합니다. 커널 명령 행에 `io_delay` 옵션을 전달하여 `io_delay_override`를 가변적으로 설정할 수 있습니다. [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)에서 읽을 수있는 `io_delay` 옵션은 다음과 같습니다. :

```
io_delay=	[X86] I/O delay method
    0x80
        Standard port 0x80 based delay
    0xed
        Alternate port 0xed based delay (needed on some systems)
    udelay
        Simple two microseconds delay
    none
        No delay
```

우리는 `io_delay` 커맨드라인 파라미터가 `early_param` 매크로를 설정한다는 것을 [arch/x86/kernel/io_delay.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/io_delay.c)에서 볼 수 있습니다.

```C
early_param("io_delay", io_delay_param);
```
`early_param`에 대한 자세한 내용은 이전 [파트]https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-6에서 읽을 수 있습니다. 이에 따라 [io_delay_override](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c#L413) 변수를 설정하는 `io_delay_param` 함수는 [do_early_param] 함수에서 호출됩니다. `io_delay_param` 함수는 `io_delay` 커널 명령 행 매개 변수의 인수를 가져오고 `io_delay_type`을 설정합니다 :

```C
static int __init io_delay_param(char *s)
{
        if (!s)
                return -EINVAL;

        if (!strcmp(s, "0x80"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_0X80;
        else if (!strcmp(s, "0xed"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_0XED;
        else if (!strcmp(s, "udelay"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_UDELAY;
        else if (!strcmp(s, "none"))
                io_delay_type = CONFIG_IO_DELAY_TYPE_NONE;
        else
                return -EINVAL;

        io_delay_override = 1;
        return 0;
}
```

다음 함수는 `io_delay_init` 후의 `acpi_boot_table_init`, `early_acpi_boot_init`, `initmem_init`입니다. 그러나 위에서 언급 한 것처럼 이 `Linux 커널 초기화 프로세스' 챕터에서 [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface) 관련 내용은 다루지 않습니다.

DMA를 위한 영역 할당
--------------------------------------------------------------------------------

다음 단계에서는 [drivers/base/dma-contiguous.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/base/dma-contiguous.c)에 정의 된 `dma_contiguous_reserve` 함수를 사용하여 [직접 메모리 액세스](http://en.wikipedia.org/wiki/Direct_memory_access)에 대한 영역을  할당해야합니다. `DMA`는 장치가 CPU없이 메모리와 통신 할 때의 특수한 모드입니다. 파라미터 하나 -  max_pfn_mapped << PAGE_SHIFT 를 `dma_contiguous_reserve` 함수에 전달한다는 것에 주목해봅시다. 이 표현에서 알 수 있듯이 이것은 예약 된 메모리의 리미트입니다. 이 함수의 구현을 살펴 봅시다. 다음 변수의 정의에서 시작합니다. :

```C
phys_addr_t selected_size = 0;
phys_addr_t selected_base = 0;
phys_addr_t selected_limit = limit;
bool fixed = false;
```

여기서 첫 번째는 예약 된 영역의 크기를 바이트 단위로 나타내고, 두 번째는 예약 된 영역의 기본 주소이고, 세 번째는 예약 된 영역의 종료 주소이며 마지막 'fixed' 매개 변수는 예약 된 영역을 배치 할 위치를 나타냅니다. `fixed`가 `1`이면 `memblock_reserve`로 영역을 예약하고,`0`이면 `kmemleak_alloc`으로 영역을 할당합니다. 다음 단계에서 우리는 `size_cmdline` 변수를 확인하고 그것이 '-1'이 아닌 경우 위에서 볼 수있는 모든 변수를 `cma` 커널 커맨들 라인 매개 변수의 값으로 채웁니다 :

```C
if (size_cmdline != -1) {
   ...
   ...
   ...
}
```

이 소스 코드 파일에서 초기 매개 변수의 정의를 찾을 수 있습니다. :

```C
early_param("cma", early_cma);
```

`cma` :

```
cma=nn[MG]@[start[MG][-end[MG]]]
		[ARM,X86,KNL]
		Sets the size of kernel global memory area for
		contiguous memory allocations and optionally the
		placement constraint by the physical address range of
		memory allocations. A value of 0 disables CMA
		altogether. For more information, see
		include/linux/dma-contiguous.h
```

`cma` 옵션을 커널 명령 행에 전달하지 않으면 `size_cmdline`은 `-1`과 같습니다. 이런 식으로 다음 커널 구성 옵션에 따라 예약 영역의 크기를 계산해야합니다. :

* `CONFIG_CMA_SIZE_SEL_MBYTES` - `CMA_SIZE_MBYTES * SZ_1M` or `CONFIG_CMA_SIZE_MBYTES * 1M`와 동일한 메가바이트의 크기와 디폴트 전역 `CMA` 영역.
* `CONFIG_CMA_SIZE_SEL_PERCENTAGE` - 전체 메모리의 비율;
* `CONFIG_CMA_SIZE_SEL_MIN` - 더 낮은 값 사용;
* `CONFIG_CMA_SIZE_SEL_MAX` - 더 높은 값 사용.

예약 영역의 크기를 계산할 때, 먼저 다음의 함수를 호출하는 `dma_contiguous_reserve_area` 함수를 호출하여 영역을 예약합니다. :

```
ret = cma_declare_contiguous(base, size, limit, 0, 0, fixed, res_cma);
```

`cma_declare_contiguous`는 주어진 베이스 주소에서 주어진 크기를 이용해 연속 된 영역을 예약합니다. DMA를 위한 영역을 예약한 다음 함수는 `memblock_find_dma_reserve` 입니다. 이름에서 알 수 있듯이 이 함수는 DMA 영역에서 예약 된 페이지를 계산합니다. 지금 파트는 CMA 및 DMA에 대한 모든 세부 사항을 다루지는 않습니다. Linux 커널 메모리 관리안에 있는 특수한 파트에서 인접 메모리 할당자와 그 영역을 다루는 훨씬 더 자세한 내용을 볼 수 있습니다.


sparce 메모리의 초기화
--------------------------------------------------------------------------------

다음 단계는 `x86_init.paging.pagetable_init` 함수 호출입니다. Linux 커널 소스 코드에서 이 기능을 찾고자한다면, 검색이 끝날 때 다음 매크로를 볼 수 있습니다. :

The next step is the call of the function - `x86_init.paging.pagetable_init`. If you try to find this function in the linux kernel source code, in the end of your search, you will see the following macro:

```C
#define native_pagetable_init        paging_init
```

이것은 [arch/x86/mm/init_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/init_64.c)에서 `paging_init` 함수의 호출을 볼 수 있듯이 확장됩니다. `paging_init` 함수는 스파스 메모리와 존 크기를 초기화합니다. 존은 무엇이고 무엇이 `Sparsemem`은 무엇일까요? `Sparsemem`은 [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access) 시스템에서 메모리 영역을 다른 메모리 뱅크로 분할하는 데 사용되는 Linux 커널 메모리 관리자의 특수한 기반입니다. `paginig_init` 함수의 구현을 살펴봅시다. :

```C
void __init paging_init(void)
{
        sparse_memory_present_with_active_regions(MAX_NUMNODES);
        sparse_init();

        node_clear_state(0, N_MEMORY);
        if (N_MEMORY != N_NORMAL_MEMORY)
                node_clear_state(0, N_NORMAL_MEMORY);

        zone_sizes_init();
}
```

보다시피 `struct page` 배열의 구조체에 대한 포인터를 포함하는 `mem_section` 구조체 배열에 대한 모든 'NUMA'노드의 메모리 영역을 기록하는 `sparse_memory_present_with_active_regions` 함수의 호출이 있습니다. 다음 `sparse_init` 함수는 비선형 `mem_section`와 `mem_map`을 할당합니다. 다음 단계에서 이동식 메모리 노드의 상태를 지우고 영역의 크기를 초기화합니다. 모든 `NUMA` 노드는 'zones'라고 불리는 여러 조각으로 나뉩니다. 따라서 [arch/x86/mm/init.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/init.c)의 `zone_sizes_init` 함수는 존의 크기를 초기화합니다.

지금 파트와 다음 파트에서는 이 주제를 자세히 다루지 않습니다. NUMA에 대한 특별한 파트는 있을 것입니다.

vsyscall 매핑
--------------------------------------------------------------------------------

`SparseMem` 초기화 이후에 다음 단계는 `cr4` [Control register] (http://en.wikipedia.org/wiki/Control_register)의 내용을 포함해야하는 `trampoline_cr4_features`의 설정입니다. 우선 우리는 현재 CPU가 `cr4` 레지스터를 지원하는지 확인해야하고 만약 있다면, 리얼 모드에서 `cr4`를 저장하는 `trampoline_cr4_features`에 내용을 저장합니다 :

```C
if (boot_cpu_data.cpuid_level >= 0) {
    mmu_cr4_features = __read_cr4();
	if (trampoline_cr4_features)
	    *trampoline_cr4_features = mmu_cr4_features;
}
```

다음으로 살펴 볼 함수는 [arch/x86/kernel/vsyscall_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vsyscall_64.c)의 `map_vsyscal`입니다. 이 함수는 [vsyscalls](https://lwn.net/Articles/446528/)의 메모리 공간을 매핑하며 `CONFIG_X86_VSYSCALL_EMULATION` 커널 구성 옵션에 따라 다릅니다. 실제로 `vsyscall`은 `getcpu`와 같은 특정 시스템 호출에 빠르게 액세스 할 수있는 특수 세그먼트입니다. 이 함수의 구현을 살펴 보겠습니다. :

```C
void __init map_vsyscall(void)
{
        extern char __vsyscall_page;
        unsigned long physaddr_vsyscall = __pa_symbol(&__vsyscall_page);

        if (vsyscall_mode != NONE)
                __set_fixmap(VSYSCALL_PAGE, physaddr_vsyscall,
                             vsyscall_mode == NATIVE
                             ? PAGE_KERNEL_VSYSCALL
                             : PAGE_KERNEL_VVAR);

        BUILD_BUG_ON((unsigned long)__fix_to_virt(VSYSCALL_PAGE) !=
                     (unsigned long)VSYSCALL_ADDR);
}
```

`map_vsyscall `의 시작 부분에서 우리는 두 변수의 정의를 볼 수 있습니다. 첫 번째는 extern 변수인 `__vsyscall_page`입니다. extern 변수로서 다른 소스 코드 파일 어딘가에 정의되었습니다. 실제로 우리는 [arch/x86/kernel/vsyscall_emu_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vsyscall_emu_64.S)에서 `__vsyscall_page`의 정의를 볼 수 있습니다. `__vsyscall_page` 심볼은 `vsyscalls`의 정렬 된 호출을 `gettimeofday` 등으로 가리 킵니다. :

```assembly
	.globl __vsyscall_page
	.balign PAGE_SIZE, 0xcc
	.type __vsyscall_page, @object
__vsyscall_page:

	mov $__NR_gettimeofday, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_time, %rax
	syscall
	ret
    ...
    ...
    ...
```

두 번째 변수는 `physaddr_vsyscall`이며 `__vsyscall_page` 심볼의 물리적 주소만 저장합니다. 다음 단계에서 우리는 `vsyscall_mode` 변수를 확인하고, 이때 `NONE`이 아니라면 기본적으로 `EMULATE`입니다. :

```C
static enum { EMULATE, NATIVE, NONE } vsyscall_mode = EMULATE;
```

그리고 이 확인 후에 동일한 매개 변수로 `native_set_fixmap`을 호출하는 `__set_fixmap` 함수의 호출을 볼 수 있습니다.

```C
void native_set_fixmap(enum fixed_addresses idx, unsigned long phys, pgprot_t flags)
{
        __native_set_fixmap(idx, pfn_pte(phys >> PAGE_SHIFT, flags));
}

void __native_set_fixmap(enum fixed_addresses idx, pte_t pte)
{
        unsigned long address = __fix_to_virt(idx);

        if (idx >= __end_of_fixed_addresses) {
                BUG();
                return;
        }
        set_pte_vaddr(address, pte);
        fixmaps_set++;
}
```

여기서 우리는 `native_set_fixmap`가 주어진 물리적 주소(여기서는 `__vsyscall_page` 심볼의 물리적 주소)에서 `Page Table Entry`의 값을 만들고, 내부 함수인 `__native_set_fixmap`을 호출한다는 것을 알 수 있습니다. 내부 함수는 주어진 `fixed_addresses` 인덱스 (여기서는 `VSYSCALL_PAGE`)의 가상 주소를 가져오고 주어진 인덱스가 수정-매핑 된 주소의 끝보다 크지 않은지 확인합니다. 그런 다음 `set_pte_vaddr` 함수를 호출하여 페이지 테이블 항목을 설정하고 수정 매핑 된 주소의 수를 늘립니다. 그리고 `map_vsyscall`의 끝에서 우리는 `VSYSCALL_PAGE`의 가상 주소 (`fixed_addresses`의 첫 번째 인덱스)가 `-10UL << 20` 또는 `ffffffffff600000`인 VSYSCALL_ADDR보다 크지 않은지 `BUILD_BUG_ON` 매크로로 확인합니다. :

```C
BUILD_BUG_ON((unsigned long)__fix_to_virt(VSYSCALL_PAGE) !=
                     (unsigned long)VSYSCALL_ADDR);
```

이제 `vsyscall`영역은 `fix-mapped`영역에 위치합니다. 수정 된 주소에 대해 아무것도 모르는 경우 [Fix-Mapped Addresses and ioremap] (https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-2.html)에서 읽을 수 있습니다. 우리는 `vsyscalls 와 vdso` 부분에서 `vsyscalls`에 대해 더 많이 볼 것입니다.

SMP의 설정 얻기
--------------------------------------------------------------------------------

이전 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-6)에서 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing) 설정을 검색 하는 방법을 알았습니다. 이제 찾은 `SMP`의 설정이 필요합니다. 이를 위해 우리는 `smp_scan_config` 함수에서 설정 한 `smp_found_config` 변수를 검사하고(이전 파트 참고) `get_smp_config` 함수를 호출합니다 :

```C
if (smp_found_config)
	get_smp_config();
```

`get_smp_config`는 [arch/x86/kernel/mpparse.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/mpparse.c)에 정의 된`x86_init.mpparse.default_get_smp_config` 함수로 확장됩니다. 이 함수는 멀티 프로세서 플로팅 포인터 구조체인 `mpf_intel`에 대한 포인터를 정의하고(이전 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-6)에서 읽을 수 있음) 몇 가지 검사를 수행합니다. :

```C
struct mpf_intel *mpf = mpf_found;

if (!mpf)
    return;

if (acpi_lapic && early)
   return;
```

여기서 우리는 멀티 프로세서 설정이 `smp_scan_config` 함수에서 발견되거나, 그렇지 않은 경우 그 함수의 리턴에서 볼 수 있습니다. 다음으로 확인할 것은 `acpi_lapic`과 `early`입니다. 그리고 이 검사를 수행하면서 `SMP` 설정을 읽기 시작합니다. 읽은 후, 다음 단계에서 `prefill_possible_map` 함수로 CPU의 `cpumask '를 미리 채웁니다(자세한 내용은 [cpumasks 소개](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-2)에서 읽을 수 있습니다).


setup_arch의 나머지
--------------------------------------------------------------------------------

여기서 우리는 `setup_arch` 함수의 끝을 얻었습니다. 나머지 함수도 물론 중요하지만 이러한 내용에 대한 자세한 내용은 지금 파트에 포함되지 않습니다. 위에서 언급 한대로 중요하긴하지만 `NUMA`, `SMP`, `ACPI`, `APIC` 등과 관련된 제네릭이 아닌 커널 함수들이기 때문에, 이러한 함수에 대해서는 간단히만 살펴 보겠습니다. 우선, 다음에 `init_apic_mappings` 함수를 호출합니다. 우리가 알고 있듯이 이 함수는 로컬 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)의 주소를 설정합니다. 다음 함수는 `x86_io_apic_ops.init`이며, 이 함수는 I/O APIC를 초기화합니다.인터럽트 및 예외 처리에 관한 챕터에서 APIC와 관련된 모든 세부 사항을 볼 수 있습니다. 다음 단계에서는 `x86_init.resources.reserve_resources` 함수를 호출하여 `DMA`, `TIMER`, `FPU` 등과 같은 표준 I/O 리소스를 예약합니다. 다음은  `Machine check Exception`을 초기화하는 `mcheck_init` 함수이고, 마지막함수는 [jiffy](http://en.wikipedia.org/wiki/Jiffy_%28time%29)를 등록하는`register_refined_jiffies`입니다 (커널의 타이머에 대한 별도의 챕터가 있습니다).

이것이 다입니다. 마지막으로 이 파트에서 큰 `setup_arch` 함수를 마쳤습니다. 물론 이미 여러 번 쓴 것처럼 이 함수에 대한 자세한 내용은 보지 못했지만 걱정하지 않아도됩니다. 플랫폼에 따라 다른 파트가 초기화되는 방법을 이해하기 위해 다른 챕터에서 이 함수로 두 번 이상 돌아올 것입니다.

이것으로 끝났습니다. 이제 우리는`setup_arch에서 `start_kernel`로 돌아갈 수 있습니다. :

main.c로 돌아가서
================================================================================

위에서 쓴 것처럼, 우리는`setup_arch` 함수로 끝났고 이제 [init/main.c]의 `start_kernel` 함수로 돌아갈 수 있습니다. `start_kernel`은 `setup_arch`만큼 큰 기능을 한다는 것을 기억할 것입니다. 다음 파트에서는 이 함수를 배우는 데 전념 할 것입니다. `setup_arch` 이후에 `mm_init_cpumask` 함수의 호출을 볼 수 있습니다. 이 함수는 [cpumask](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-2) 포인터를 메모리 디스크립터 `cpumask`로 설정합니다. 함수의 구현에서 살펴볼 수 있습니다. :

```C
static inline void mm_init_cpumask(struct mm_struct *mm)
{
#ifdef CONFIG_CPUMASK_OFFSTACK
        mm->cpu_vm_mask_var = &mm->cpumask_allocation;
#endif
        cpumask_clear(mm->cpu_vm_mask_var);
}
```

[init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에서 볼 수 있듯이 init 프로세스의 메모리 디스크립터를 `mm_init_cpumask`에 전달합니다. `CONFIG_CPUMASK_OFFSTACK` 설정 옵션에 따라 [TLB](http://en.wikipedia.org/wiki/Translation_lookaside_buffer) 스위치 `cpumask`를 지웁니다.

다음 단계에서 다음 함수의 호출을 볼 수 있습니다. :

```C
setup_command_line(command_line);
```

이 함수는 커널 커맨드 라인에 대한 포인터를 가져 와서 커맨드 라인을 저장하기 위한 몇 개의 버퍼를 할당합니다. 버퍼 하나는 나중에 참조하고 커맨드 라인에 엑세스하는 데에 사용되고, 하나는 매개 변수 파싱에 사용되므로 두 개의 버퍼가 필요합니다. 다음 버퍼를 위한 공간을 할당합니다. :

* `saved_command_line` - 부팅 커맨드 라인을 포함할 것.
* `initcall_command_line` - 부팅 커맨드 라인을 포함할 것. `do_initcall_level`에서 사용 될 것.;
* `static_command_line` - 매개변수 파싱을 위한 커맨드 라인을 포함.

`memblock_virt_alloc` 함수를 사용하여 공간을 할당합니다. 이 함수는 [slab](http://en.wikipedia.org/wiki/Slab_allocation)을 사용할 수 없거나 `kzalloc_node`를 사용하는 경우 `memblock_reserve`로 부팅 메모리 블록을 할당하는 `memblock_virt_alloc_try_nid`를 호출합니다 (자세한 내용은 Linux 메모리 관리 챕터에 있음). `memblock_virt_alloc`은 `BOOTMEM_LOW_LIMIT`(`(PAGE_OFFSET + 0x1000000)`의 물리적 주소) 및 `BOOTMEM_ALLOC_ACCESSIBLE '(`memblock.current_limit`의 현재 값과 동일)을 각각 메모리 영역의 최소 주소 및 최대 주소로 사용합니다.

`setup_command_line`의 구현을 살펴 봅시다. :

```C
static void __init setup_command_line(char *command_line)
{
        saved_command_line =
                memblock_virt_alloc(strlen(boot_command_line) + 1, 0);
        initcall_command_line =
                memblock_virt_alloc(strlen(boot_command_line) + 1, 0);
        static_command_line = memblock_virt_alloc(strlen(command_line) + 1, 0);
        strcpy(saved_command_line, boot_command_line);
        strcpy(static_command_line, command_line);
 }
 ```

여기서 우리는 다른 목적을 위해 커널 명령 행을 포함할 3 개의 버퍼를 위한 공간을 할당한다는 것을 알 수 있습니다(위에서 읽음). 그리고 공간을 할당 할 때 `boot_command_line`을 `saved_command_line` 및 `command_line` (`setup_arch`의 커널 명령 줄)에`static_command_line`에 저장합니다.

`setup_command_line` 다음에 오는 함수는 `setup_nr_cpu_ids`입니다. 이 함수 설정은 `cpu_possible_mask`의 마지막 비트에 따라 'nr_cpu_ids'(CPU 수)를 설정합니다 (자세한 내용은 [cpumasks](https://junsoolee.gitbook.io/linux-insides-ko/summary/concepts/linux-cpu-2) 개념에서 설명합니다). 구현에 대해 살펴 봅시다. :

```C
void __init setup_nr_cpu_ids(void)
{
        nr_cpu_ids = find_last_bit(cpumask_bits(cpu_possible_mask),NR_CPUS) + 1;
}
```

CPU들의 수를 나타내는 여기 `nr_cpu_ids`에서,`NR_CPUS`는 설정 시간을 설정할 수 있는 CPU들의 최대 수를 나타냅니다. :

![CONFIG_NR_CPUS](http://oi59.tinypic.com/28mh45h.jpg)

NR_CPUS는 컴퓨터의 실제 CPU 수보다 클 수 있으므로 실제로 이 함수를 호출해야합니다. 여기서 우리는 `find_last_bit` 함수를 호출하고 두 개의 매개 변수를 전달하는 것을 볼 수 있습니다 :

* `cpu_possible_mask` bits;
* maximum number of CPUS.

`setup_arch`에서 CPU의 실제 수인 `cpu_possible_mask`를 계산하고 기록하는 `prefill_possible_map` 함수의 호출을 찾을 수 있습니다. 주소와 최대 크기를 검색하여 첫 번째 설정 비트의 비트 번호를 리턴하는 `find_last_bit` 함수를 호출합니다. 우리는 `cpu_possible_mask` 비트와 최대 CPU 수를 전달했습니다. 먼저 `find_last_bit` 함수는 `signed long`형 주소가 주어지면 [워드](http://en.wikipedia.org/wiki/Word_%28computer_architecture%29)로 분할됩니다. :

```C
words = size / BITS_PER_LONG;
```

여기서 `BITS_PER_LONG`은 `x86_64`에서 `64`입니다. 주어진 사이즈의 검색 데이터 안에 워드의 수가 있으므로, 크기가 부분 단어를 포함하지는 않는지 다음을 이용하여 확인해야합니다. :

```C
if (size & (BITS_PER_LONG-1)) {
         tmp = (addr[words] & (~0UL >> (BITS_PER_LONG
                                 - (size & (BITS_PER_LONG-1)))));
         if (tmp)
                 goto found;
}
```

부분 단어가 포함 된 경우 마지막 워드를 마스크하고 확인합니다. 마지막 워드가 0이 아니면 현재 단어에 하나 이상의 설정 비트가 포함되어 있음을 의미합니다. 이제 `found` 레이블로 이동합니다. :


```C
found:
    return words * BITS_PER_LONG + __fls(tmp);
```

다음은 `bsr` 명령어의 도움으로 주어진 워드의 마지막 설정 비트를 리턴하는 `__fls` 함수를 볼 수 있습니다 :

```C
static inline unsigned long __fls(unsigned long word)
{
        asm("bsr %1,%0"
            : "=r" (word)
            : "rm" (word));
        return word;
}
```

주어진 피연산자를 스캔하여 첫 번째 비트 세트를 찾는 `bsr` 명령어. 마지막 워드가 부분적이지 않으면 주어진 주소의 모든 워드를 살펴보고 첫 번째 설정 비트를 찾습니다. :

```C
while (words) {
    tmp = addr[--words];
    if (tmp) {
found:
        return words * BITS_PER_LONG + __fls(tmp);
    }
}
```

여기서 마지막 단어를 `tmp` 변수에 넣고 `tmp`에 적어도 하나의 설정 비트가 포함되어 있는지 확인합니다. 설정된 비트가 발견되면 이 비트 수를 리턴합니다. 한 워드에 설정 비트가 포함되어 있지 않으면 주어진 크기를 반환합니다. :

```C
return size;
```

이 후 `nr_cpu_ids`는 사용 가능한 CPU의 정확한 수를 포함하게 됩니다.

끝났습니다.

결론
================================================================================

리눅스 커널 초기화 과정에 대한 일곱 번째 부분의 끝입니다. 이 부분에서 마지막으로 `setup_arch` 함수를 끝냈고 `start_kernel` 함수로 돌아 왔습니다. 다음 부분에서는 `start_kernel`에서 일반 커널 코드를 계속 배우고 첫 번째 `init` 프로세스로 계속 진행할 것입니다.

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견이나 핑을 남겨주세요.

**영어는 제 모국어가 아닙니다. 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수를 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한글 번역](https://github.com/junsooo/linux-insides-ko)으로 PR을 보내주세요.**

Links
================================================================================

* [Desktop Management Interface](http://en.wikipedia.org/wiki/Desktop_Management_Interface)
* [x86_64](http://en.wikipedia.org/wiki/X86-64)
* [initrd](http://en.wikipedia.org/wiki/Initrd)
* [Kernel panic](http://en.wikipedia.org/wiki/Kernel_panic)
* [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)
* [ACPI](http://en.wikipedia.org/wiki/Advanced_Configuration_and_Power_Interface)
* [Direct memory access](http://en.wikipedia.org/wiki/Direct_memory_access)
* [NUMA](http://en.wikipedia.org/wiki/Non-uniform_memory_access)
* [Control register](http://en.wikipedia.org/wiki/Control_register)
* [vsyscalls](https://lwn.net/Articles/446528/)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [jiffy](http://en.wikipedia.org/wiki/Jiffy_%28time%29)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-6.html)
