리눅스 커널에서의 시스템 콜. Part 3.
================================================================================

vsyscalls 와 vDSO
--------------------------------------------------------------------------------

이것은 Linux 커널에서 시스템 호출을 설명하는 [챕터](https://junsoolee.gitbook.io/linux-insides-ko/summary/syscall)의 세 번째 파트이며, 사용자 공간 응용 프로그램 이전 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/syscall/linux-syscall-2)에서 시스템 콜 처리 프로세스로 인한 시스템 호출 후의 준비 사항을 확인했습니다. 이 파트에서는 시스템 콜 개념에 매우 가까운 두 가지 개념인 `vsyscall`과 `vdso`를 살펴 보겠습니다.

우리는 이미 `시스템 콜`이 무엇인지 알고 있습니다. 시스템 콜은 사용자 공간 응용 프로그램이 파일 읽기 또는 쓰기, 소켓 열기 등과 같은 권한있는 작업을 수행하도록 요청하는 Linux 커널의 특수 루틴입니다. 아시다시피, 시스템 콜 호출은 Linux에서 비용이 많이 드는 작업입니다. 프로세서는 현재 실행중인 작업을 중단하고 컨텍스트를 커널 모드로 전환해야하기 때문에 시스템 호출 처리기가 작업을 완료 한 후 사용자 공간으로 다시 점프해야합니다. 이 두 가지 메커니즘인 `vsyscall`과 `vdso`는 특정 시스템 호출에 대해 이 프로세스의 속도를 높이도록 설계되었으며 이 부분에서는 이러한 메커니즘의 작동 방식을 이해하려고합니다.

vsyscall 소개
--------------------------------------------------------------------------------

`vsyscall` 또는 `virtual system call` 은 특정 시스템 호출의 실행을 가속화하도록 설계된 Linux 커널에서 최초이자 가장 오래된 메커니즘입니다. `vsyscall` 개념의 작동 원리는 간단합니다. Linux 커널은 일부 변수와 일부 시스템 호출 구현이 포함 된 페이지를 사용자 공간에 매핑합니다. 이 메모리 공간에 대한 정보는 [x86_64](https://en.wikipedia.org/wiki/X86-64)의 Linux 커널 [문서](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt)에서 찾을 수 있습니다. :

```
ffffffffff600000 - ffffffffffdfffff (=8 MB) vsyscalls
```

or:

```
~$ sudo cat /proc/1/maps | grep vsyscall
ffffffffff600000-ffffffffff601000 r-xp 00000000 00:00 0                  [vsyscall]
```

그 후, 이러한 시스템 호출은 사용자 공간에서 실행되며 이는 [컨텍스트 전환]이 없음을 의미합니다. `vsyscall` 페이지의 매핑은 [arch/x86/ entry/vsyscall/vsyscall_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vsyscall/vsyscall_64.c) 소스 코드 파일에 정의 된 `map_vsyscall` 함수에서 발생합니다. 이 함수는 Linux 커널을 초기화하는 동안에 [arch/x86/kernel/setup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup.c) 소스 코드 파일에 정의 된 `setup_arch` 함수에서 호출됩니다 (이 함수는 Linux 커널 초기화의 다섯 번째 [파트]에서 보았습니다).

`map_vsyscall` 함수의 구현은 `CONFIG_X86_VSYSCALL_EMULATION` 커널 설정 옵션에 따라 다릅니다. :

```C
#ifdef CONFIG_X86_VSYSCALL_EMULATION
extern void map_vsyscall(void);
#else
static inline void map_vsyscall(void) {}
#endif
```

도움말 텍스트에서 읽을 수 있듯이 `CONFIG_X86_VSYSCALL_EMULATION` 설정 옵션은 `vsyscall 모방을 가능하게 함` 입니다. 왜 `vsyscall`을 모방할까요? 실제로 `vsyscall`은 보안상의 이유로 레거시 [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)입니다. 가상 시스템 호출의 주소는 고정되어 있습니다. 즉, `vsyscall` 페이지는 매번 같은 위치에 있으며 이 페이지의 위치는 `map_vsyscall` 함수에서 결정됩니다. 이 함수의 구현을 살펴봅시다. :

```C
void __init map_vsyscall(void)
{
    extern char __vsyscall_page;
    unsigned long physaddr_vsyscall = __pa_symbol(&__vsyscall_page);
	...
	...
	...
}
```

보시다시피, `map_vsyscall` 함수의 시작 부분에서 우리는 `__pa_symbol` 매크로를 가진 `vsyscall` 페이지의 물리적 주소를 얻습니다(우리는 이미 리눅스 커널 초기화 프로세스의 네 번째 [파트]에서 이 매크로가 구현 된 것을 보았습니다). 어셈블리 소스 코드 파일 [arch/x86/entry/vsyscall/vsyscall_emu_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vsyscall/vsyscall_emu_64.S)에 정의 된 `__vsyscall_page` 기호는 다음과 같은 [가상 주소](https://en.wikipedia.org/wiki/Virtual_address_space)를 갖습니다. :

```
ffffffff81881000 D __vsyscall_page
```

`.data..page_aligned, aw` [section](https://en.wikipedia.org/wiki/Memory_segmentation)에서 다음 세 가지 시스템 콜에 대한 호출을 포함합니다. :

* `gettimeofday`;
* `time`;
* `getcpu`.

Or:

```assembly
__vsyscall_page:
	mov $__NR_gettimeofday, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_time, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_getcpu, %rax
	syscall
	ret
```

우선 `map_vsyscall` 함수의 구현으로 돌아가고 나중에 `__vsyscall_page`의 구현으로 돌아갑시다. `__vsyscall_page`의 물리적 주소를 수신 한 후 `vsyscall_mode` 변수의 값을 확인하고 `__set_fixmap` 매크로를 사용하여 `vsyscall` 페이지의 [fix-mapped](https://junsoolee.gitbook.io/linux-insides-ko/summary/mm/linux-mm-2) 주소를 설정합니다. :

```C
if (vsyscall_mode != NONE)
	__set_fixmap(VSYSCALL_PAGE, physaddr_vsyscall,
                 vsyscall_mode == NATIVE
                             ? PAGE_KERNEL_VSYSCALL
                             : PAGE_KERNEL_VVAR);
```

`__set_fixmap` 은 세 개의 매개변수를 가집니다.: 첫 번째는 `fixed_addresses` [열거형](https://en.wikipedia.org/wiki/Enumerated_type)의 인덱스입니다. 우리의 경우 `VSYSCALL_PAGE`는 `x86_64` 아키텍처에 대한 `fixed_addresses` 열거형의 첫 번째 원소입니다. :

```C
enum fixed_addresses {
...
...
...
#ifdef CONFIG_X86_VSYSCALL_EMULATION
	VSYSCALL_PAGE = (FIXADDR_TOP - VSYSCALL_ADDR) >> PAGE_SHIFT,
#endif
...
...
...
```

이는 `511`과 같습니다. 두 번째 매개변수는 매핑되어야하는 페이지의 실제 주소이고, 세 번째 매개변수는 페이지의 플래그입니다. `VSYSCALL_PAGE`의 플래그는 `vsyscall_mode` 변수에 의존합니다. `vsyscall_mode` 변수가 `NATIVE`이면 `PAGE_KERNEL_VSYSCALL`이고, 그렇지 않으면 `PAGE_KERNEL_VVAR`입니다. 두 매크로(`PAGE_KERNEL_VSYSCALL` 및`PAGE_KERNEL_VVAR`)는 다음 플래그로 확장됩니다. :

```C
#define __PAGE_KERNEL_VSYSCALL          (__PAGE_KERNEL_RX | _PAGE_USER)
#define __PAGE_KERNEL_VVAR              (__PAGE_KERNEL_RO | _PAGE_USER)
```

이 플래그는 `vsyscall` 페이지에 대한 액세스 권한을 나타냅니다. 두 플래그는 동일한 `_PAGE_USER` 플래그를 가지며 이는 더 낮은 권한 레벨에서 실행되는 사용자 모드 프로세스에 의해 페이지에 액세스 할 수 있음을 의미합니다. 두 번째 플래그는 `vsyscall_mode` 변수의 값에 따라 다릅니다. `vsyscall_mode`가 `NATIVE`인 경우 첫 번째 플래그 (`__PAGE_KERNEL_VSYSCALL`)가 설정됩니다. 즉, 가상 시스템 호출은 기본적으로 `syscall` 명령입니다. 다른 방법으로 vsyscall_mode 변수가 `emulate`이면 vsyscall은 `PAGE_KERNEL_VVAR`을 갖습니다. 이 경우 가상 시스템 호출이 트랩으로 바뀌고 합리적으로 모방됩니다. `vsyscall_mode` 변수는 `vsyscall_setup` 함수에서 값을 가져옵니다.:

```C
static int __init vsyscall_setup(char *str)
{
	if (str) {
		if (!strcmp("emulate", str))
			vsyscall_mode = EMULATE;
		else if (!strcmp("native", str))
			vsyscall_mode = NATIVE;
		else if (!strcmp("none", str))
			vsyscall_mode = NONE;
		else
			return -EINVAL;

		return 0;
	}

	return -EINVAL;
}
```

이는 초기 커널 매개 변수 파싱 중에 호출됩니다. :

```C
early_param("vsyscall", vsyscall_setup);
```

`early_param` 매크로에 대한 자세한 내용은 Linux 커널 초기화 프로세스를 설명하는 챕터의 여섯 번째 파트에서 읽을 수 있습니다.

`vsyscall_map` 함수의 끝에서 `vsyscall` 페이지의 가상 주소가 [BUILD_BUG_ON](https://junsoolee.gitbook.io/linux-insides-ko/summary/initialization/linux-initialization-6) 매크로를 사용하여 `VSYSCALL_ADDR`의 값과 같은지 확인합니다. :

```C
BUILD_BUG_ON((unsigned long)__fix_to_virt(VSYSCALL_PAGE) !=
             (unsigned long)VSYSCALL_ADDR);
```

이게 전부 입니다. `vsyscall` 페이지가 설정되었습니다. 위의 모든 결과는 다음과 같습니다. `vsyscall = native` 매개 변수를 커널 명령 행에 전달하면 가상 시스템 호출은 [arch/x86/entry/vsyscall/vsyscall_emu_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vsyscall/vsyscall_emu_64.S)에서 기본 `syscall` 명령으로 처리됩니다. [glibc](https://en.wikipedia.org/wiki/GNU_C_Library)는 가상 시스템 호출 핸들러의 주소를 알고 있습니다. 가상 시스템 호출 핸들러는`1024`(또는 '0x400`) 바이트로 정렬됩니다. :

```assembly
__vsyscall_page:
	mov $__NR_gettimeofday, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_time, %rax
	syscall
	ret

	.balign 1024, 0xcc
	mov $__NR_getcpu, %rax
	syscall
	ret
```

`vsyscall` 페이지의 시작 주소는 항상 `ffffffffff600000`입니다. 따라서 [glibc](https://en.wikipedia.org/wiki/GNU_C_Library)는 모든 가상 시스템 호출 처리기의 주소를 알고 있습니다. 이 주소들의 정의는 `glibc`소스 코드에서 찾을 수 있습니다. :

```C
#define VSYSCALL_ADDR_vgettimeofday   0xffffffffff600000
#define VSYSCALL_ADDR_vtime 	      0xffffffffff600400
#define VSYSCALL_ADDR_vgetcpu	      0xffffffffff600800
```

모든 가상 시스템 콜 요청은 `__vsyscall_page` + `VSYSCALL_ADDR_vsyscall_name` 오프셋에 속하며, 가상 시스템 호출 수를 `rax` 범용 [레지스터]((https://en.wikipedia.org/wiki/Processor_register)에 넣고 x86_64 `syscall` 명령의 기본 명령이 실행됩니다.

두 번째 경우, `vsyscall = emulate` 매개 변수를 커널 명령 행에 전달하면, 가상 시스템 호출 핸들러는 [page fault](https://en.wikipedia.org/wiki/Page_fault) 예외를 발생시킬 것입니다. 물론, `vsyscall` 페이지에는 실행을 금지하는 `__PAGE_KERNEL_VVAR` 액세스 권한이 있습니다. `do_page_fault` 함수는 `# PF` 또는 페이지 결함 처리기입니다. 마지막 페이지 결함의 원인을 이해하려고 시도합니다. 가상 시스템 콜이 호출되고 `vsyscall`모드가 `emulate`인 상황이 그 원인 중 하나 일 수 있습니다. 이 경우 `vsyscall`은 [arch/x86/entry/vsyscall/vsyscall_64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vsyscall/vsyscall_64.c) 소스 코드 파일에 정의 된 `emulate_vsyscall` 함수에 의해 처리됩니다.

`emulate_vsyscall` 함수는 가상 시스템 호출의 수를 가져 와서 확인한 후, 오류를 출력하고 [세그먼트 폴트](https://en.wikipedia.org/wiki/Segmentation_fault)를 간단히 보냅니다. :

```C
...
...
...
vsyscall_nr = addr_to_vsyscall_nr(address);
if (vsyscall_nr < 0) {
	warn_bad_vsyscall(KERN_WARNING, regs, "misaligned vsyscall...);
	goto sigsegv;
}
...
...
...
sigsegv:
	force_sig(SIGSEGV, current);
	reutrn true;
```

가상 시스템 호출 수를 확인 했으므로 `access_ok` 위반과 같은 또 다른 확인을 수행하고, 가상 시스템 콜 수에 따라 다른 시스템 콜 함수를 실행합니다. :

```C
switch (vsyscall_nr) {
	case 0:
		ret = sys_gettimeofday(
			(struct timeval __user *)regs->di,
			(struct timezone __user *)regs->si);
		break;
	...
	...
	...
}
```

결국 우리는 `sys_gettimeofday` 또는 다른 가상 시스템 호출 핸들러의 결과를 `ax` 범용 레지스터에 넣었다. 우리는 일반적인 시스템 호출로 했던 것처럼 [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) 레지스터를 복원하고 `8` 바이트를 [스택 포인터](https://en.wikipedia.org/wiki/Stack_register) 레지스터에 추가한다. 이 작업은 ret 명령을 에뮬레이트합니다.

```C
	regs->ax = ret;

do_ret:
	regs->ip = caller;
	regs->sp += 8;
	return true;
```		

이것으로 끝입니다. 이제 현대적인 개념인 `vDSO`를 보겠습니다.

vDSO 소개
--------------------------------------------------------------------------------

위에서 이미 언급했듯, `vsyscall`은 이제 쓸모없는 개념이며 `vDSO` 또는 `virtual dynamic shared object`로 대체되었습니다. `vsyscall`과 `vDSO` 메커니즘의 주된 차이점은 `vDSO`는 메모리 페이지를 공유 객체 [form](https://en.wikipedia.org/wiki/Library_%28computing%29#Shared_libraries)의 각 프로세스에 매핑하지만, `vsyscall`은 메모리에서 정적이며 매번 같은 주소를 갖는다는 것입니다. `x86_64` 아키텍처의 경우 이름은 `linux-vdso.so.1`입니다. `glibc`에 동적으로 연결되는 모든 사용자 공간 응용 프로그램은 `vDSO`를 자동으로 사용합니다. 예를 들면 다음과 같습니다. :

```
~$ ldd /bin/uname
	linux-vdso.so.1 (0x00007ffe014b7000)
	libc.so.6 => /lib64/libc.so.6 (0x00007fbfee2fe000)
	/lib64/ld-linux-x86-64.so.2 (0x00005559aab7c000)
```

또는 :

```
~$ sudo cat /proc/1/maps | grep vdso
7fff39f73000-7fff39f75000 r-xp 00000000 00:00 0       [vdso]
```

여기서 [uname](https://en.wikipedia.org/wiki/Uname) util이 세 개의 라이브러리와 연결되어 있음을 알 수 있습니다. :

* `linux-vdso.so.1`;
* `libc.so.6`;
* `ld-linux-x86-64.so.2`.

첫 번째는 `vDSO` 기능을 제공하고, 두 번째는 `C` [표준 라이브러리](https://en.wikipedia.org/wiki/C_standard_library)이고, 세 번째는 프로그램 번역기입니다 (자세한 내용은 [링커](https://junsoolee.gitbook.io/linux-insides-ko/summary/misc/linux-misc-3)에 대해 설명하는 부분에서 읽을 수 있음). 따라서 `vDSO`는 `vsyscall`의 한계를 해결합니다. `vDSO`의 구현은 `vsyscall`과 유사합니다.

`vDSO`의 초기화는 [arch/x86/entry/vdso/vma.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vdso/vma.c) 소스 코드 파일에 정의 된 `init_vdso` 함수에서 시작합니다. 이 함수는 32비트 와 64비트에 대한 `vDSO` 이미지의 초기화에서 시작되며 `CONFIG_X86_X32_ABI` 커널 설정 옵션에 따라 다릅니다. :

```C
static int __init init_vdso(void)
{
	init_vdso_image(&vdso_image_64);

#ifdef CONFIG_X86_X32_ABI
	init_vdso_image(&vdso_image_x32);
#endif
```

두 함수 모두 `vdso_image` 구조체를 초기화합니다. 이 구조체는 생성된 두 소스 코드 파일인 [arch/x86/entry/vdso/vdso-image-64.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vdso/vdso-image-64.c)와 [arch/x86/entry/vdso/vdso-image-32.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vdso/vdso-image-32.c)에 정의되어 있습니다. 다른 소스 코드 파일에서 [vdso2c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vdso/vdso2c.c) 프로그램에 의해 생성 된 이러한 소스 코드 파일들은 `int 0x80`, `sysenter` 등과 같은 시스템 콜을 호출하는 다른 접근 방식을 보여줍니다. 이미지의 전체 세트는 커널 설정에 따라 다릅니다.

예를 들어 `x86_64` Linux 커널의 경우 `vdso_image_64`가 포함됩니다. :

```C
#ifdef CONFIG_X86_64
extern const struct vdso_image vdso_image_64;
#endif
```

But for the `x86` - `vdso_image_32`:

```C
#ifdef CONFIG_X86_X32
extern const struct vdso_image vdso_image_x32;
#endif
```

커널이 `x86` 아키텍처 또는  `x86_64`와 호환성 모드로 설정된 경우 `int 0x80` 인터럽트로 시스템 콜을 호출 할 수 있습니다. 호환성 모드가 활성화 된 경우에는 다른 방식으로 기본 `syscall instruction` 또는 `sysenter` 명령으로 시스템 콜을 호출 할 수 있습니다. :

```C
#if defined CONFIG_X86_32 || defined CONFIG_COMPAT
  extern const struct vdso_image vdso_image_32_int80;
#ifdef CONFIG_COMPAT
  extern const struct vdso_image vdso_image_32_syscall;
#endif
 extern const struct vdso_image vdso_image_32_sysenter;
#endif
```

`vdso_image` 구조체의 이름에서 알 수 있듯이 시스템 콜 엔트리의 특정 모드에 대한 `vDSO` 이미지를 나타냅니다. 이 구조체는 항상 `PAGE_SIZE` (`4096` 바이트)의 배수인 `vDSO` 영역의 바이트 사이즈에 대한 정보, 텍스트 매핑에 대한 포인터, `alternatives`(특정한 타입의 프로세스를 위한 더 나은 대안의 명령어 세트)의 시작과 끝 주소 등을 포함합니다. 예를 들어 `vdso_image_64` 는 다음과 같습니다. :

```C
const struct vdso_image vdso_image_64 = {
	.data = raw_data,
	.size = 8192,
	.text_mapping = {
		.name = "[vdso]",
		.pages = pages,
	},
	.alt = 3145,
	.alt_len = 26,
	.sym_vvar_start = -8192,
	.sym_vvar_page = -8192,
	.sym_hpet_page = -4096,
};
```

`raw_data`는 8 Kilobytes 크기 또는 `2` 페이지 크기인 64 비트 `vDSO` 시스템 호출의 raw 이진 코드를 포함하거나 :
Where the `raw_data` contains raw binary code of the 64-bit `vDSO` system calls which are `2` page size:

```C
static struct page *pages[2];
```

`init_vdso_image` 함수는 동일한 소스 코드 파일에 정의되어 있으며 `vdso_image.text_mapping.pages` 만 초기화합니다. 우선 이 함수는 페이지 수를 계산하고 주어진 주소를 `page` 구조체로 변환하는 `virt_to_page` 매크로로 각 `vdso_image.text_mapping.pages [number_of_page]`를 초기화합니다.


```C
void __init init_vdso_image(const struct vdso_image *image)
{
	int i;
	int npages = (image->size) / PAGE_SIZE;

	for (i = 0; i < npages; i++)
		image->text_mapping.pages[i] =
			virt_to_page(image->data + i*PAGE_SIZE);
	...
	...
	...
}
```

`subsys_initcall` 매크로에 전달 된 `init_vdso` 함수는 주어진 함수를 `initcalls` 목록에 추가합니다. 이 목록의 모든 함수는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c) 소스 코드 파일의 `do_initcalls` 함수에서 호출됩니다. :

```C
subsys_initcall(init_vdso);
```

자, 우리는 `vDSO`의 초기화와 `vDSO` 시스템 호출을 포함하는 메모리 페이지와 관련된 `page` 구조체의 초기화를 보았습니다. 그렇다면 이들의 페이지는 어디로 매핑될까요? 실제로 바이너리를 메모리에 로드할 때 커널에 의해 매핑됩니다. 리눅스 커널은 [arch/x86/entry/vdso/vma.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/vdso/vma.c) 소스 코드 파일에서 `arch_setup_additional_pages` 함수를 호출하여 `x86_64`에 대해 `vDSO`가 활성화되어 있는지 확인하고 `map_vdso` 함수를 호출합니다. :

```C
int arch_setup_additional_pages(struct linux_binprm *bprm, int uses_interp)
{
	if (!vdso64_enabled)
		return 0;

	return map_vdso(&vdso_image_64, true);
}
```

`map_vdso` 함수는 동일한 소스 코드 파일에서 정의되며 `vDSO`와 공유 `vDSO` 변수에 대한 페이지를 매핑합니다. 이것으로 끝입니다. `vsyscall`과 `vDSO` 개념의 주요 차이점은 `vsyscall`은 고정 주소 `ffffffffff600000`을 가지며 `3` 시스템 콜을 구현하는 반면,`vDSO`는 동적으로 로드되고 4 개의 시스템 콜을 구현한다는 것입니다. :

* `__vdso_clock_gettime`;
* `__vdso_getcpu`;
* `__vdso_gettimeofday`;
* `__vdso_time`.


끝났습니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널의 시스템 호출 개념에 대한 세 번째 파트의 끝이 났습니다. 이전의 [파트](https://junsoolee.gitbook.io/linux-insides-ko/summary/syscall/linux-syscall-2)에서 시스템 호출 전에 Linux 커널에서의 준비 구현과, 시스템 콜 핸들러에서의 `exit` 프로세스의 구현에 대해 논의했습니다. 이 파트에서 우리는 계속해서 시스템 콜 개념과 관련된 내용을 살펴보고 시스템 콜과 매우 유사한 두 가지 새로운 개념인 `vsyscall` 과 `vDSO`를 배웠습니다.

이 세 파트을 모두 진행한 후에는 시스템 콜과 관련된 거의 모든 사항을 알고 시스템 콜이 무엇이며 사용자 응용 프로그램에 필요한 이유를 알고 있습니다. 또한 사용자 응용 프로그램이 시스템 콜을 호출 할 때 발생하는 일과 커널이 시스템 콜을 처리하는 방법도 알고 있습니다.

다음 파트는 이 [챕터](https://junsoolee.gitbook.io/linux-insides-ko/summary/syscall)의 마지막 파트이며 사용자가 프로그램을 실행할 때 어떤 일이 발생하는지 볼 수 있습니다.

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견이나 핑을 남겨주시거나. [email](anotherworldofworld@gmail.com) 보내주시거나, [issue](https://github.com/0xAX/linux-insides/issues/new)를 만들어주세요.

**영어는 제 모국어가 아닙니다. 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수를 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한글 번역](https://github.com/junsooo/linux-insides-ko)으로 PR을 보내주세요.**

Links
--------------------------------------------------------------------------------

* [x86_64 memory map](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/x86/x86_64/mm.txt)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [context switching](https://en.wikipedia.org/wiki/Context_switch)
* [ABI](https://en.wikipedia.org/wiki/Application_binary_interface)
* [virtual address](https://en.wikipedia.org/wiki/Virtual_address_space)
* [Segmentation](https://en.wikipedia.org/wiki/Memory_segmentation)
* [enum](https://en.wikipedia.org/wiki/Enumerated_type)
* [fix-mapped addresses](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-2.html)
* [glibc](https://en.wikipedia.org/wiki/GNU_C_Library)
* [BUILD_BUG_ON](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)
* [Processor register](https://en.wikipedia.org/wiki/Processor_register)
* [Page fault](https://en.wikipedia.org/wiki/Page_fault)
* [segmentation fault](https://en.wikipedia.org/wiki/Segmentation_fault)
* [instruction pointer](https://en.wikipedia.org/wiki/Program_counter)
* [stack pointer](https://en.wikipedia.org/wiki/Stack_register)
* [uname](https://en.wikipedia.org/wiki/Uname)
* [Linkers](https://0xax.gitbooks.io/linux-insides/content/Misc/linux-misc-3.html)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-2.html)
