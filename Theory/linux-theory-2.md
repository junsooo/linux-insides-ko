Executable and Linkable Format
================================================================================

ELF(Executable and Linkable Format)는 실행 파일, 오브젝트 파일, 공유 라이브러리와 코어 덤프를 위한 표준 파일 포맷입니다. 리눅스와 많은 유닉스 운영체제가 이 포맷을 사용합니다. 리눅스 커널 소스에서 ELF-64 오브젝트 파일 포맷과 일부 정의를 살펴 봅시다.

ELF 오브젝트 파일은 다음과 같은 부분으로 구성됩니다:

* ELF 헤더 - 오브젝트 파일의 주요 특성(characteristics)을 기술함: 형식, CPU 아키텍쳐, 진입점의 가상 주소, 나머지 부분들의 크기와 오프셋 등등...;
* 프로그램 헤더 테이블 - 유효한 세그먼트들과 속성 리스트. 로더는 프로그램 헤더 테이블은 파일 섹션을 가상 메모리 세그먼트로 위치 시키기 위해 사용함;
* 섹션 헤더 테이블 - 섹션 세부 정보 포함.

이제 이 구성 요소들을 좀더 자세히 살펴 봅시다.

**ELF 헤더**

ELF 헤더는 오브젝트 파일의 시작 부분에 위치합니다. 주요 목적은 오브젝트 파일의 나머지 모든 부분을 찾는 것입니다. 파일 헤더는 다음과 같은 필드를 포함합니다:

* ELF 신분증명(identification) - 파일을 ELF 오브젝트 파일로 식별하는데 도움이 되는 바이트 배열과 일반 오브젝트 파일 특성에 대한 정보를 제공;
* 오브젝트 파일 유형  - 오브젝트 파일의 유형을 식별함. 재배치 가능 오브젝트 파일, 실행 파일 등등으로 이 필드에 유형을 표기함.
* 타겟 아키텍쳐;
* 오브젝트 파일 포맷의 버전;
* 프로그램 진입점의 가상 주소;
* 프로그램 헤더 테이블의 파일 오프셋;
* 섹션 헤더 테이블의 파일 오프셋;
* ELF 헤더 크기;
* 프로그램 헤더 테이블 엔트리의 크기;
* 그리고 나머지 필드들...

리눅스 커널 소스 코드에서 ELF 헤더를 나타내는 `elf64_hdr` 구조체를 찾을 수 있습니다:

```C
typedef struct elf64_hdr {
	unsigned char	e_ident[EI_NIDENT];
	Elf64_Half e_type;
	Elf64_Half e_machine;
	Elf64_Word e_version;
	Elf64_Addr e_entry;
	Elf64_Off e_phoff;
	Elf64_Off e_shoff;
	Elf64_Word e_flags;
	Elf64_Half e_ehsize;
	Elf64_Half e_phentsize;
	Elf64_Half e_phnum;
	Elf64_Half e_shentsize;
	Elf64_Half e_shnum;
	Elf64_Half e_shstrndx;
} Elf64_Ehdr;
```

[elf.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/uapi/linux/elf.h#L220) 안에 이 구조체가 있습니다.

**섹션들**

모든 데이터는 ELF 오브젝트 파일 내에 섹션들에 저장합니다. 섹션들은 섹션 헤더 테이블의 인덱스로 식별됩니다. 섹션 헤더는 다음과 같은 필드를 포함합니다:

* 섹션 이름;
* 섹션 유형;
* 섹션 속성;
* 메모리 내 가상 주소;
* 파일 내 오프셋;
* 섹션의 크기;
* 타 섹션으로 연결;
* 기타 정보;
* 얼라인먼트(alignment) 경계 주소;
* 섹션이 테이블을 포함하면 엔트리의 크기;

그리고 리눅스 커널 내에 다음과 같은 `elf64_shdr` 구조체로 표현됩니다:

```C
typedef struct elf64_shdr {
	Elf64_Word sh_name;
	Elf64_Word sh_type;
	Elf64_Xword sh_flags;
	Elf64_Addr sh_addr;
	Elf64_Off sh_offset;
	Elf64_Xword sh_size;
	Elf64_Word sh_link;
	Elf64_Word sh_info;
	Elf64_Xword sh_addralign;
	Elf64_Xword sh_entsize;
} Elf64_Shdr;
```

[elf.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/uapi/linux/elf.h#L312)

**프로그램 헤더 테이블**

모든 섹션은 실행 가능 또는 공유 오브젝트 파일 내에 세그먼트로 그룹화됩니다. 프로그램 헤더는 모든 세그먼트를 기술하는 구조체의 배열입니다. 다음과 같은 모양입니다:
```C
typedef struct elf64_phdr {
	Elf64_Word p_type;
	Elf64_Word p_flags;
	Elf64_Off p_offset;
	Elf64_Addr p_vaddr;
	Elf64_Addr p_paddr;
	Elf64_Xword p_filesz;
	Elf64_Xword p_memsz;
	Elf64_Xword p_align;
} Elf64_Phdr;
```

`elf64_phdr`은 리눅스 커널 소스 코드 내에 [elf.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/uapi/linux/elf.h#L254)에 정의되어 있습니다.

또한 ELF 오브젝트 파일은 [Documentation](http://www.uclibc.org/docs/elf-64-gen.pdf)에서 찾을 수 있는 다른 필드/구조체를 포함하고 있습니다. 이제 `vmlinux` ELF 오브젝트를 살펴 봅시다.

vmlinux
--------------------------------------------------------------------------------

`vmlinux`도 재배치 가능 ELF 오브젝트 파일입니다. `readelf` 유틸리티로 확인 가능합니다. 먼저 헤더 부분을 확인해 봅시다:

```
$ readelf -h  vmlinux
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              EXEC (Executable file)
  Machine:                           Advanced Micro Devices X86-64
  Version:                           0x1
  Entry point address:               0x1000000
  Start of program headers:          64 (bytes into file)
  Start of section headers:          381608416 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         5
  Size of section headers:           64 (bytes)
  Number of section headers:         73
  Section header string table index: 70
```

`vmlinux`는 64비트 실행 가능 파일이라는 것을 확인할 수 있습니다.

[Documentation/x86/x86_64/mm.txt](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt#L21)에서 다음과 부분을 볼 수 있습니다:

```
ffffffff80000000 - ffffffffa0000000 (=512 MB)  kernel text mapping, from phys 0
```

이 주소를 `vmlinux` ELF 오브젝트에서 이 부분들 확인해 보면:

```
$ readelf -s vmlinux | grep ffffffff81000000
     1: ffffffff81000000     0 SECTION LOCAL  DEFAULT    1
 65099: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 _text
 90766: ffffffff81000000     0 NOTYPE  GLOBAL DEFAULT    1 startup_64
```

`startup_64` 루틴의 주소가 `ffffffff80000000`이 아니라 `ffffffff81000000`라는 것에 주목해주세요. 이에 대한 이유를 설명하겠습니다.

[arch/x86/kernel/vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/vmlinux.lds.S)에서 다음과 같은 정의를 확인할 수 있습니다:

```
    . = __START_KERNEL;
	...
	...
	..
	/* Text and read-only data */
	.text :  AT(ADDR(.text) - LOAD_OFFSET) {
		_text = .;
		...
		...
		...
	}
```

`__START_KERNEL`은 다음과 같이 정의되어 있고:

```
#define __START_KERNEL		(__START_KERNEL_map + __PHYSICAL_START)
```

위 문서로부터의 `__START_KERNEL_map` 값은 `ffffffff80000000`이고 `__PHYSICAL_START`는 `0x1000000`입니다. 그래서 `startup_64` 값은 `ffffffff81000000`입니다.

그리고 마지막으로 다음과 같은 명령으로 `vmlinux`에서 프로그램 헤더를 확인할 수 있습니다:

```
readelf -l vmlinux

Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000000cfd000 0x0000000000cfd000  R E    200000
  LOAD           0x0000000001000000 0xffffffff81e00000 0x0000000001e00000
                 0x0000000000100000 0x0000000000100000  RW     200000
  LOAD           0x0000000001200000 0x0000000000000000 0x0000000001f00000
                 0x0000000000014d98 0x0000000000014d98  RW     200000
  LOAD           0x0000000001315000 0xffffffff81f15000 0x0000000001f15000
                 0x000000000011d000 0x0000000000279000  RWE    200000
  NOTE           0x0000000000b17284 0xffffffff81917284 0x0000000001917284
                 0x0000000000000024 0x0000000000000024         4

 Section to Segment mapping:
  Segment Sections...
   00     .text .notes __ex_table .rodata __bug_table .pci_fixup .builtin_fw
          .tracedata __ksymtab __ksymtab_gpl __kcrctab __kcrctab_gpl
		  __ksymtab_strings __param __modver
   01     .data .vvar
   02     .data..percpu
   03     .init.text .init.data .x86_cpu_dev.init .altinstructions
          .altinstr_replacement .iommu_table .apicdrivers .exit.text
		  .smp_locks .data_nosave .bss .brk
```

섹션 리스트에서 5개의 세그먼트가 보입니다. `arch/x86/kernel/vmlinux.lds`에 생성된 링커 스크립트에서 이 섹션들을 모드 찾을 수 있습니다.

이게 끝입니다. 물론 이것이 ELF (Executable and Linkable Format)에 대해 모두 기술한 것은 아니며 더 정보가 필요하면 [여기](http://www.uclibc.org/docs/elf-64-gen.pdf)에서 찾을 수 있습니다.
