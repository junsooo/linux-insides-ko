커널 초기화. Part 6.
================================================================================

아키텍쳐 별 초기화
================================================================================

이전의 [파트](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-5.html)에서 우리는 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c)에서 아키텍쳐 별(우리의 경우 `x86_64`) 초기화를 보았고 [NX bit](http://en.wikipedia.org/wiki/NX_bit)의 지원에 의존하는 `_PAGE_NX`플래그로 설정된 `x86_configure_nx`함수를 끝냈습니다. 이전에 작성한 `setup_arch`함수와 `start_kernel`이 매우 크므로 이번과 다음 파트에서 우리는 아키텍쳐 별 초기화 프로세스를 계속할 것입니다. `x86_configure_nx`의 다음 함수는 `parse_early_param`입니다. 이 함수는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에서 정의됐으며 이름에서 알 수 있듯이, 이 함수는 커널 커맨드 라인을 분석하고 주어진 매개변수에 따라 다른 서비스를 설정합니다.(모든 커널 커맨드 라인 매개변수는 [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)에서 찾을 수 있습니다). 당신은 아마 초기 [파트](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)에서 우리가 어떻게 `earlyprintk`을 설정했는지 기억할 것입니다. 초반부에 우리는 커널 매개변수를 봤습니다. 그것들의 값은 [arch/x86/boot/cmdline.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/cmdline.c)의 `cmdline_find_option`함수와 `__cmdline_find_option`,`__cmdline_find_option_bool`헬퍼입니다. 우리는 아키텍쳐에 의존하지 않는 일반적인 커널 파트를 보며 다른 접근법을 사용합니다. 리눅스 소스 코드를 읽는 중이라면 다음과 같은 호출에 주목하십시오:

```C
early_param("gbpages", parse_direct_gbpages_on);
```

`early_param`매크로는 두 매개변수를 가집니다:

* 커맨드 라인 매개변수의 이름;
* 주어진 매개변수가 전달되면 호출되는 함수

다음과 같이 정의됩니다:

```C
#define early_param(str, fn) \
        __setup_param(str, fn, fn, 1)
```

[include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h)에 포함됩니다. `early_param`매크로가 `__setup_param`매크로를 호출하는 것을 볼 수 있습니다:

```C
#define __setup_param(str, unique_id, fn, early)                \
        static const char __setup_str_##unique_id[] __initconst \
                __aligned(1) = str; \
        static struct obs_kernel_param __setup_##unique_id      \
                __used __section(.init.setup)                   \
                __attribute__((aligned((sizeof(long)))))        \
                = { __setup_str_##unique_id, fn, early }
```

이 매크로는 `__setup_str_*_id`변수(*주어진 함수 이름에 따라 다름)를 정의하고 주어진 커맨드 라인 매개변수의 이름을 정의합니다. 다음 줄에서 우리는 `obs_kernel_param`타입의 `__setup_*`변수의 정의와 초기화를 볼 수 있습니다. `obs_kernel_param`구조체의 정의:

```C
struct obs_kernel_param {
        const char *str;
        int (*setup_func)(char *);
        int early;
};
```

and contains three fields:
그리고 3개의 필드를 포함합니다:

* 커널 매개변수의 이름;
* 매개변수에 따라 다른 무언가를 설정하는 함수;
* 필드가 초기 값(1)인지 아닌지(0)를 결정.

참고 `__set_param`매크로를 정의하는 `__section(.init.setup)`속성. 이것은 모든 `__setup_str_*`이 `.init.setup`섹션에 위치될 수 있음을 의미합니다. 더욱이 [include/asm-generic/vmlinux.lds.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/asm-generic/vmlinux.lds.h)에서 볼 수 있는 것처럼 그것들은 `__setup_start`과 `__setup_end`사이에 배치될 것입니다:

```
#define INIT_SETUP(initsetup_align)                \
                . = ALIGN(initsetup_align);        \
                VMLINUX_SYMBOL(__setup_start) = .; \
                *(.init.setup)                     \
                VMLINUX_SYMBOL(__setup_end) = .;
```

이제 우리는 매개변수가 어떻게 정의되는지 알았습니다. `parse_early_param`의 구현으로 돌아갑시다:

```C
void __init parse_early_param(void)
{
        static int done __initdata;
        static char tmp_cmdline[COMMAND_LINE_SIZE] __initdata;

        if (done)
                return;

        /* All fall through to do_early_param. */
        strlcpy(tmp_cmdline, boot_command_line, COMMAND_LINE_SIZE);
        parse_early_options(tmp_cmdline);
        done = 1;
}
```

이 `parse_early_param`함수는 두개의 정적 변수를 정의합니다. 먼저 `done`은 `parse_early_param`가 이미 호출됬는지 확인하고 두번째는 커널 커맨드 라인을 위한 임시 저장소 입니다. 그 다음 우리는 방금 정의한 임시 커맨드 라인에 `boot_command_line`를 복사하고 동일한 소스 코드 `main.c`파일에서 `parse_early_options`함수를 호출합니다. `parse_early_options`는 [kernel/params.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/)에서 `parse_args`함수를 호출합니다. `parse_args`는 주어진 커맨드 라인을 분석하고 `do_early_param`함수를 호출합니다. 이 [함수](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c#L413) ` __setup_start`에서 `__setup_end`으로 이동하고, 매개변수가 early인 경우 `obs_kernel_param`함수를 호출합니다. 그 다음 early 커맨드 라인 매개변수에 의존하는 모든 서비스가 설정된 다음 `parse_early_param`는 `x86_report_nx`을 호출합니다. 이 파트의 도입부에서 썻듯이 우리는 이미 `NX-bit`를 `x86_configure_nx`로 설정했습니다. 다음으로 [arch/x86/mm/setup_nx.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/mm/setup_nx.c)의 `x86_report_nx`함수는 `NX`의 정보를 출력합니다. 참고로 `x86_report_nx`는 `x86_configure_nx`의 바로 다음이 아니라 `parse_early_param`를 호출한 다음 호출합니다. 대답은 간단합니다: 커널이 `noexec` 매개변수를 지원하므로 우리는 그것을 `parse_early_param`다음에 호출합니다:

```
noexec		[X86]
			On X86-32 available only on PAE configured kernels.
			noexec=on: enable non-executable mappings (default)
			noexec=off: disable non-executable mappings
```

우리는 그것의 booting 시간을 볼 수 있습니다:

![NX](http://oi62.tinypic.com/swwxhy.jpg)

다음으로 함수의 호출을 볼 수 있습니다:

```C
	memblock_x86_reserve_range_setup_data();
```

이 함수는 같은 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c) 소스 코드 파일에서 정의됐습니다. `setup_data`메모리를 다시 매핑하고  `setup_data`(`setup_data`에 대한 정보는 이전의 [파트](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-5.html)에서 읽을 수 있고 `ioremap`와 `memblock`에 관해서는 [리눅스 커널 메모리 관리](https://0xax.gitbooks.io/linux-insides/content/MM/index.html)에서 읽을 수 있습니다)메모리 블록을 전달합니다.

다음 단계에서는 다음과 같은 조건문을 볼 수 있습니다:

```C
	if (acpi_mps_check()) {
#ifdef CONFIG_X86_LOCAL_APIC
		disable_apic = 1;
#endif
		setup_clear_cpu_cap(X86_FEATURE_APIC);
	}
```

먼저 [arch/x86/kernel/acpi/boot.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/acpi/boot.c)의 `acpi_mps_check`함수는 `CONFIG_X86_LOCAL_APIC`과 `CONFIG_x86_MPPARSE`작동 옵션에 의존합니다:

```C
int __init acpi_mps_check(void)
{
#if defined(CONFIG_X86_LOCAL_APIC) && !defined(CONFIG_X86_MPPARSE)
        /* mptable code is not built-in*/
        if (acpi_disabled || acpi_noirq) {
                printk(KERN_WARNING "MPS support code is not built-in.\n"
                       "Using acpi=off or acpi=noirq or pci=noacpi "
                       "may have problem\n");
                 return 1;
        }
#endif
        return 0;
}
```

내장 `MPS`또는 [다중프로세서 사양](http://en.wikipedia.org/wiki/MultiProcessor_Specification)테이블을 확인합니다. `CONFIG_X86_LOCAL_APIC`가 설정되고 `CONFIG_x86_MPPAARSE`이 설정되지 않은 경우, 다음 옵션들 중 하나가 커널에 전달되면 `acpi_mps_check`는 경고 메시지를 출력합니다: `acpi=off`, `acpi=noirq` 또는 `pci=noacpi`. `acpi_mps_check`가 `1`을 반환하면 그것은 로컬 [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)가 비활성화되고 `setup_clear_cpu_cap`매크로에서 현재 CPUdml `X86_FEATURE_APIC`비트가 지워진 것을 의미합니다.(CPU마스크에 대한 것은 [CPU마스크](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)에서 읽을 수 있습니다).

초기 PCI 덤프
--------------------------------------------------------------------------------

다음 단계에서는 다음 코드를 사용하여 [PCI](http://en.wikipedia.org/wiki/Conventional_PCI)장치의 덤프를 만듭니다:

```C
#ifdef CONFIG_PCI
	if (pci_early_dump_regs)
		early_dump_pci_devices();
#endif
```

`pci_early_dump_regs`변수는 [arch/x86/pci/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/pci/common.c)에 정의되었고 이 값은 커널 커맨드 라인 매개변수:`pci=earlydump`에 의존합니다. 우리는 [drivers/pci/pci.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch)안에서 이 매개변수의 정의를 찾을 수 있습니다:

```C
early_param("pci", pci_setup);
```

`pci_setup`함수는 `pci=`문자열을 가져온 다음 분석합니다. 이 함수는 [drivers/pci/pci.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch)안의 `__weak`에서 정의된 `pcibios_setup`을 호출하고 모든 아키텍쳐는 `__weak`분석을 중단하는 동일한 함수를 정의합니다. 예를 들어 `x86_64`아키텍쳐 종속 버전은 [arch/x86/pci/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/pci/common.c)에 있습니다:

```C
char *__init pcibios_setup(char *str) {
        ...
		...
		...
		} else if (!strcmp(str, "earlydump")) {
                pci_early_dump_regs = 1;
                return NULL;
        }
		...
		...
		...
}
```

따라서 `CONFIG_PCI`옵션이 설정되고 `pci=earlydump`옵션이 커널 커맨드 라인에 전달되면, [arch/x86/pci/early.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/pci/early.c)의 `early_dump_pci_devices`함수가 호출됩니다. 이 함수는 다음과 같이 `noearly` pci 매개변수를 확인합니다:

```C
if (!early_pci_allowed())
        return;
```

그리고 그것이 전달되면 반환합니다. 각 PCI 도메인은 최대 `256`개의 버스를 호스트 할 수 있으며 각 버스는 최대 32개의 장치를 호스트 할 수 있습니다. 루프로 가봅시다:

```C
for (bus = 0; bus < 256; bus++) {
                for (slot = 0; slot < 32; slot++) {
                        for (func = 0; func < 8; func++) {
						...
						...
						...
                        }
                }
}
```

`read_pci_config`함수로 `pci`설정을 읽읍시다.

이것이 전부입니다. 우리는 `pci`의 세부사항은 깊게 들어가지 않지만 특수 `Drivers/PCI`파트에서 자세한 것을 볼 것입니다.

메모리 분석 완료
--------------------------------------------------------------------------------

다음은 [커널 설정의 첫 단계](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-2.html)파트에서 얻은 `early_dump_pci_devices`의 사용 가능한 메모리 및 [e820](http://en.wikipedia.org/wiki/E820)과 관련된 몇 가지 함수입니다.

```C
	/* update the e820_saved too */
	e820_reserve_setup_data();
	finish_e820_parsing();
	...
	...
	...
	e820_add_kernel_range();
	trim_bios_range(void);
	max_pfn = e820_end_of_ram_pfn();
	early_reserve_e820_mpc_new();
```

이것을 봅시다. 보시다시피 첫 번째 함수는 `e820_reserve_setup_data`입니다. 이 함수는 우리가 위에서 본 `memblock_x86_reserve_range_setup_data`와 거의 같지만 우리의 경우 주어진 `E820_RESERVED_KERN` 타입의 `e820map`에 새 지역을 더하는 `e820_update_range`을 호출합니다. 다음 함수는 `sanitize_e820_map`함수의 `e820map`을 제거하는 `finish_e820_parsing`함수입니다. 이 두 함수외에도 우리는 [e820](http://en.wikipedia.org/wiki/E820)과 연관된 몇 가지 함수를 위의 목록에서 볼 수 있습니다.  `e820_add_kernel_range`함수는 커널의 시작과 끝의 물리적 주소를 취합니다:

```C
u64 start = __pa_symbol(_text);
u64 size = __pa_symbol(_end) - start;
```

`.text` `.data`와 `.bss`가 `e820map`의 `E820RAM`로 표시되는지 확인하고 아니라면 경고 메시지를 출력합니다. 다음으로 `trm_bios_range`함수는 `e820Map`의 4096바이트를 `E820_RESERVED`으로 업데이트하고 `sanitize_e820_map`를 호출해서 그것을 다시 제거합니다. 다음으로 `e820_end_of_ram_pfn`함수를 호출해서 마지막 페이지의 프레임 넘버를 얻습니다. 모든 메모리 페이지는 고유의 넘버 `Page frame number`를 가집니다. `e820_end_of_ram_pfn`함수는 `e820_end_pfn` 호출의 최대값을 반환합니다:

```C
unsigned long __init e820_end_of_ram_pfn(void)
{
	return e820_end_pfn(MAX_ARCH_PFN);
}
```

`e820_end_pfn`에서 특정 아키텍쳐(`MAX_ARCH_PFN`은 `x86_64`를 위해 `0x400000000`)의 최대 페이지 프레임 수를 가져옵니다. `e820_end_pfn`에서 우리는 모든 `e820`슬롯을 통과하고 `e820`이 `E820_RAM` 또는 `E820_PRAM`타입에 진입했는지 확인합니다. 왜냐하면 우리는 이 타입들만을 위해 페이지 프레임 넘버를 계산했기 때문입니다. 기본 주소와 현재 `e820` 진입을 위한 페이지 프레임 넘버의 마지막 주소를 얻고 이 주소에 대해 몇 가지 확인을 합니다:

```C
for (i = 0; i < e820.nr_map; i++) {
		struct e820entry *ei = &e820.map[i];
		unsigned long start_pfn;
		unsigned long end_pfn;

		if (ei->type != E820_RAM && ei->type != E820_PRAM)
			continue;

		start_pfn = ei->addr >> PAGE_SHIFT;
		end_pfn = (ei->addr + ei->size) >> PAGE_SHIFT;

        if (start_pfn >= limit_pfn)
			continue;
		if (end_pfn > limit_pfn) {
			last_pfn = limit_pfn;
			break;
		}
		if (end_pfn > last_pfn)
			last_pfn = end_pfn;
}
```

```C
	if (last_pfn > max_arch_pfn)
		last_pfn = max_arch_pfn;

	printk(KERN_INFO "e820: last_pfn = %#lx max_arch_pfn = %#lx\n",
			 last_pfn, max_arch_pfn);
	return last_pfn;
```

다음으로 우리는 `last_pfn`에서 루프에 있는 것이 특정 아키텍쳐(우리의 경우 `x86_64`)의 최대 페이지 프레임 수보다 크지 않은지 확인하고 마지막 페이지 프레임 넘버의 정보를 출력하고 그것을 반환합니다. 우리는 `dmesg`출력에서 `last_pfn`을 볼 수 있습니다:

```
...
[    0.000000] e820: last_pfn = 0x41f000 max_arch_pfn = 0x400000000
...
```

그 후, 가장 큰 페이지 프레임 넘버를 계산할 때, `max_low_pfn`가 `low memory` 또는 첫 `4`기가바이트 이하의 가장 큰 페이지 프레임 넘버인지를 계산합니다. 4기가바이트 이상의 RAM을 설치한 경우 `max_low_pfn`은 `e820_end_of_ram_pfn`과 동일한 `e820_end_of_low_ram_pfn`함수의 결과가 될 것이지만 4기가바이트의 제한이 있습니다. 다른 방식으로는 `max_low_pfn`이 다음처럼 `max_pfn`과 같아질 것입니다:

```C
if (max_pfn > (1UL<<(32 - PAGE_SHIFT)))
	max_low_pfn = e820_end_of_low_ram_pfn();
else
	max_low_pfn = max_pfn;
		
high_memory = (void *)__va(max_pfn * PAGE_SIZE - 1) + 1;
```

다음으로 우리는 주어진 물리적 메모리에 의해 가상 주소를 반환하는 `__va`매크로의 `high_memory`(직접 맵 메모리에서 상한을 정의)를 계산합니다.

DMI 스캐닝 
-------------------------------------------------------------------------------

다음 단계는 다른 메모리 지역과 `e820` 슬롯을 조작한 후 컴퓨터에 대한 정보를 모으는 것입니다. [데스크탑 관리 인터페이스](http://en.wikipedia.org/wiki/Desktop_Management_Interface) 및 다음 함수를 통해 모든 정보를 얻을 수 있습니다:

```C
dmi_scan_machine();
dmi_memdev_walk();
```

먼저 `dmi_scan_machine`은 [drivers/firmware/dmi_scan.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/firmware/dmi_scan.c)에 정의되어 있습니다. 이 함수는 [시스템 관리 BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS)구조체를 통해서 정보를 추출합니다. `SMBIOS`테이블에 접근하기 위해 지정된 두 가지 방법이 있습니다: [EFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)구성 테이블에서 `SMBIOS`테이블의 포인터를 얻고 `0xF0000`와 `0x10000`주소 사이의 물리적 메모리를 스캔합니다. 두 번째 방법을 보겠습니다. `dmi_scan_machine`함수는 `early_ioremap`를 확장하는 `dmi_early_remap`에서 `0xf0000`와 `0x10000` 사이의 메모미를 다시 매핑합니다:

```C
void __init dmi_scan_machine(void)
{
	char __iomem *p, *q;
	char buf[32];
	...
	...
	...
	p = dmi_early_remap(0xF0000, 0x10000);
	if (p == NULL)
			goto error;
```

모든 `DMI`헤더 주소를 반복하고 `_SM_`문자열을 검색해서 찾습니다:

```C
memset(buf, 0, 16);
for (q = p; q < p + 0x10000; q += 16) {
		memcpy_fromio(buf + 16, q, 16);
		if (!dmi_smbios3_present(buf) || !dmi_present(buf)) {
			dmi_available = 1;
			dmi_early_unmap(p, 0x10000);
			goto out;
		}
		memcpy(buf, buf + 16, 16);
}
```

`_SM_`문자열은 `000F0000h`와 `0x000FFFFF` 사이에 있어야 합니다. 여기에서 우리는 16바이트를 동일한 `memcpy`에서 `memcpy_fromio`의 `buf`에 복사하고 버퍼에서 `dmi_smbios3_present`과 `dmi_present`를 실행합니다. 이 함수는 첫 4바이트가 `_SM_` 문자열인지 확인하고 `SMBIOS`버전을 얻고 `DMI`구조체 테이블 길이, 테이블 주소 등등 `_DMI_`의 속성을 얻습니다. 다음으로 이 함수 중 하나가 완료되면 `dmesg`출력에서 결과를 볼 수 있습니다:

```
[    0.000000] SMBIOS 2.7 present.
[    0.000000] DMI: Gigabyte Technology Co., Ltd. Z97X-UD5H-BK/Z97X-UD5H-BK, BIOS F6 06/17/2014
```

`dmi_scan_machine`의 끝으로 우리는 이전에 재매핑된 메모리를 언매핑합니다:

```C
dmi_early_unmap(p, 0x10000);
```

두 번째 함수는 `dmi_memdev_walk`입니다. 이해할 수 있도록 메모리 장치를 살펴봅시다:

```C
void __init dmi_memdev_walk(void)
{
	if (!dmi_available)
		return;

	if (dmi_walk_early(count_mem_devices) == 0 && dmi_memdev_nr) {
		dmi_memdev = dmi_alloc(sizeof(*dmi_memdev) * dmi_memdev_nr);
		if (dmi_memdev)
			dmi_walk_early(save_mem_devices);
	}
}
```

이것은 `DMI`가 사용가능한지(이전 함수 `dmi_scan_machine`에서 얻었습니다) 확인하고 `dmi_walk_early`에서 메모리 장치의 정보와 다음과 같이 정의된 `dmi_alloc`을 수집합니다.

```
#ifdef CONFIG_DMI
RESERVE_BRK(dmi_alloc, 65536);
#endif
```

`RESERVE_BRK`은 [arch/x86/include/asm/setup.h](http://en.wikipedia.org/wiki/Desktop_Management_Interface)에 정의되었고 `brk`섹션에서 주어진 크기의 공간을 예약합니다.

-------------------------
	init_hypervisor_platform();
	x86_init.resources.probe_roms();
	insert_resource(&iomem_resource, &code_resource);
	insert_resource(&iomem_resource, &data_resource);
	insert_resource(&iomem_resource, &bss_resource);
	early_gart_iommu_check();


SMP 설정
--------------------------------------------------------------------------------

다음 단계는 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)구성의 분석입니다. 함수 내부를 호출하는 `find_smp_config`함수의 호출을 실행합니다:

```C
static inline void find_smp_config(void)
{
        x86_init.mpparse.find_smp_config();
}
```

`x86_init.mpparse.find_smp_config`는 [arch/x86/kernel/mpparse.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/mpparse.c)의 `default_find_smp_config`함수에 있습니다. `default_find_smp_config`함수 안에서 우리는 `SMP`설정을 위해 몇 개의 메모리 지역을 스캔해서 그것들을 찾은 경우 반환합니다.

```C
if (smp_scan_config(0x0, 0x400) ||
            smp_scan_config(639 * 0x400, 0x400) ||
            smp_scan_config(0xF0000, 0x10000))
            return;
```

우선 모든 `smp_scan_config`함수는 몇 가지 변수를 정의합니다:

```C
unsigned int *bp = phys_to_virt(base);
struct mpf_intel *mpf;
```

첫 번째는 `SMP`구성을 스캔 할 메모리 지역의 가상 주소이고, 두 번째는 `mpf_intel`구조체에 대한 포인터입니다. `mpf_intel`가 무엇인지 이해해봅시다. 모든 정보는 멀티 프로세서 구성 데이터 구조체에 저장됩니다. `mpf_intel`은 이 구조체에 있습니다:

```C
struct mpf_intel {
        char signature[4];
        unsigned int physptr;
        unsigned char length;
        unsigned char specification;
        unsigned char checksum;
        unsigned char feature1;
        unsigned char feature2;
        unsigned char feature3;
        unsigned char feature4;
        unsigned char feature5;
};
```

설명서에서 읽었듯이 시스템 BIOS의 주요 기능 중 하나는 MP 부동 포인터 구조체와 MP 구성 테이블을 구성하는 것입니다. 그리고 운영 체제는 멀티 프로세서 구성에 대한 정보에 접근해야하며 `mpf_intel`은 멀티프로세서 구성 테이블의 물리적 주소(두 번째 매개변수를 참조)를 저장해야합니다. 따라서 `smp_scan_config`는 주어진 메모리 범위를 통해서 루프를 탐색하고 그곳에서 `MP floating pointer structure`를 찾으려 시도합니다. 현재 바이트 포인터가 `SMP`서명을 가리키는 지 확인하고 체크썸을 확인합니다. 그리고 루프에서 `mpf->specification`가 1 또는 4(`1` 또는 `4`로 설명되어야 함)인지 확인합니다:

```C
while (length > 0) {
if ((*bp == SMP_MAGIC_IDENT) &&
    (mpf->length == 1) &&
    !mpf_checksum((unsigned char *)bp, 16) &&
    ((mpf->specification == 1)
    || (mpf->specification == 4))) {

        mem = virt_to_phys(mpf);
        memblock_reserve(mem, sizeof(*mpf));
        if (mpf->physptr)
            smp_reserve_memory(mpf);
	}
}
```

`memblock_reserve`의 검색에 성공하면 주어진 메모리 블록을 예약하고 멀티프로세서 구성 테이블의 물리적 주소를 예약합니다. 이것에 관한 문서 - [멀티프로세서 설명서](http://www.intel.com/design/pentium/datashts/24201606.pdf)에서 찾아볼 수 있습니다. `SMP` 특별 파트에서도 더 상세한 내용을 읽을 수 있습니다.

추가적인 초기 메모리 초기화 루틴
--------------------------------------------------------------------------------

`setup_arch`의 다음 단계에서 우리는 초기 단계를 위해 페이지 테이블 버퍼를 할당하는 `early_alloc_pgt_buf`함수의 호출을 볼 수 있습니다. 페이지 테이블 버퍼는 `brk`영역에 배치됩니다. 구현을 살펴봅시다:

```C
void  __init early_alloc_pgt_buf(void)
{
        unsigned long tables = INIT_PGT_BUF_SIZE;
        phys_addr_t base;

        base = __pa(extend_brk(tables, PAGE_SIZE));

        pgt_buf_start = base >> PAGE_SHIFT;
        pgt_buf_end = pgt_buf_start;
        pgt_buf_top = pgt_buf_start + (tables >> PAGE_SHIFT);
}
```

첫째로 페이지 테이블 버퍼의 크기를 얻으면 현재 리눅스 커널 4.0에서 `(6 * PAGE_SIZE)`의 `INIT_PGT_BUF_SIZE`가 될 것입니다. 따라서 페이지 테이블 버퍼의 크기를 얻으면 크기와 정렬이라는 두 매개변수를 사용해 `extend_brk`함수를 호출합니다. 이름에서 알 수 있듯이 이 함수는 `brk`영역을 확장합니다. 리눅스 커널 스크립트 `brk`는 [BSS](http://en.wikipedia.org/wiki/.bss)의 바로 다음 메모리에서 볼 수 있습니다:

```C
	. = ALIGN(PAGE_SIZE);
	.brk : AT(ADDR(.brk) - LOAD_OFFSET) {
		__brk_base = .;
		. += 64 * 1024;		/* 64k alignment slop space */
		*(.brk_reservation)	/* areas brk users have reserved */
		__brk_limit = .;
	}
```

또는 `readelf`유틸을 사용해서 찾을 수 있습니다:

![brk area](http://oi61.tinypic.com/71lkeu.jpg)

`__pa`매크로를 통해 새로운 `brk`의 물리적 주소를 얻었으면, 기본 주소와 페이지 테이블 버퍼의 끝을 계산합니다. 다음 단계는 페이지 테이블 주소를 얻었으니 `reserve_brk`함수를 사용해서 brk영역을 위한 메모리 블록을 예약합니다.

```C
static void __init reserve_brk(void)
{
	if (_brk_end > _brk_start)
		memblock_reserve(__pa_symbol(_brk_start),
				 _brk_end - _brk_start);

	_brk_start = 0;
}
```

참고로 `reserve_brk`의 끝에서 아무것도 할당하지 않기 때문에 `brk_start`을 0으로 설정했습니다. ` brk`를 위한 메모리 블록을 예약한 다음 `cleanup_highmap`함수를 사용해 커널 매핑 범위를 벗어난 메모리 영역을 언매핑해야합니다. 커널 매핑이 `__START_KERNEL_map`과 `_end - _text` 또는 `_text`커널을 매핑하는 `level2_kernel_pgt`, `data`와 `bss`인 것을 기억하십시오. `clean_high_map`의 시작에서 다음과 같은 매개변수들을 정의합니다.

```C
unsigned long vaddr = __START_KERNEL_map;
unsigned long end = roundup((unsigned long)_end, PMD_SIZE) - 1;
pmd_t *pmd = level2_kernel_pgt;
pmd_t *last_pmd = pmd + PTRS_PER_PMD;
```

커널 매핑의 시작과 끝이 정의된 지금, 우리는 `_text`와 `end`사이에 있지 않은 모든 커널 페이지 미들 디렉터리 엔트리와 클린 엔트리를 통해 루프로 이동합니다:
```C
for (; pmd < last_pmd; pmd++, vaddr += PMD_SIZE) {
        if (pmd_none(*pmd))
            continue;
        if (vaddr < (unsigned long) _text || vaddr > end)
            set_pmd(pmd, __pmd(0));
}
```

그 후에 `memblock_set_current_limit`함수(`memblock`에 대한 것은[리눅스 커널 메모리 관리 파트 2](https://github.com/0xAX/linux-insides/blob/master/MM/linux-mm-2.md)에서 읽을 수 있습니다)를 사용해 `memblock`할당을 위한 제한을 설정합니다. 그것은 `ISA_END_ADDRESS` 또는 `0x100000`가 되고 `memblock_x86_fill`함수의 호출 `e820`에 따라서 `memblock`의 정보를 채웁니다. 커널 초기화 시간에서 이 함수의 결과를 볼 수 있습니다.

```
MEMBLOCK configuration:
 memory size = 0x1fff7ec00 reserved size = 0x1e30000
 memory.cnt  = 0x3
 memory[0x0]	[0x00000000001000-0x0000000009efff], 0x9e000 bytes flags: 0x0
 memory[0x1]	[0x00000000100000-0x000000bffdffff], 0xbfee0000 bytes flags: 0x0
 memory[0x2]	[0x00000100000000-0x0000023fffffff], 0x140000000 bytes flags: 0x0
 reserved.cnt  = 0x3
 reserved[0x0]	[0x0000000009f000-0x000000000fffff], 0x61000 bytes flags: 0x0
 reserved[0x1]	[0x00000001000000-0x00000001a57fff], 0xa58000 bytes flags: 0x0
 reserved[0x2]	[0x0000007ec89000-0x0000007fffffff], 0x1377000 bytes flags: 0x0
```

The rest functions after the `memblock_x86_fill` are: `early_reserve_e820_mpc_new` allocates additional slots in the `e820map` for MultiProcessor Specification table, `reserve_real_mode` - reserves low memory from `0x0` to 1 megabyte for the trampoline to the real mode (for rebooting, etc.), `trim_platform_memory_ranges` - trims certain memory regions started from `0x20050000`, `0x20110000`, etc. these regions must be excluded because [Sandy Bridge](http://en.wikipedia.org/wiki/Sandy_Bridge) has problems with these regions, `trim_low_memory_range` reserves the first 4 kilobyte page in `memblock`, `init_mem_mapping` function reconstructs direct memory mapping and setups the direct mapping of the physical memory at `PAGE_OFFSET`, `early_trap_pf_init` setups `#PF` handler (we will look on it in the chapter about interrupts) and `setup_real_mode` function setups trampoline to the [real mode](http://en.wikipedia.org/wiki/Real_mode) code.

That's all. You can note that this part will not cover all functions which are in the `setup_arch` (like `early_gart_iommu_check`, [mtrr](http://en.wikipedia.org/wiki/Memory_type_range_register) initialization, etc.). As I already wrote many times, `setup_arch` is big, and linux kernel is big. That's why I can't cover every line in the linux kernel. I don't think that we missed something important, but you can say something like: each line of code is important. Yes, it's true, but I missed them anyway, because I think that it is not realistic to cover full linux kernel. Anyway we will often return to the idea that we have already seen, and if something is unfamiliar, we will cover this theme.

결론
--------------------------------------------------------------------------------

It is the end of the sixth part about linux kernel initialization process. In this part we continued to dive in the `setup_arch` function again and it was long part, but we are not finished with it. Yes, `setup_arch` is big, hope that next part will be the last part about this function.

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-insides).**

링크
--------------------------------------------------------------------------------

* [멀티프로세서 설명서](http://en.wikipedia.org/wiki/MultiProcessor_Specification)
* [NX bit](http://en.wikipedia.org/wiki/NX_bit)
* [Documentation/kernel-parameters.txt](https://github.com/torvalds/linux/blob/master/Documentation/admin-guide/kernel-parameters.rst)
* [APIC](http://en.wikipedia.org/wiki/Advanced_Programmable_Interrupt_Controller)
* [CPU 마스크](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [리눅스 커널 메모리 관리](https://0xax.gitbooks.io/linux-insides/content/MM/index.html)
* [PCI](http://en.wikipedia.org/wiki/Conventional_PCI)
* [e820](http://en.wikipedia.org/wiki/E820)
* [시스템 관리 BIOS](http://en.wikipedia.org/wiki/System_Management_BIOS)
* [EFI](http://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [멀티프로세서 설명서](http://www.intel.com/design/pentium/datashts/24201606.pdf)
* [BSS](http://en.wikipedia.org/wiki/.bss)
* [SMBIOS 설명서](http://www.dmtf.org/sites/default/files/standards/documents/DSP0134v2.5Final.pdf)
* [이전 파트](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-5.html)
