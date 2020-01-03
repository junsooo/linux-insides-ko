커널 부팅 과정. Part 6.
================================================================================

시작
--------------------------------------------------------------------------------

커널 부팅 프로세스 시리즈의 여섯 번째 부분입니다. [이전 부분](linux-bootstrap-5.md)에서 커널 부팅 프로세스의 끝을 보았습니다. 그러나 중요한 고급 부분을 건너 뛰었습니다.

리눅스 커널의 엔트리 포인트는 LOAD_PHYSICAL_ADDR 주소로 실행되기 시작하는 소스코드 파일 [main.c](https://github.com/torvalds/linux/blob/v4.16/init/main.c) 의 `start_kernel` 함수입니다. 소스 코드 파일이 LOAD_PHYSICAL_ADDR 주소에서 실행되기 시작했습니다. 이 주소는 디폴트가 `0x1000000` 인 커널 구성 옵션 `CONFIG_PHYSICAL_START` 에 따라 다릅니다.:

```
config PHYSICAL_START
	hex "Physical address where the kernel is loaded" if (EXPERT || CRASH_DUMP)
	default "0x1000000"
	---help---
	  This gives the physical address where the kernel is loaded.
      ...
      ...
      ...
```

이 값은 커널 구성 중에 변경 될 수 있으며, 로드될 주소는 랜덤한 값으로 선택할 수 있습니다. 이를 위해 커널 구성 중에 커널 구성 옵션 `CONFIG_RANDOMIZE_BASE` 을 활성화해야합니다.

이 경우 Linux 커널 이미지를 압축 해제하고 로드할 실제 주소는 랜덤으로 지정됩니다. 이 부분에서는 이 옵션이 활성화되어있고 커널 이미지의 로드 주소가 [보안상의 이유로](https://en.wikipedia.org/wiki/Address_space_layout_randomization) 랜덤한 경우를 고려합니다.

페이지 테이블의 초기화
--------------------------------------------------------------------------------

커널 압축 해제 프로그램이 커널을 압축 해제하고 로드할 랜덤한 메모리 범위를 찾기 시작하기 전에 아이디 매핑 페이지 테이블을 초기화해야합니다. 만약 [부트로더](https://ko.wikipedia.org/wiki/부팅)가 [16 비트 또는 32 비트 부트 프로토콜](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt)을 사용한다면, 우리에게는 이미 페이지 테이블이 있습니다. 그러나 커널 압축 해제 프로그램이 메모리 범위 밖에서 메모리 범위를 선택하는 경우 필요에 따라 새 페이지가 필요할 수 있습니다. 그렇기 때문에 ID 매핑 페이지 테이블을 새로 만들어야합니다.

그렇습니다, ID 매핑 된 페이지 테이블을 작성하는 것은 로드 주소를 랜덤화하는 첫 번째 단계 중 하나입니다. 그러나 우리가 그것을 고려하기 전에, 우리가 어디에서 왔는지 기억해 봅시다.

우리는 [이전 부분](linux-bootstrap-5.md)에서 [롱 모드](https://en.wikipedia.org/wiki/Long_mode)을 보았고, 커널 압축 해제의 엔트리 포인트인 `extract_kernel` 함수로 이동합니다. 랜덤화는 다음 호출 함수:

```C
void choose_random_location(unsigned long input,
                            unsigned long input_size,
                            unsigned long *output,
                            unsigned long output_size,
                            unsigned long *virt_addr)
{}
```

에서 시작합니다. 보시다시피, 이 기능에는 다음과 같은 5 가지 매개 변수가 사용됩니다.

  * `input`;
  * `input_size`;
  * `output`;
  * `output_isze`;
  * `virt_addr`.

이 매개 변수가 무엇인지 이해해봅시다. 첫 번째 `input` 매개 변수는 소스 코드 파일 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/v4.16/arch)의 `extract_kernel` 함수의 매개 변수에서 가져왔습니다. :

```C
asmlinkage __visible void *extract_kernel(void *rmode, memptr heap,
				                          unsigned char *input_data,
				                          unsigned long input_len,
				                          unsigned char *output,
				                          unsigned long output_len)
{
  ...
  ...
  ...
  choose_random_location((unsigned long)input_data, input_len,
                         (unsigned long *)&output,
				         max(output_len, kernel_total_size),
				         &virt_addr);
  ...
  ...
  ...
}
```

이 매개 변수는 어셈블러 코드에서 전달됩니다. :

```C
leaq	input_data(%rip), %rdx
```

[arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)에서 `input_data`는 작은 [mkpiggy](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/mkpiggy.c) 프로그램에 의해 생성됩니다. 리눅스 커널 소스 코드를 컴파일했다면 `linux/arch/x86/boot/compressed/piggy.S`에 있는 이 프로그램에 의해 생성 된 파일을 찾을 수 있습니다. 필자의 경우 이 파일은 다음과 같습니다.:


```assembly
.section ".rodata..compressed","a",@progbits
.globl z_input_len
z_input_len = 6988196
.globl z_output_len
z_output_len = 29207032
.globl input_data, input_data_end
input_data:
.incbin "arch/x86/boot/compressed/vmlinux.bin.gz"
input_data_end:
```

보시다시피 4 개의 전역 기호가 포함되어 있습니다. 첫번째와 두번째는 `vmlinux.bin.gz`의 압축된 크기인 `z_input_len` 와 압축되지 않은 크기인 `z_output_len` 이고, 세번째는 `input_data` 이며, 알 수 있듯이 raw binary 형식의 Linux 커널 이미지를 가리킵니다(모든 디버깅 기호, 주석 및 재배치 정보가 제거됨). 마지막은 `input_data_end` 이며, 압축된 리눅스 이미지의 끝을 가리킵니다.

따라서 'choose_random_location' 함수의 첫 번째 매개 변수는 `piggy.o` 오브젝트 파일에 임베드 된 압축 커널 이미지에 대한 포인터입니다.

`choose_random_location` 함수의 두 번째 매개 변수는 우리가 지금 본 `z_input_len`입니다.

`choose_random_location` 함수의 세 번째 및 네 번째 매개 변수는 각각 압축 해제 된 커널 이미지를 배치할 위치와 압축 해제 된 커널 이미지의 길이입니다. 압축 해제 된 커널을 넣을 주소는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) 에서 나왔으며, 2MB 경계에 정렬 된 `startup_32`의 주소입니다. 압축 해제 된 커널의 크기는 동일한 `piggy.S` 에서 왔으며 `z_output_len` 입니다.

'choose_random_location' 함수의 마지막 매개 변수는 커널 로드 주소의 가상 주소입니다. 보시다시피 기본적으로 기본 실제 로드 주소와 일치합니다. :

```C
unsigned long virt_addr = LOAD_PHYSICAL_ADDR;
```

이는 커널 특성에 따라 다릅니다. :

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

이제 `choose_random_location` 함수의 매개 변수를 이해했으므로 구현부를 살펴 보겠습니다. 이 함수는 커널 명령 행에서 `nokaslr` 옵션을 체크하는 것으로 시작합니다. :

```C
if (cmdline_find_option_bool("nokaslr")) {
	warn("KASLR disabled: 'nokaslr' on cmdline.");
	return;
}
```

그리고 옵션이 주어지면 우리는 `choose_random_location` 함수에서 나가고 커널 로드 주소는 랜덤화 되지 않을 것입니다. 관련 명령 행 옵션은 [커널 문서](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst)에서 찾을 수 있습니다. :

```
kaslr/nokaslr [X86]

Enable/disable kernel and module base offset ASLR
(Address Space Layout Randomization) if built into
the kernel. When CONFIG_HIBERNATION is selected,
kASLR is disabled by default. When kASLR is enabled,
hibernation will be disabled.
```

`nokaslr`을 커널 명령 행에 전달하지 않고 `CONFIG_RANDOMIZE_BASE` 커널 구성 옵션이 활성화되었다고 가정해 봅시다. 이 경우 커널로드 플래그에 `kASLR` 플래그를 추가합니다.

```C
boot_params->hdr.loadflags |= KASLR_FLAG;
```

그리고 다음 단계에서 호출 되는 함수:

```C
initialize_identity_maps();
```

는 [arch/x86/boot/compressed/kaslr_64.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr_64.c) 소스코드 파일에 정의되어 있습니다. 이 함수는`x86_mapping_info` 구조체의 인스턴스인 `mapping_info`의 초기화에서 시작합니다.

```C
mapping_info.alloc_pgt_page = alloc_pgt_page;
mapping_info.context = &pgt_data;
mapping_info.page_flag = __PAGE_KERNEL_LARGE_EXEC | sev_me_mask;
mapping_info.kernpg_flag = _KERNPG_TABLE;
```

`x86_mapping_info` 구조는 헤더파일 [arch/x86/include/asm/init.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/init.h) 에 정의되어 있습니다. :

```C
struct x86_mapping_info {
	void *(*alloc_pgt_page)(void *);
	void *context;
	unsigned long page_flag;
	unsigned long offset;
	bool direct_gbpages;
	unsigned long kernpg_flag;
};
```

이 구조는 메모리 매핑에 대한 정보를 제공합니다. 이전 부분에서 초기 페이지 테이블울 0에서 4G까지 설정했던 것을 기억하실 것 입니다. 현재 커널을 임의의 위치에 로드하기 위해 `4G` 이상의 메모리에 액세스해야 할 수도 있습니다. 따라서 `initialize_identity_maps` 함수는 필요한 새 페이지 테이블에 대해 메모리 영역의 초기화를 실행합니다. 우선 `x86_mapping_info` 구조의 정의를 살펴 봅시다.

`alloc_pgt_page`는 페이지 테이블 엔트리를 위한 공간을 할당하기 위해 호출되는 콜백 함수입니다. `context` 필드는 할당 된 페이지 테이블을 추적하는 데 사용될 우리의 경우 `alloc_pgt_data` 구조의 인스턴스입니다. `page_flag` 및 `kernpg_flag` 필드는 페이지 플래그입니다. 첫 번째는 'PMD'또는 'PUD'항목에 대한 플래그를 나타냅니다. 두 번째 'kernpg_flag' 필드는 나중에 무시할 수있는 커널 페이지에 대한 플래그를 나타냅니다. `direct_gbpages` 필드는 거대한 페이지에 대한 지원을 나타내고, 마지막 'offset'필드는 커널 가상 주소와 실제 주소 사이의 최대 PMD 수준 오프셋을 나타냅니다.

`alloc_pgt_page` 콜백은 새로운 페이지를 위한 공간이 있는지 확인하고 새로운 페이지를 할당합니다. :

```C
entry = pages->pgt_buf + pages->pgt_buf_offset;
pages->pgt_buf_offset += PAGE_SIZE;
```

버퍼에서 :

```C
struct alloc_pgt_data {
	unsigned char *pgt_buf;
	unsigned long pgt_buf_size;
	unsigned long pgt_buf_offset;
};
```

새 페이지의 주소와 구조를 반환합니다. `initialize_identity_maps` 함수의 마지막 목표는 `pgdt_buf_size` 및 `pgt_buf_offset`을 초기화하는 것입니다. 초기화 단계에만 있기 때문에 `initialze_identity_maps` 함수는 `pgt_buf_offset`을 0으로 설정합니다 :

```C
pgt_data.pgt_buf_offset = 0;
```

`pgt_data.pgt_buf_size`는 `77824` 또는 `69632`로 설정되며, 부트 로더 (64 비트 또는 32 비트)가 사용할 부트 프로토콜에 따라 다릅니다. `pgt_data.pgt_buf`도 마찬가지입니다. 부트 로더가 `startup_32`에서 커널을 로드했다면 `pgdt_data.pgdt_buf`는 [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) 에서 이미 초기화 된 페이지 테이블의 끝을 가리킵니다.:

```C
pgt_data.pgt_buf = _pgtable + BOOT_INIT_PGT_SIZE;
```

여기서 _pgtable은 이 페이지 테이블의 시작을 가리킵니다 [_pgtable](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S). 다른 방법으로, 부트 로더가 64-비트 부트 프로토콜을 사용하고 `startup_64`에서 커널을 로드했다면, 초기 페이지 테이블은 부트 로더 자체에 의해 작성되어야하며 `_pgtable`은 그냥 덮어씌워집니다. :

```C
pgt_data.pgt_buf = _pgtable
```

새로운 페이지 테이블을위한 버퍼가 초기화되면, 우리는 `choose_random_location` 함수로 되돌아 갈 수 있습니다.

예약 된 메모리 범위는 피하세요.
--------------------------------------------------------------------------------

속성 페이지 테이블과 관련된 내용이 초기화 된 후 압축 해제 된 커널 이미지를 넣을 위치를 임의로 선택할 수 있습니다. 그러나 짐작할 수 있듯이 주소를 선택할 수 없습니다. 메모리 범위에는 일부 예약된 주소가 있습니다. 이러한 주소는 [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk)나 커널 명령 행 등과 같은 중요한 것들이 차지합니다.

```C
mem_avoid_init(input, input_size, *output);
```

기능을 통해 이 작업을 수행 할 수 있습니다. 안전하지 않은 모든 메모리 영역이 다음 배열에 수집됩니다.:

```C
struct mem_vector {
	unsigned long long start;
	unsigned long long size;
};

static struct mem_vector mem_avoid[MEM_AVOID_MAX];
```

`MEM_AVOID_MAX`는 다른 유형의 예약된 메모리 영역을 나타내는 [열거형](https://en.wikipedia.org/wiki/Enumerated_type#C) `mem_avoid_index`에서 온 것입니다 :

```C
enum mem_avoid_index {
	MEM_AVOID_ZO_RANGE = 0,
	MEM_AVOID_INITRD,
	MEM_AVOID_CMDLINE,
	MEM_AVOID_BOOTPARAMS,
	MEM_AVOID_MEMMAP_BEGIN,
	MEM_AVOID_MEMMAP_END = MEM_AVOID_MEMMAP_BEGIN + MAX_MEMMAP_REGIONS - 1,
	MEM_AVOID_MAX,
};
```

둘 다 소스 코드 파일 [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr.c) 에 정의되어 있습니다.

`mem_avoid_init` 함수의 구현을 살펴 봅시다. 이 함수의 주요 목표는 `mem_avoid` 배열의 `mem_avoid_index` 열거형을 이용하여 예약 된 메모리 영역에 대한 정보를 저장하고 새로운 아이디 매핑 버퍼에서 이러한 영역에 대한 새 페이지를 만드는 것입니다. `mem_avoid_index` 함수의 수많은 부분은 비슷하지만 그중 하나를 살펴 봅시다.:

```C
mem_avoid[MEM_AVOID_ZO_RANGE].start = input;
mem_avoid[MEM_AVOID_ZO_RANGE].size = (output + init_size) - input;
add_identity_map(mem_avoid[MEM_AVOID_ZO_RANGE].start,
		 mem_avoid[MEM_AVOID_ZO_RANGE].size);
```

`mem_avoid_init` 함수의 시작 부분에서 현재 커널 압축 해제에 사용되는 메모리 영역을 피하려고 합니다. 우리는 `mem_avoid` 배열의 엔트리를 시작과 그 영역의 사이즈로 채우고 이 영역에 대해 아이디 매핑된 페이지를 빌드해야하는 `add_identity_map` 함수를 호출합니다. `add_identity_map` 함수는 소스코드 파일 [arch/x86/boot/compressed/kaslr_64.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr_64.c) 에 정의되어 있습니다. :

```C
void add_identity_map(unsigned long start, unsigned long size)
{
	unsigned long end = start + size;

	start = round_down(start, PMD_SIZE);
	end = round_up(end, PMD_SIZE);
	if (start >= end)
		return;

	kernel_ident_mapping_init(&mapping_info, (pgd_t *)top_level_pgt,
				  start, end);
}
```

보시다시피 메모리 영역을 2MB 경계에 맞추고 주어진 시작과 끝 주소를 확인합니다.

끝으로, 소스코드 파일 [arch/x86/mm/ident_map.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/mm/ident_map.c)에서 `kernel_ident_mapping_init` 함수를 호출하고, 위에서 초기화 된 `mapping_info` 인스턴스와 최상위 페이지 테이블 주소 및 새 아이디 매핑이 빌드되어야 하는 메모리 영역 주소를 전달합니다.

`kernel_ident_mapping_init` 함수는 새 페이지가 제공되지 않은 경우 새 플래그에 대한 디폴트 플래그를 설정합니다. :

```C
if (!info->kernpg_flag)
	info->kernpg_flag = _KERNPG_TABLE;
```

그리고 주어진 주소와 관련된 새 2MB(`mapping_info.page_flag`의 `PSE` 비트로 인해)를 페이지 엔트리([5레벨 페이지 테이블](https://lwn.net/Articles/717293/)의 경우 `PGD -> P4D -> PUD -> PMD`, [4레벨 페이지 테이블](https://lwn.net/Articles/117749/)의 경우 `PGD -> PUD -> PMD`)에 빌드하기 시작합니다.

```C
for (; addr < end; addr = next) {
	p4d_t *p4d;

	next = (addr & PGDIR_MASK) + PGDIR_SIZE;
	if (next > end)
		next = end;

    p4d = (p4d_t *)info->alloc_pgt_page(info->context);
	result = ident_p4d_init(info, p4d, addr, next);

    return result;
}
```
우선 여기서 우리는 주어진 주소에 대한 `Page Global Directory`의 다음 엔트리를 찾고, 주어진 메모리 영역의 `end`보다 큰 경우 이를 `end`로 설정합니다. 그런 다음 위에서 이미 고려한 `x86_mapping_info` 콜백으로 새 페이지를 할당하고 `ident_p4d_init` 함수를 호출합니다. `ident_p4d_init` 함수는 같은 기능을 하지만 낮은 래밸의 페이지 디렉토리 (`p4d`->`pud`->`pmd`)에 대해서도 동일합니다.

이제 끝났습니다.

예약 된 주소와 관련된 새 페이지 항목은 페이지 테이블에 있습니다. 이것은 `mem_avoid_init` 함수의 끝이 아니지만 다른 부분은 비슷합니다. [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk), 커널 명령 줄 등에 대한 페이지만 빌드합니다.

이제 우리는`choose_random_location` 함수로 돌아갈 수 있습니다.

물리적 주소 랜덤화
--------------------------------------------------------------------------------

예약 된 메모리 영역이 `mem_avoid` 배열에 저장되고 이를 위해 아이디 매핑 페이지가 구축 된 후, 커널을 압축 해제하기 위한 랜덤한 메모리 영역을 선택하기 위해 사용 가능한 최소 주소를 선택합니다.

```C
min_addr = min(*output, 512UL << 20);
```

보시다시피 영역은 512MB보다 작아야합니다. 이 `512`MB 값은 낮은 메모리에서는 알 수없는 것을 피하기 위해 선택되었습니다.

다음 단계는 랜덤한 물리적 주소와 가상 주소를 선택하여 커널을 로드하는 것입니다. 첫 번째로 실제 주소를 선택합시다. :

```C
random_addr = find_random_phys_addr(min_addr, output_size);
```

`find_random_phys_addr` 함수는 [같은](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr.c) 소스 코드 파일에 정의되어 있습니다.

```
static unsigned long find_random_phys_addr(unsigned long minimum,
                                           unsigned long image_size)
{
	minimum = ALIGN(minimum, CONFIG_PHYSICAL_ALIGN);

	if (process_efi_entries(minimum, image_size))
		return slots_fetch_random();

	process_e820_entries(minimum, image_size);
	return slots_fetch_random();
}
```

'process_efi_entries' 함수의 주요 목표는 커널을 로드하기 위해 접근 가능한 전체 메모리에서 모두 적합한 메모리 범위를 찾는 것입니다. 만약 커널이 [EFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)의 서포트가 없는 시스템에서 컴파일 및 실행되었다면, 우리는 메모리 영역을 [e820](https://en.wikipedia.org/wiki/E820) 영역에서 계속 찾습니다. 모든 찾은 메모리 영억은 다음 배열에 저장될 것입니다. :

```C
struct slot_area {
	unsigned long addr;
	int num;
};

#define MAX_SLOT_AREA 100

static struct slot_area slot_areas[MAX_SLOT_AREA];
```

커널은 압축 해제 할 이 배열의 랜덤한 인덱스를 선택합니다. 이 선택은 `slots_fetch_random` 함수에 의해 실행됩니다. `slots_fetch_random` 함수의 주요 목표는 `kaslr_get_random_long` 함수를 통해 `slot_areas` 배열에서 랜덤 메모리 범위를 선택하는 것입니다. :

```C
slot = kaslr_get_random_long("Physical") % slot_max;
```
`kaslr_get_random_long` 함수는 소스코드파일 [arch/x86/lib/kaslr.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/lib/kaslr.c)에 정의되어 있으며, 난수를 리턴합니다. 난수는 커널 구성 및 시스템 기회([time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter), [rdrand](https://en.wikipedia.org/wiki/RdRand) 등등에 기반하여 선택)에 따라 다른 방식으로 선택된다는 것을 알아두세요.

이 시점에서 모든 메모리 범위가 선택되는 것으로 끝입니다.

가상 주소 랜덤화
--------------------------------------------------------------------------------

커널 압축 해제 프로그램에 의해 랜덤한 메모리 영역이 선택된 후, 이 영역에 대한 새로운 아이디 매핑 페이지가 필요에 따라 작성됩니다. :

```C
random_addr = find_random_phys_addr(min_addr, output_size);

if (*output != random_addr) {
		add_identity_map(random_addr, output_size);
		*output = random_addr;
}
```

이제부터 `output`은 커널이 압축 해제 될 메모리 영역의 기본 주소를 저장합니다. 그러나 현재로서는 기억하듯이 물리적 주소만 랜덤으로 배정했습니다. [x86_64](https://ko.wikipedia.org/wiki/X86-64) 아키텍처의 경우에는 가상 주소도 무작위로 지정해야합니다.

```C
if (IS_ENABLED(CONFIG_X86_64))
	random_addr = find_random_virt_addr(LOAD_PHYSICAL_ADDR, output_size);

*virt_addr = random_addr;
```

'x86_64' 아키텍쳐의 경우에서 볼 수 있듯이, 랜덤 가상 주소는 랜덤 물리 주소와 일치합니다. `find_random_virt_addr` 함수는 커널 이미지를 가질 수 있는 가상 메모리 범위의 양을 계산하고 이전 랜덤한 `물리적`주소를 찾으려고 시도했을 때 보았던 `kaslr_get_random_long`을 호출합니다.

이 시점부터 우리는 압축 해제 된 커널에 대한 랜덤화 된 기본 물리적 (`* output`) 및 가상 (`* virt_addr`) 주소를 모두 가지고 있습니다.

끝입니다.

결론
--------------------------------------------------------------------------------

이것이 리눅스 커널 부팅 프로세스의 6 번째이자 마지막 부분입니다. 우리는 더 이상 커널 부팅에 대한 게시물을 볼 수 없지만 (이 게시물과 이전 게시물에 대한 업데이트는 있을 수 있음) 다른 커널 내부에 대한 게시물이 많이 있습니다.

다음 장에서는 커널 초기화에 대해 설명하고 Linux 커널 초기화 코드의 첫 단계를 봅니다.

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견이나 핑을 남겨주세요.

**영어는 제 모국어가 아닙니다. 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수를 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한글 번역](https://github.com/junsooo/linux-insides-ko)으로 PR을 보내주세요.**

링크
--------------------------------------------------------------------------------

* [Address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [Linux kernel boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt)
* [long mode](https://en.wikipedia.org/wiki/Long_mode)
* [initrd](https://en.wikipedia.org/wiki/Initial_ramdisk)
* [Enumerated type](https://en.wikipedia.org/wiki/Enumerated_type#C)
* [four-level page tables](https://lwn.net/Articles/117749/)
* [five-level page tables](https://lwn.net/Articles/717293/)
* [EFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
* [e820](https://en.wikipedia.org/wiki/E820)
* [time stamp counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [rdrand](https://en.wikipedia.org/wiki/RdRand)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [Previous part](linux-bootstrap-5.md)
