커널 초기화. Part 1

커널 코드의 첫 단계
--------------------------------------------------------------------------------

이전 [포스트](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-6.html)는 Linux 커널 [부팅 프로세스](https : // 0xax.gitbooks.io/linux-insides/content/Booting/index.html)챕터의 마지막 부분이었습니다. 그리고 이제 Linux 커널의 초기화 과정을 시작합니다. Linux 커널 이미지가 압축 해제되고 메모리의 올바른 위치에 배치되면 작동하기 시작합니다. 이전의 모든 부분에서는 Linux 커널 코드의 첫 바이트가 실행되기 전에 준비하는 Linux 커널 설정 코드의 작업에 대해 설명합니다. 이제 우리는 커널에 있으며 이 장에서는 [pid](https://en.wikipedia.org/wiki/Process_identifier) `1`로 프로세스를 시작하기 전에 커널의 초기화 과정에 집중 할 것입니다. 커널이 `init`프로세스를 시작하기 전에 해야 할 일이 많이 있습니다. 이 큰 장에서 커널이 시작되기 전에 모든 준비 과정을 보게되기를 바랍니다. [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)에 있는 커널 진입 점에서 시작하겠습니다. [init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)에서 `start_kernel` 함수를 호출 되는 것을 보기 전에 초기 페이지 테이블 초기화, 커널 공간에서 새 디스크립터로 전환하는 등과 같은 첫 번째 준비를 살펴볼 것입니다.

```assembly
jmp	*%rax
```

현재 `rax` 레지스터에는 [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/master/arch/x86/boot/compressed/misc.c) 소스 코드 파일에서 `decompress_kernel` 함수를 호출 한 결과 얻은 Linux 커널 진입 점의 주소가 포함되어 있습니다. 커널 설정 코드의 마지막 명령은 커널 진입 점을 뛰어 넘는 것입니다. 우리는 이미 Linux 커널의 진입 점이 정의되어 있다는 것을 알고 있으므로 Linux 커널이 무엇을하는지 배울 수 있습니다.

커널의 첫 단계
--------------------------------------------------------------------------------

압축 해제 된 커널 이미지의 주소를 `decompress_kernel` 함수에서 `rax`레지스터로 가져 와서 바로 점프 하였습니다. 우리가 이미 알고 있듯이 압축 해제 된 커널 이미지의 시작점은 [arch/x86/kernel/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)어셈블리 소스 코드 파일에서 시작하고 그 시작 부분에서 다음 정의를 볼 수 있습니다:

```assembly
    .text
	__HEAD
	.code64
	.globl startup_64
startup_64:
	...
	...
	...
```

실행 가능한 `.head.text`섹션의 정의로 확장되는 매크로 인 `__HEAD`섹션에 정의 된 `startup_64`루틴의 정의를 볼 수 있습니다:

```C
#define __HEAD		.section	".head.text","ax"
```

이 섹션의 정의를 [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S) 링커 스크립트에서 볼 수 있다:

```
.text : AT(ADDR(.text) - LOAD_OFFSET) {
	_text = .;
	...
	...
	...
} :text = 0x9090
```

`.text`섹션의 정의 외에도 링커 스크립트에서 기본 가상 주소와 물리적 주소를 이해할 수 있습니다. `_text`의 주소는 다음과 같이 정의되는 위치 카운터입니다:

```
. = __START_KERNEL;
```

[x86_64](https://en.wikipedia.org/wiki/X86-64) 위해. `__START_KERNEL`매크로의 정의는 [arch/x86/include/asm/page_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_types.h)헤더 파일에 있고 커널 매핑의 기본 가상 주소와 물리적 시작의 합으로 표시됩니다:

```C
#define __START_KERNEL	(__START_KERNEL_map + __PHYSICAL_START)

#define __PHYSICAL_START  ALIGN(CONFIG_PHYSICAL_START, CONFIG_PHYSICAL_ALIGN)
```

혹은 다른 표현으로:

* 리눅스 커널의 기본 물리 주소 - `0x1000000`;
* 리눅스 커널의 기본 가상 주소 - `0xffffffff81000000`.

CPU구성을 삭제한 후, [arch/x86/kernel/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)에 정의된 `__startup_64`함수를 호출한다:

```assembly
	leaq	_text(%rip), %rdi
	pushq	%rsi
	call	__startup_64
	popq	%rsi
```

```C
unsigned log __head __startup_64(unsigned long physaddr,
				 struct boot_params *bp)
{
	unsigned long load_delta, *p;
	unsigned long pgtable_flags;
	pgdval_t *pgd;
	p4dval_t *p4d;
	pudval_t *pud;
	pmdval_t *pmd, pmd_entry;
	pteval_t *mask_ptr;
	bool la57;
	int i;
	unsigned int *next_pgt_ptr;
	...
	...
	...
}
```

[kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization#Linux)을 사용하도록 설정되었으므로 로드 된 'startup_64'루틴 주소가 컴파일 된 주소와 다를 수 있으므로 델타를 다음 코드를 통해 계산해야합니다:

```C
	load_delta = physaddr - (unsigned long)(_text - __START_KERNEL_map);
```

결과적으로, `load_delta`는 컴파일 된 주소와 실제로 로드 된 주소 사이의 델타를 포함합니다.

델타를 얻은 후 `_text` 주소가 `2` 메가 바이트에 올바르게 정렬되어 있는지 확인합니다. 다음 코드를 사용하여 수행합니다:

```C
	if (load_delta & ~PMD_PAGE_MASK)
		for (;;);
```

`_text` 주소가 `2` 메가 바이트로 정렬되지 않으면 무한 루프가 됩니다. `PMD_PAGE_MASK`는 `Page middle directory`에 대한 마스크를 나타냅니다([Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html) 참조) 그리고 다음과 같이 정의됩니다:

```C
#define PMD_PAGE_MASK           (~(PMD_PAGE_SIZE-1))
```

where `PMD_PAGE_SIZE` macro is defined as:

```C
#define PMD_PAGE_SIZE           (_AC(1, UL) << PMD_SHIFT)
#define PMD_SHIFT		21
```

쉽게 계산할 수있는 `PMD_PAGE_SIZE`는 `2`MB입니다.

[SME](https://en.wikipedia.org/wiki/Zen_%28microarchitecture%29#Enhanced_security_and_virtualization_support)가 지원되고 허용 된 경우 이를 활성화하고 `load_delta`에 SME 암호화 마스크를 포함시킵니다:

```C
	sme_enable(bp);
	load_delta += sme_get_me_mask();
```

자, 우리는 몇 가지 체크를 하였고 계속 진행할 수 있습니다.

페이지 테이블의 기본 주소 수정
--------------------------------------------------------------------------------

다음 단계에서는 페이지 테이블의 실제 주소를 수정합니다:

```C
	pgd = fixup_pointer(&early_top_pgt, physaddr);
	pud = fixup_pointer(&level3_kernel_pgt, physaddr);
	pmd = fixup_pointer(level2_fixmap_pgt, physaddr);
```

전달 된 인자의 물리적 주소를 반환하는 `fixup_pointer`함수의 정의를 봅시다:

```C
static void __head *fixup_pointer(void *ptr, unsigned long physaddr)
{
	return ptr - (void *)_text + (void *)physaddr;
}
```

다음으로 우리는 `early_top_pgt`와 위에서 본 다른 페이지 테이블 심볼에 초점을 맞출 것입니다. 이 기호들이 무엇을 의미하는지 이해하려고 노력합시다. 우선 그들의 정의를 봅시다:

```assembly
NEXT_PAGE(early_top_pgt)
	.fill	512,8,0
	.fill	PTI_USER_PGD_FILL,8,0

NEXT_PAGE(level3_kernel_pgt)
	.fill	L3_START_KERNEL,8,0
	.quad	level2_kernel_pgt - __START_KERNEL_map + _KERNPG_TABLE_NOENC
	.quad	level2_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC

NEXT_PAGE(level2_kernel_pgt)
	PMDS(0, __PAGE_KERNEL_LARGE_EXEC,
		KERNEL_IMAGE_SIZE/PMD_SIZE)

NEXT_PAGE(level2_fixmap_pgt)
	.fill	506,8,0
	.quad	level1_fixmap_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC
	.fill	5,8,0

NEXT_PAGE(level1_fixmap_pgt)
	.fill	512,8,0
```

 어려워 보이지만 그렇지 않습니다. 우선 `early_top_pgt`를 봅시다. `4096`바이트의 0으로 (`CONFIG_PAGE_TABLE_ISOLATION`이 활성화 된 경우 `8192`바이트로) 시작합니다. 이는 첫 번째 `512`항목을 사용하지 않음을 의미합니다. 그리고 나서 `level3_kernel_pgt`항목을 볼 수 있습니다. 정의의 시작 부분에서 `4080`바이트의 0으로 채워져 있음을 알 수 있습니다(`L3_START_KERNEL`은 `510`). 이어서 커널 공간을 매핑하는 두 개의 항목을 저장합니다. `level2_kernel_pgt` 과 `level2_fixmap_pgt`에서 `__START_KERNEL_map`을 뺍니다. 우리가 알고있는 것처럼 `__START_KERNEL_map`은 커널 텍스트의 기본 가상 주소이므로 `__START_KERNEL_map`을 빼면 `level2_kernel_pgt` 과 `level2_fixmap_pgt`의 물리적 주소를 얻게됩니다.

다음으로 `_KERNPG_TABLE_NOENC`와 `_PAGE_TABLE_NOENC`를 봅시다. 이들은 페이지 항목 액세스 권한입니다:

```C
#define _KERNPG_TABLE_NOENC   (_PAGE_PRESENT | _PAGE_RW | _PAGE_ACCESSED | \
			       _PAGE_DIRTY)
#define _PAGE_TABLE_NOENC     (_PAGE_PRESENT | _PAGE_RW | _PAGE_USER | \
			       _PAGE_ACCESSED | _PAGE_DIRTY)
```

`level2_kernel_pgt`는 커널 공간을 매핑하는 페이지 중간 디렉토리에 대한 포인터를 포함하는 페이지 테이블 항목입니다. 커널 `.text`에 대한 `__START_KERNEL_map`에서 `512` 메가 바이트를 생성하는 PDMS 매크로를 호출합니다.  (이 `512`메가 바이트는 모듈 메모리 공간이됩니다)

`level2_fixmap_pgt`는 커널 공간에서도 물리적 주소를 참조 할 수있는 가상 주소입니다. 그것들은 `4048` 바이트의 0, `level1_fixmap_pgt` 엔트리, [vsyscalls](https://lwn.net/Articles/446528/)매핑을 위해 예약 된 `8`메가 바이트와 `2` 메가 바이트의 홀로 표시됩니다.

자세한 내용은 [Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html)부분에서 읽을 수 있습니다.

이제이 심볼의 정의를 확인한 후 코드로 돌아가 보겠습니다. 다음으로 `level3_kernel_pgt`로 `pgd`의 마지막 엔트리를 초기화합니다:

```C
	pgd[pgd_index(__START_KERNEL_map)] = level3_kernel_pgt - __START_KERNEL_map + _PAGE_TABLE_NOENC;
```

`startup_64`가 기본 `0x1000000` 주소와 같지 않으면 모든 p * d 주소가 잘못되었을 수 있습니다. `load_delta`는 커널 [링킹](https://en.wikipedia.org/wiki/Linker_%28computing%29) 동안 얻은 `startup_64`심볼의 주소와 실제 주소 사이의 델타를 포함합니다. 따라서 `p * d`의 특정 항목에 델타를 추가합니다.

```C
	pgd[pgd_index(__START_KERNEL_map)] += load_delta;
	pud[510] += load_delta;
	pud[511] += load_delta;
	pmd[506] += load_delta;
```

이 모든 후에 다음 결과를 얻게 됩니다:

```
early_top_pgt[511] -> level3_kernel_pgt[0]
level3_kernel_pgt[510] -> level2_kernel_pgt[0]
level3_kernel_pgt[511] -> level2_fixmap_pgt[0]
level2_kernel_pgt[0]   -> 512 MB kernel mapping
level2_fixmap_pgt[506] -> level1_fixmap_pgt
```

우리는 `early_top_pgt`의 기본 주소와 일부 다른 페이지 테이블 디렉토리를 고치지 않았습니다. 이 페이지 테이블의 building/filling 구조때 이것이 보이기 때문입니다. 페이지 테이블의 기본 주소를 수정하면 빌드를 시작할 수 있습니다.

아이디  매핑 설정
--------------------------------------------------------------------------------

이제 초기 페이지 테이블의 아이디 매핑 설정을 볼 수 있습니다. Identity Mapped Paging에서 가상 주소는 물리적 주소에 동일하게 매핑됩니다. 자세히 살펴 보겠습니다. 우선 `pud`와 `pmd`를 `early_dynamic_pgts`의 첫 번째와 두 번째 항목에 대한 포인터로 바꿉니다:

```C
	next_pgt_ptr = fixup_pointer(&next_early_pgt, physaddr);
	pud = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);
	pmd = fixup_pointer(early_dynamic_pgts[(*next_pgt_ptr)++], physaddr);
```

`early_dynamic_pgts` 정의를 봅시다:

```assembly
NEXT_PAGE(early_dynamic_pgts)
	.fill	512*EARLY_DYNAMIC_PAGE_TABLES,8,0
```

초기 커널에 대한 임시 페이지 테이블을 저장합니다.

다음으로 `p*d` 항목을 초기화 할 때 사용되는 `pgtable_flags`를 초기화합니다:

```C
	pgtable_flags = _KERNPG_TABLE_NOENC + sme_get_me_mask();
```

`sme_get_me_mask`함수는 `sme_enable`함수에서 초기화 된`sme_me_mask`를 반환합니다.

다음으로　`pud`와　`pgtable_flags`로 위에서 두 개의　`pgd` 항목을 채운다:

```C
	i = (physaddr >> PGDIR_SHIFT) % PTRS_PER_PGD;
	pgd[i + 0] = (pgdval_t)pud + pgtable_flags;
	pgd[i + 1] = (pgdval_t)pud + pgtable_flags;
```

｀PGDIR_SHFT｀는 가상 주소의 페이지 글로벌 디렉토리 비트에 대한 마스크를 나타냅니다. 여기에서는　`512`보다 큰 인덱스에 액세스하지 않도록　`PTRS_PER_PGD`(`512`로 확장)로 모듈로를 계산합니다. 모든 유형의 페이지 디렉토리에 대한 매크로가 있습니다：

```C
#define PGDIR_SHIFT     39
#define PTRS_PER_PGD	512
#define PUD_SHIFT       30
#define PTRS_PER_PUD	512
#define PMD_SHIFT       21
#define PTRS_PER_PMD	512
```

다음에도 위에서와 같은 일을 합니다：

```C
	i = (physaddr >> PUD_SHIFT) % PTRS_PER_PUD;
	pud[i + 0] = (pudval_t)pmd + pgtable_flags;
	pud[i + 1] = (pudval_t)pmd + pgtable_flags;
```

다음으로 `pmd_entry`를 초기화하고 지원되지 않는 `__PAGE_KERNEL_ *` 비트를 걸러냅니다:

```C
	pmd_entry = __PAGE_KERNEL_LARGE_EXEC & ~_PAGE_GLOBAL;
	mask_ptr = fixup_pointer(&__supported_pte_mask, physaddr);
	pmd_entry &= *mask_ptr;
	pmd_entry += sme_get_me_mask();
	pmd_entry += physaddr;
```

다음으로 모든 `pmd` 항목을 채워서 커널의 전체 크기를 다룹니다:

```C
	for (i = 0; i < DIV_ROUND_UP(_end - _text, PMD_SIZE); i++) {
		int idx = i + (physaddr >> PMD_SHIFT) % PTRS_PER_PMD;
		pmd[idx] = pmd_entry + i * PMD_SIZE;
	}
```

다음으로 커널 텍스트 + 데이터 가상 주소를 수정합니다. 커널을 재배치 할 때 유효하지 않은 pmd를 작성할 수 있습니다 (`cleanup_highmap` 함수는 `_end` 이외의 매핑과 함께 이를 수정합니다).

```C
	pmd = fixup_pointer(level2_kernel_pgt, physaddr);
	for (i = 0; i < PTRS_PER_PMD; i++) {
		if (pmd[i] & _PAGE_PRESENT)
			pmd[i] += load_delta;
	}
```

다음으로 메모리 암호화 마스크를 제거하여 실제 주소를 얻습니다(`load_delta`에 마스크가 포함되어 있음을 기억하십시오):

```C
	*fixup_long(&phys_base, physaddr) += load_delta - sme_get_me_mask();
```

`phys_base`는 `level2_kernel_pgt`의 첫 번째 항목과 일치해야합니다.

`__startup_64` 함수의 마지막 단계로 커널을 암호화하고 (SME가 활성화 된 경우) `cr3` 레지스터에 프로그래밍 된 초기 페이지 디렉토리 항목의 제어자로 사용할 SME 암호화 마스크를 반환합니다:

```C
	sme_encrypt_kernel(bp);
	return sme_get_me_mask();
```

이제 어셈블리 코드로 돌아가 봅시다. 다음 코드를 사용하여 다음을 준비합니다:

```assembly
	addq	$(early_top_pgt - __START_KERNEL_map), %rax
	jmp 1f
```

`early_top_pgt`의 물리적 주소를 `rax` 레지스터에 추가하여 `rax` 레지스터에 주소와 SME 암호화 마스크의 합이 포함되도록합니다.

지금은 여기까지입니다. 초기 페이징이 준비되었으므로 커널 준비 지점으로 이동하기 전에 마지막 준비를 마치면됩니다.

커널 진입 점에서 점프하기 전에 마지막 준비
--------------------------------------------------------------------------------

그런 다음 레이블 `1`로 이동하여 `PAE`, `PGE` (Paging Global Extension)를 활성화하고 `phys_base`(위 참조)의 내용을 `rax` 레지스터에 넣고`cr3` 레지스터를 채웁니다:

```assembly
1:
	movl	$(X86_CR4_PAE | X86_CR4_PGE), %ecx
	movq	%rcx, %cr4

	addq	phys_base(%rip), %rax
	movq	%rax, %cr3
```

다음 단계에서 CPU가 다음을 사용하여 [NX](http://en.wikipedia.org/wiki/NX_bit)비트를 지원하는지 확인합니다:

```assembly
	movl	$0x80000001, %eax
	cpuid
	movl	%edx,%edi
```

`0x80000001` 값을 `eax`에 넣고 확장 프로세서 정보 및 기능 비트를 얻기 위해 `cpuid` 명령을 실행합니다. 결과는 `edi`에 넣은 `edx` 레지스터에 있을 것이다.

이제 우리는 `0xc0000080` 또는 `MSR_EFER`를 `ecx`에 넣고 판독 모델 특정 레지스터에 대해 `rdmsr` 명령을 실행합니다.

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
```

결과는 `edx : eax`에 나타납니다. `EFER`의 일반적인 견해는 다음과 같습니다:

```
63                                                                              32
 --------------------------------------------------------------------------------
|                                                                               |
|                                Reserved MBZ                                   |
|                                                                               |
 --------------------------------------------------------------------------------
31                            16  15      14      13   12  11   10  9  8 7  1   0
 --------------------------------------------------------------------------------
|                              | T |       |       |    |   |   |   |   |   |   |
| Reserved MBZ                 | C | FFXSR | LMSLE |SVME|NXE|LMA|MBZ|LME|RAZ|SCE|
|                              | E |       |       |    |   |   |   |   |   |   |
 --------------------------------------------------------------------------------
```

여기서 모든 필드를 자세히 볼 수는 없지만 이에 대한 특별한 부분에서 이 필드와 다른 `MSR`에 대해 알아 봅니다. `edx : eax`에 `EFER`을 읽으면서, `btsl` 명령으로 `_EFER_SCE` 또는 `System Call Extensions` 인 0 비트를 체크하고 1로 설정합니다. `SCE` 비트를 설정함으로써 우리는 `SYSCALL`과 `SYSRET` 명령을 활성화합니다. 다음 단계에서 우리는 `edi`에서 20 번째 비트를 검사합니다. 이 레지스터는`cpuid`의 결과를 저장한다는 것을 기억하십시오 (위 참조). 만약 `20` 비트가 설정되면 (`NX` 비트) 우리는 `EFER_SCE`를 모델 특정 레지스터에 씁니다.

```assembly
	btsl	$_EFER_SCE, %eax
	btl	$20,%edi
	jnc     1f
	btsl	$_EFER_NX, %eax
	btsq	$_PAGE_BIT_NX,early_pmd_flags(%rip)
1:	wrmsr
```

[NX](https://en.wikipedia.org/wiki/NX_bit)비트가 지원되는 경우 `_EFER_NX`를 활성화하고 `wrmsr`명령을 사용하여 쓰십시오. [NX](https://en.wikipedia.org/wiki/NX_bit)비트가 설정된 후, 우리는 다음 어셈블리 코드를 사용하여 `cr0` [제어 레지스터](https://en.wikipedia.org/wiki/Control_register)에 있는 몇몇 비트들을 설정합니다:

```assembly
	movl	$CR0_STATE, %eax
	movq	%rax, %cr0
```

특히 다음 비트들:

* `X86_CR0_PE` - system is in protected mode;
* `X86_CR0_MP` - controls interaction of WAIT/FWAIT instructions with TS flag in CR0;
* `X86_CR0_ET` - on the 386, it allowed to specify whether the external math coprocessor was an 80287 or 80387;
* `X86_CR0_NE` - enable internal x87 floating point error reporting when set, else enables PC style x87 error detection;
* `X86_CR0_WP` - when set, the CPU can't write to read-only pages when privilege level is 0;
* `X86_CR0_AM` - alignment check enabled if AM set, AC flag (in EFLAGS register) set, and privilege level is 3;
* `X86_CR0_PG` - enable paging.

우리는 코드를 실행하고 어셈블리에서 더 많은 [C](https://en.wikipedia.org/wiki/C_%28programming_language%29) 코드를 실행하려면 스택을 설정해야 한다는 것을 이미 압니다. 항상 그렇듯이 우리는 [스택 포인터](https://en.wikipedia.org/wiki/Stack_register)를 메모리의 올바른 위치로 설정하고 [플래그](https://en.wikipedia.org/wiki/FLAGS_register)를 재설정하여 이 작업을 수행합니다:

```assembly
	movq initial_stack(%rip), %rsp
	pushq $0
	popfq
```

여기서 가장 흥미로운 것은 `initial_stack`입니다. 이 심볼은 [소스 코드 파일](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)코드 파일에 정의되어 있으며 다음과 같습니다.

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union + THREAD_SIZE - SIZEOF_PTREGS
```

`THREAD_SIZE` 매크로는 [arch/x86/include/asm/page_64_types.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/page_64_types.h)헤더 파일에 정의되어 있습니다. `KASAN_STACK_ORDER` 매크로의 값에 따라 다릅니다:

```C
#ifdef CONFIG_KASAN
#define KASAN_STACK_ORDER 1
#else
#define KASAN_STACK_ORDER 0
#endif

#define THREAD_SIZE_ORDER       (2 + KASAN_STACK_ORDER)
#define THREAD_SIZE  (PAGE_SIZE << THREAD_SIZE_ORDER)
```

[kasan](https://github.com/torvalds/linux/blob/master/Documentation/dev-tools/kasan.rst)이 비활성화되고 `PAGE_SIZE`가 `4096` 바이트 인 경우를 고려합니다. 따라서 `THREAD_SIZE`는 `16`킬로바이트로 확장되며 스레드 스택의 크기를 나타냅니다. 왜 `스레드`일까요? 각 [프로세스](https://en.wikipedia.org/wiki/Process_%28computing%29)에 [부모 프로세스](https://en.wikipedia.org/wiki/Parent_process)와 [자식 프로세스](https://en.wikipedia.org/wiki/Child_process)가 있을 것을 이미 알고 있을 것입니다. 실제로 부모 프로세스와 자식 프로세스는 스택에서 다릅니다. 새로운 프로세스에 새로운 커널 스택이 할당됩니다. 리눅스 커널에서　이 스택은　`thread_info` 구조를 가진 [union](https://en.wikipedia.org/wiki/Union_type#C.2FC.2B.2B)으로 표현됩니다.

`init_thread_union`은　`thread_union`으로 표시됩니다. `thread_union`은 다음과 같이 [include/linux/sched.h](https://github.com/torvalds/linux/blob/master/include/linux/sched.h) 파일에 정의되어 있습니다：

```C
union thread_union {
#ifndef CONFIG_ARCH_TASK_STRUCT_ON_STACK
	struct task_struct task;
#endif
#ifndef CONFIG_THREAD_INFO_IN_TASK
	struct thread_info thread_info;
#endif
	unsigned long stack[THREAD_SIZE/sizeof(long)];
};
```

`CONFIG_ARCH_TASK_STRUCT_ON_STACK` 커널 구성 옵션은　`ia64`아키텍처에서만 활성화되며　`CONFIG_THREAD_INFO_IN_TASK` 커널 구성 옵션은　`x86_64` 아키텍처에서 활성화됩니다. 따라서 ｀thread_info｀구조체는 ｀thread_union｀공용체 대신 ｀task_struct｀구조체에 배치됩니다.

`init_thread_union`은 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/blob/master/include/asm-generic/vmlinux.lds.h) 파일에 다음과 같은 `INIT_TASK_DATA`매크로의 일부로 배치됩니다:

```C
#define INIT_TASK_DATA(align)  \
	. = ALIGN(align);      \
	...                    \
	init_thread_union = .; \
	...
```

이 매크로는 [arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S)파일에서 다음과 같이 사용됩니다:

```
.data : AT(ADDR(.data) - LOAD_OFFSET) {
	...
	INIT_TASK_DATA(THREAD_SIZE)
	...
} :data
```

즉, `init_thread_union`은 `16`킬로바이트인 `THREAD_SIZE`에 정렬 된 주소로 초기화됩니다.

이제 우리는 이 표현을 이해할 수 있습니다:

```assembly
GLOBAL(initial_stack)
    .quad  init_thread_union + THREAD_SIZE - SIZEOF_PTREGS
```

`initial_stack` 기호는 16 킬로바이트인 `thread_union.stack` 배열 +`THREAD_SIZE`의 시작과 커널 내 언와인더가 스택의 끝을 안정적으로 감지하는 데 도움이되는 규칙 인`SIZEOF_PTREGS`를 나타냅니다.

초기 부팅 스택을 설정 한 후,`lgdt`명령으로 [글로벌 디스크립터 테이블](https://en.wikipedia.org/wiki/Global_Descriptor_Table)을 업데이트합니다:

```assembly
lgdt	early_gdt_descr(%rip)
```

여기서 `early_gdt_descr`은 다음과 같이 정의됩니다:

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

커널은 낮은 사용자 공간 주소에서 작동하지만 곧 커널은 자체 공간에서 작동하기 때문에 `글로벌 디스크럽터 테이블`을 다시 로드해야합니다.

이제 `early_gdt_descr`의 정의를 봅시다. `GDT_ENTRIES`는 `32`로 확장되어 글로벌 디스크럽터 테이블에 커널 코드, 데이터, 스레드 로컬 스토리지 세그먼트 등에 대한 `32` 항목이 포함됩니다.

이제 `early_gdt_descr_base`의 정의를 봅시다. `gdt_page`구조체는 [arch/x86/include/asm/desc.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/desc.h)에 정의되어 있습니다:

```C
struct gdt_page {
	struct desc_struct gdt[GDT_ENTRIES];
} __attribute__((aligned(PAGE_SIZE)));
```


여기에는 다음과 같이 정의되는 `desc_struct` 구조체의 배열인 하나의 필드 `gdt`가 포함됩니다:

```C
struct desc_struct {
         union {
                 struct {
                         unsigned int a;
                         unsigned int b;
                 };
                 struct {
                         u16 limit0;
                         u16 base0;
                         unsigned base1: 8, type: 4, s: 1, dpl: 2, p: 1;
                         unsigned limit: 4, avl: 1, l: 1, d: 1, g: 1, base2: 8;
                 };
         };
 } __attribute__((packed));
```

친숙한 `GDT` 디스크립터가 보입니다. `gdt_page` 구조는 `4096`바이트 인 `PAGE_SIZE`에 맞춰집니다. 이것은 `gdt`가 한 페이지를 차지한다는 것을 의미합니다.
which looks familiar `GDT` descriptor. Note that `gdt_page` structure is aligned to `PAGE_SIZE` which is `4096` bytes. Which means that `gdt` will occupy one page.

이제 `INIT_PER_CPU_VAR`이 무엇인지 이해해 봅시다. `INIT_PER_CPU_VAR`은 [arch/x86/include/asm/percpu.h](https://github.com/torvalds/linux/blob/master/arch/x86/include/asm/percpu.h)에 정의 된 매크로입니다. 그리고 주어진 매개 변수로 `init_per_cpu__`를 연결합니다.

```C
#define INIT_PER_CPU_VAR(var) init_per_cpu__##var
```

`INIT_PER_CPU_VAR` 매크로가 확장 된 후, 우리는 `init_per_cpu__gdt_page`를 갖게됩니다. [링커 스크립터](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/vmlinux.lds.S)에서 `init_per_cpu__gdt_page`의 초기화를 볼 수 있습니다.

```
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
```

`INIT_PER_CPU_VAR`에 `init_per_cpu__gdt_page`가 있고 링커 스크립트에서 `INIT_PER_CPU` 매크로가 확장되면 `__per_cpu_load`에서 오프셋을 얻습니다. 이 계산 후에는 새 GDT의 정확한 기본 주소를 갖게됩니다.

일반적으로 CPU 별 변수는 2.6 커널 기능입니다. 이름에서 무엇인지 이해할 수 있습니다. 우리가 `per-CPU '변수를 만들면, 각 CPU는 이 변수의 자체 복사본을 갖게됩니다. 여기서 우리는 CPU 당 `gdt_page`를 만들고 있습니다. 각 CPU가 고유한 변수 사본 등으로 작동하므로 이 타입의 변수에는 잠금이 없다는 것과 같은 많은 이점이 있습니다. 따라서 멀티 프로세서의 모든 코어에는 자체 `GDT`테이블과 코어에서 실행 된 스레드에서 액세스 할 수있는 메모리 세그먼트를 나타내는 테이블의 모든 항목이 있습니다. [Concepts/per-cpu](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) 포스트에서 `per-CPU`변수에 대한 자세한 내용을 읽을 수 있습니다.

새로운　글로벌　디스크럽터　테이블을　로드 할 때마다 매번 해왔듯이 세그먼트를 다시로드합니다：

```assembly
	xorl %eax,%eax
	movl %eax,%ds
	movl %eax,%ss
	movl %eax,%es
	movl %eax,%fs
	movl %eax,%gs
```

이 모든 단계 후에 `gs` 레지스터를 설정하여 [interrupts](https://en.wikipedia.org/wiki/Interrupt)가 처리 될 특별한 스택을 나타내는　`irqstack`에 게시합니다:

```assembly
	movl	$MSR_GS_BASE,%ecx
	movl	initial_gs(%rip),%eax
	movl	initial_gs+4(%rip),%edx
	wrmsr
```

여기서 MSR_GS_BASE는 다음과 같습니다:

```C
#define MSR_GS_BASE             0xc0000101
```

`MSR_GS_BASE`를 `ecx` 레지스터에 넣고 `wrmsr` 명령으로 `eax` 및 `edx`(`initial_gs`를 가리킴)에서 데이터를 로드해야합니다. 64 비트 모드 주소 지정에서 `cs`, `fs`, `ds` 및 `ss` 세그먼트 레지스터를 사용하지 않지만 `fs` 및 `gs` 레지스터를 사용할 수 있습니다. `fs`와 `gs`는 숨겨진 부분이 있고 (`cs`의 실제 모드에서 본 것처럼) 이 부분에는 [Model Specific Registers](https://en.wikipedia.org/wiki/Model-specific_register)에 매핑 된 설명자가 들어 있습니다. 위의 `0xc0000101`은 `gs.base` MSR 주소입니다. [시스템 호출](https://en.wikipedia.org/wiki/System_call) 또는 [인터럽트](https://en.wikipedia.org/wiki/Interrupt)가 발생하면 진입점에 커널 스택이 없습니다. 따라서 MSR_GS_BASE의 값은 인터럽트 스택의 주소를 저장합니다.

다음 단계에서 우리는 리얼 모드 bootparam 구조체의 주소를 `rdi`에 넣고 (`rsi`는 처음부터 이 구조에 대한 포인터를 가지고 있음을 기억하세요) 다음과 같이 C 코드로 점프합니다:

```assembly
	pushq	$.Lafter_lret	# put return address on stack for unwinder
	xorq	%rbp, %rbp	# clear frame pointer
	movq	initial_code(%rip), %rax
	pushq	$__KERNEL_CS	# set correct cs
	pushq	%rax		# target address in negative space
	lretq
.Lafter_lret:
```

여기서 우리는 `initial_code`의 주소를 `rax`에 넣고 리턴 주소 `__KERNEL_CS`와 `initial_code`의 주소를 스택에 푸시합니다. 이 후 우리는 `lretq` 명령을 볼 수 있는데, 이는 반환 주소가 스택에서 추출되고(이제 `initial_code`의 주소가 있음) 그곳으로 점프함을 의미합니다. `initial_code`는 동일한 소스 코드 파일에 정의되어 있으며 다음과 같습니다:

```assembly
	.balign	8
	GLOBAL(initial_code)
	.quad	x86_64_start_kernel
	...
	...
	...
```

보시다시피 `initial_code`는 [arch/x86/kerne/head64.c](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head64.c)에 정의 되어있는 `x86_64_start_kernel`주소를 포함합니다. 다음과 같습니다:

```C
asmlinkage __visible void __init x86_64_start_kernel(char * real_mode_data)
{
	...
	...
	...
}
```

여기에는 하나의 전달인자인 `real_mode_data`가 있습니다.(전에 리얼 모드 데이터의 주소를 `rdi` 레지스터에 전달한 것을 기억하세요).

start_kernel
--------------------------------------------------------------------------------

"커널 진입점"-[init/main.c](https://github.com/torvalds/linux/blob/master/init/main.c)에서 start_kernel 함수를 보기 전 마지막 준비를 해야합니다.

우선 우리는 `x86_64_start_kernel` 함수에서 몇 가지 확인를 할 수 있습니다:

```C
BUILD_BUG_ON(MODULES_VADDR < __START_KERNEL_map);
BUILD_BUG_ON(MODULES_VADDR - __START_KERNEL_map < KERNEL_IMAGE_SIZE);
BUILD_BUG_ON(MODULES_LEN + KERNEL_IMAGE_SIZE > 2*PUD_SIZE);
BUILD_BUG_ON((__START_KERNEL_map & ~PMD_MASK) != 0);
BUILD_BUG_ON((MODULES_VADDR & ~PMD_MASK) != 0);
BUILD_BUG_ON(!(MODULES_VADDR > __START_KERNEL));
MAYBE_BUILD_BUG_ON(!(((MODULES_END - 1) & PGDIR_MASK) == (__START_KERNEL & PGDIR_MASK)));
BUILD_BUG_ON(__fix_to_virt(__end_of_fixed_addresses) <= MODULES_END);
```

모듈 공간의 가상 주소가 커널 텍스트의 기본 주소보다 작지 않은 `__STAT_KERNEL_map`과 같은 다른 것들에 대한 검사가 있습니다, 모듈이 있는 커널 텍스트는 커널 이미지보다 작지 않습니다. `BUILD_BUG_ON`은 다음과 같은 매크로:

```C
#define BUILD_BUG_ON(condition) ((void)sizeof(char[1 - 2*!!(condition)]))
```

이 트릭이 어떻게 작동하는지 이해해 봅시다. 첫 번째 조건 인 `MODULES_VADDR <__START_KERNEL_map`을 예로 들어 보겠습니다. `!! conditions`는 `condition! = 0`과 같습니다. 따라서 `MODULES_VADDR <__START_KERNEL_map`이 true이면 `!! (condition)`에서 `1`을 얻거나 그렇지 않으면 0을 얻습니다. `2 * !! (조건)`후에 우리는 `2`또는 `0`을 얻습니다. 계산이 끝나면 두 가지 다른 동작을 얻을 수 있습니다.

* 음수 인덱스를 가진 char 배열의 크기를 얻으려고 시도하기 때문에 컴파일 오류가 발생합니다(우리의 경우 처럼 `MODULES_VADDR`이 `__START_KERNEL_map`보다 작을 수 없기 때문에);
* 컴파일 오류가 없습니다.

그게 다입니다. 일부 상수에 따라 컴파일 오류가 발생하는 흥미로운 C 트릭입니다.

다음 단계에서는 CPU 당 `cr4`의 쉐도우 복사본을 저장하는 `cr4_init_shadow` 함수의 호출을 볼 수 있습니다. 컨텍스트 스위치는 `cr4`의 비트를 변경할 수 있으므로 각 CPU마다 `cr4`를 저장해야합니다. 그리고 나서 모든 페이지 글로벌 디렉토리 엔트리를 재설정하고 `cr3`에서 PGT에 대한 새로운 포인터를 작성하는 `reset_early_page_tables` 함수의 호출을 볼 수 있습니다.

```C
	memset(early_top_pgt, 0, sizeof(pgd_t)*(PTRS_PER_PGD-1));
	next_early_pgt = 0;
	write_cr3(__sme_pa_nodebug(early_top_pgt));
```

곧 새로운 페이지 테이블을 만들 것입니다. 여기서 모든 페이지 글로벌 디렉토리 항목이 0임을 알 수 있습니다. 그런 다음 `next_early_pgt`를 0으로 설정하고(다음 포스트에서 이에 대한 세부 사항을 볼 것입니다) `early_top_pgt`의 물리적 주소를 `cr3`에 씁니다.

그런 다음 `_bss`를 `__bss_stop`부터 `__bss_start`까지 지우고 `init_top_pgt`도 지웁니다. `init_top_pgt`는 다음과 같이 [arch/x86/kerne/head_64.S](https://github.com/torvalds/linux/blob/master/arch/x86/kernel/head_64.S)에 정의되어 있습니다:

```assembly
NEXT_PGD_PAGE(init_top_pgt)
	.fill	512,8,0
	.fill	PTI_USER_PGD_FILL,8,0
```

이것은 `early_top_pgt`와 정확히 같은 정의입니다.

다음 단계는 초기 `IDT`핸들러의 설정이지만, 큰 개념이므로 다음 장에서 볼 것입니다.

결론
--------------------------------------------------------------------------------

이것이 리눅스 커널 초기화에 대한 첫 번째 부분입니다.

이것이 리눅스 커널 부팅 과정의 네 번째 부분입니다. 질문이나 제안이 있으면 트위터 [0xAX](https://twitter.com/0xAX)나 [email](anotherworldofworld@gmail.com)을 보내거나 [issue](https://github.com/0xAX/linux-insides/issues/new)를 만드십시오.

다음 부분에서는 초기 인터럽트 처리기, 커널 공간 메모리 매핑 등의 초기화를 볼 수 있습니다.

**모국어가 영어가 아니면 죄송합니다. 실수를 발견하면 PR을 [linux-insides](https://github.com/0xAX/linux-internals)로 보내주십시오.**

링크
--------------------------------------------------------------------------------

* [Model Specific Register](http://en.wikipedia.org/wiki/Model-specific_register)
* [Paging](https://0xax.gitbooks.io/linux-insides/content/Theory/linux-theory-1.html)
* [Previous part - kernel load address randomization](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-6.html)
* [NX](http://en.wikipedia.org/wiki/NX_bit)
* [ASLR](http://en.wikipedia.org/wiki/Address_space_layout_randomization)
