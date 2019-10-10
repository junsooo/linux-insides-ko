커널 부팅 과정. Part 3.
===============================================================================

비디오 모드의 초기화와 보호 모드로 전환하기
--------------------------------------------------------------------------------

이것은 커널 부팅 과정 시리즈의 세번째 파트입니다. 이전 [파트](linux-bootstrap-2.md#kernel-booting-process-part-2)에서는 [main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)에서 `set_video`루틴을 호출하기 직전에 끝났습니다. 

이번 파트에서 볼 것: 

* 커널 설정 코드에서 비디오 모드 초기화하기
* 보호 모드로 전환하기 전에 준비할 것
* 보호 모드로 전환하기

**참고** 보호 모드가 무엇인지 모르는 분은 이전 [파트](linux-bootstrap-2.md#protected-mode)에서 정보를 찾아볼 수 있습니다. 또한 도움을 줄 수 있는 몇 가지  [링크](linux-bootstrap-2.md#links)가 있습니다.

소스코드파일[arch/x86/boot/video.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video.c)에 정의되어 있는 `set_video` 함수에서 시작합니다. 제일 먼저 `boot_params.hdr`구조체에서 비디오 모드를 가져옵니다.


```C
u16 mode = boot_params.hdr.vid_mode;
```


`copy_boot_params` 함수로 값을 채웠었습니다(이전 포스트 참조). `vid_mode`는 부트로더로 인해 채워지는 필수 필드입니다. `boot protocol`커널에서 이에 대한 정보를 찾을 수 있습니다.

```
Offset	Proto	Name		Meaning
/Size
01FA/2	ALL	    vid_mode	Video mode control
```

리눅스 커널 부트 프로토콜에서 읽을 수 있습니다:

```
vga=<mode>
	<mode> 이것은 정수 (C내부의 표기,
	10진법, 8진법, 또는 16진법 중에서) 또는 하나의 스트링
	"normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) 또는 "ask"
	(meaning 0xFFFD) 중 하나.  이 값은 vid_mode field에 넣어져야
        한다, 그래서 이것은 커맨드라인을 분석하기 전에 커널에서 사용된다.
```

따라서 `vga` 옵션을 grub(또는 다른 부트로더) 구성 파일에 추가할 수 있으며, 이 옵션을 커널 커맨드 라인에 전달할 수 있습니다. 이 옵션은 설명에 언급 된대로 다른 값을 가질 수 있습니다. 예를 들어 정수 `0xFFFD`이나 `ask`를 사용할 수 있습니다. `vga`에 `ask`를 전달하면 다음과 같은 메뉴가 나옵니다:

![비디오 모드 설정 메뉴](images/video_mode_setup_menu.png)

비디오 모드를 선택하라는 메시지가 표시됩니다.  구현한 것을 살펴보기 전에 봐야하는 것들이 있습니다.

커널 데이터 타입
--------------------------------------------------------------------------------

이전 커널 설정 코드에서 `u16`같은 다른 데이터 타입의 정의를 봤습니다. 커널이 제공하는 몇가지 데이터 타입을 살펴봅시다.


| Type | char | short | int | long | u8 | u16 | u32 | u64 |
|------|------|-------|-----|------|----|-----|-----|-----|
| Size |  1   |   2   |  4  |   8  |  1 |  2  |  4  |  8  |

이 커널의 소스코드들은 매우 자주 보게 될 것이니 기억하는 것이 좋습니다.

힙 API
--------------------------------------------------------------------------------

다음으로 `boot_params.hdr`의 `set_video`함수에서 `vid_mode`를 가져와 `RESET_HEAP` 함수를 호출하는 것을 볼 수 있습니다. `RESET_HEAP`은 [arch/x86/boot/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/boot.h)헤더 파일에서 정의된 매크로입니다.

이 매크로는 다음과 같이 정의됩니다:

```C
#define RESET_HEAP() ((void *)( HEAP = _end ))
```

두 번째 파트를 읽었다면 [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)함수를 사용해 힙이 초기화되는 것을 기억할 것입니다. `arch/x86/boot/boot.h`에 힙을 관리하기 위해 정의된 유틸리티 매크로와 함수가 있습니다.

그것들은:

```C
#define RESET_HEAP()
```

위에서 본 것처럼 `HEAP`의 `_end`변수를 설정함으로써 힙을 초기화합니다. `_end`는 `extern char _end[];`에 있습니다.


다음은 `GET_HEAP`매크로입니다:

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
```

힙을 할당하기 위해 `__get_heap`에 3개의 매개변수를 넣어 내부함수를 호출합니다:

* 할당할 데이터 타입의 크기
* `__alignof__(type)`은 어떻게 이 유형의 변수를 정렬할 것인지
* `n`은 할당할 항목의 수

`__get_heap`의 구현:

```C
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
```

다음과 같은 사용법을 볼 수 있습니다:

```C
saved.data = GET_HEAP(u16, saved.x * saved.y);
```

`__get_heap`이 어떻게 작동하는지 이해해봅시다. `HEAP`()의 매개변수 `a`에 따라 정렬된 메모리의 주소가 할당된 것을 알 수 있습니다. 다음으로 `HEAP`에서  `tmp`변수의 메모리주소를 저장하면 `HEAP`은 할당된 블록의 끝으로 이동하고 할당된 메모리의 시작 주소에 `tmp`가 반환됩니다. 


마지막 기능은 다음과 같습니다.

```C
static inline bool heap_free(size_t n)
{
	return (int)(heap_end - HEAP) >= (int)n;
}
```

(이전 [파트](linux-bootstrap-2.md)에서 계산)`heap_end`에서 `HEAP`포인터의 값을 빼고 사용할 수 있는 메모리가 충분하면 1을 반환합니다.

이제 힙에대한 간단한 API가 있으면 비디오 모드를 설정할 수 있습니다.

비디오 모드 설정
--------------------------------------------------------------------------------

이제 비디오 모드의 초기화로 직접 이동할 수 있습니다. `RESET_HEAP()`에서 `set_video`함수를 호출하고 멈췄습니다. 그 다음은 [include/uapi/linux/screen_info.h](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/screen_info.h)헤더 파일에 정의된 `boot_params.screen_info`구조에 비디오 모드 파라미터를  저장하는 `store_mode_params`의 호출입니다.

`store_mode_params`함수를 보면 `store_cursor_position`함수를 호출하며 시작하는 것을 볼 수 있습니다. 함수 이름에서 알 수 있듯이 커서에 대한 정보를 가져와서 저장합니다.

첫째로 `store_cursor_position`가 가진 `biosregs`와 함께 `AH = 0x3` 두 변수를 초기화하고 `0x10` BIOS 인터럽트를 호출합니다. 인터럽트가 성공적으로 실행되면 `DL`과 `DH`레지스터의행렬을 반환합니다. 행렬은 `boot_params.screen_info`구조의 `orig_x`, `orig_y`필드를 저장할 수 있습니다.

`store_cursor_position`의 실행 후 `store_video_mode`함수가 호출됩니다. 이것은 현재 비디오 모드를 얻고 `boot_params.screen_info.orig_video_mode`를 저장합니다.

그런 다음, `store_mode_params`는 현재 비디오 모드를 체크하고 `video_segment`를 설정합니다. BIOS가 부트 섹터 컨트롤을 전송한 후 따라오는 주소는 비디오 메모리 용입니다:

```
0xB000:0x0000 	32 Kb 	Monochrome Text Video Memory
0xB800:0x0000 	32 Kb 	Color Text Video Memory
```

따라서 현재 비디오 모드의 MDA, HGC 또는 VGA가 모노크롬 모드이고 `0xb800`이거나 현재 비디오 모드가 컬러 모드이면 `video_segment`변수를 `0xb000`로 설정합니다. 비디오 세그먼트의 주소를 설정했으면 `boot_params.screen_info.orig_video_points`에 폰트 사이즈를 저장해야 합니다.

```C
set_fs(0);
font_size = rdfs16(0x485);
boot_params.screen_info.orig_video_points = font_size;
```

첫번째로 `set_fs`함수를 이용해 `F5`레지스터에 0을 넣습니다. 이전 파트에서 `set_fs`같은 함수는 이미 봤습니다. 이것들은 모두 [arch/x86/boot/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/boot.h)에 정의되어 있습니다. 다음으로 주소 `0x485`(이 메모리 위치는 폰트 사이즈를 가져오는데 사용)에서 값을 읽고 `boot_params.screen_info.orig_video_points`에 폰트 사이즈를 저장합니다.

```C
x = rdfs16(0x44a);
y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;
```

다음으로 주소`0x44a`에서 열의 개수를, 주소`0x484`에서 행의 개수를 가져오고 그것들을 `boot_params.screen_info.orig_video_cols`과 `boot_params.screen_info.orig_video_lines`에 저장합니다. 그 다음 `store_mode_params`을 실행하면 끝입니다.

다음으로 화면의 내용을 힙에 저장하는 `save_screen`함수입니다. 이 함수는 이전의 함수에서 얻은 모든 데이터(행과 열, 재료 등)를 수집하여 다음과 같이 정의된 `saved_screen`구조체에 저장합니다.

```C
static struct saved_screen {
	int x, y;
	int curx, cury;
	u16 *data;
} saved;
```

그런 다음 힙에 여유 공간이 있는지 확인합니다.

```C
if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
		return;
```

공간이 충분하면 힙에 공간을 할당하고 `saved_screen`에 저장합니다.

다음 호출은 [arch/x86/boot/video-mode.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video-mode.c)소스 코드 파일의 `probe_cards(0)`입니다. 이것은 모든 비디오 카드를 지나가며 카드가 제공하는 모드의 개수를 수집합니다. 다음은 흥미로운 부분으로 루프를 볼 수 있습니다. 

```C
for (card = video_cards; card < video_cards_end; card++) {
  /* collecting number of modes here */
}
```

그러나 `video_cards`가 어디에도 선언되지 않았습니다. 답은 간단합니다. x86 커널 설정 코드에 표시되는 모든 비디오 모드는 다음과 같은 정의를 가집니다.

```C
static __videocard video_vga = {
	.card_name	= "VGA",
	.probe		= vga_probe,
	.set_mode	= vga_set_mode,
};
```

`__videocard`매크로의 위치:

```C
#define __videocard struct card_info __attribute__((used,section(".videocards")))
```

이는 `card_info`구조체를 의미합니다.

```C
struct card_info {
	const char *card_name;
	int (*set_mode)(struct mode_info *mode);
	int (*probe)(void);
	struct mode_info *modes;
	int nmodes;
	int unsafe;
	u16 xmode_first;
	u16 xmode_n;
};
```

`.videocards`세그먼트에서 우리가 찾을 수 있는 링커 스크립트[arch/x86/boot/setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)를 살펴봅시다.

```
	.videocards	: {
		video_cards = .;
		*(.videocards)
		video_cards_end = .;
	}
```

이것은 `video_cards`는 단순한 메모리 주소이며 모든 `card_info`구조체는 세그먼트에 배치된다는 것을 의미합니다. 모든 `card_info` 구조체는 `video_cards`와 `video_cards_end` 사이에 배치되므로 루프를 사용하여 모든 구조체를 살핍니다. `probe_cards`가 실행되면 `static __videocard video_vga`이나 `nmodes`(비디오 모드의 수)가 채워진 것과 같은 구조체들을 가집니다.


다음으로 `probe_cards`함수를 수행하면 `set_video`함수의 메인루프로 이동합니다. 커널 커맨드 라인에 `vid_mode=ask`가 전달되거나 비디오 모드가 정의되지 않은 경우 `set_mode`함수로 비디오 모드를 설정하거나 메뉴를 출력하는 무한루프가 있습니다.

`set_mode`함수는 [video-mode.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video-mode.c)에 정의되어 있으며 매개변수로 비디오 모드의 개수인 `mode`하나만 가져옵니다(이 값은 메뉴 또는 커널 설정 헤더의 시작부분 `setup_video`에서 얻음).


`set_mode`함수는 `mode`를 확인하고 `raw_set_mode`함수를 호출합니다. `raw_set_mode`는 `set_mode`함수에서 선택된 카드 `card->set_mode(struct mode_info*)`를 호출합니다. `card_info`구조체 함수에 접근할 수 있습니다. 모든 비디오 모드는 비디오 모드에 의해 채워진 값으로 구조체를 정의합니다.(예를 들어 `vga`는 `video_vga.set_mode`함수입니다. 위의 예시를 보면 `card_info`구조체는 `vga`입니다). `video_vga.set_mode`는 `vga_set_mode`이므로 vga 모드를 확인하고 해당 함수를 호출합니다.

```C
static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}
```

비디오 모드를 설정하는 모든 함수는 `AH`레지스터의 특정 값으로 `0x10` BIOS 인터럽트를 호출합니다.

비디오 모드를 설정한 다음 `boot_params.hdr.vid_mode`를 전달합니다.

다음 `vesa_store_edid`이 호출됩니다. 이 함수는 커널의 사용 정보 [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) (**E**xtended **D**isplay **I**dentification **D**ata)를 간단하게 저장합니다. 다음으로 `store_mode_params`을 다시 호출합니다. 마지막으로 `do_restore`을 설정하면 화면이 이전 상태로 복원됩니다.

이 작업을 완료하면 비디오 모드 설정이 완료되었으며 이제 보호 모드로 전환할 수 있습니다.

보호 모드로 전환하기 전 마지막 준비
--------------------------------------------------------------------------------

[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)의 `go_to_protected_mode`에서 마지막 함수 호출을 볼 수 있습니다. 주석:`Do the last things and invoke protected mode`에서 알 수 있듯이 마지막 사항을 확인하고 보호모드로 전환하십시오.


`go_to_protected_mode`함수는 [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pm.c)에 정의되어 있습니다. 여기에는 보호 모드로 들어가기 전에 마지막으로 준비해야 하는 함수가 포함되어 있으니 함수를 살펴보고 작동 방식을 이해하려고 노력하십시오.

먼저 `go_to_protected_mode`의 `realmode_switch_hook`함수를 호출합니다. 이 함수는 리얼 모드 스위치 후크가 있으면 이를 호출하고 [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt)를 비활성화합니다. 부트로더가 적대적인 환경에서 실행되는 경우 후크가 사용됩니다. [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)에서 후크에 대한 자세한 내용을 볼 수 있습니다(**고급 부트 로더 후크**참조).


`realmode_switch`후크는 마스크 불가능 인터럽트를 비활성화하는 16 비트 리얼 모드 원거리 서브루틴에 대한 포인터를 제공합니다. `realmode_switch`후크를 확인한 후, 마스크 불가능 인터럽트(NMI)는 사용할 수 없습니다.

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

처음에 인터럽트 플래그 (`IF`)를 지우는 인라인 어셈블리 명령문 `cli`가 있습니다. 다음으로 외부 인터럽트가 비활성화됩니다. 다음 줄은 NMI(마스크 불가능 인터럽트)를 비활성화합니다.

인터럽트는 하드웨어나 소프트웨어에 의해 발생하는 CPU신호 입니다. 신호를 얻은 후 CPU는 현재 명령 시퀀스를 중단하고 상태를 저장하며, 컨트롤을 인터럽트 핸들러로 전송합니다. 다음으로 인터럽트 핸들러는 작업을 완료하고 컨트롤을 다시 인터럽트 명령으로 전송합니다. 마스크 불가능 인터럽트(NMI)는 권한에 관계없이 항상 실행되는 인터럽트입니다. 이것들은 무시할 수 없고 일반적으로 복구할 수 없는 하드웨어 오류를 나타내는데 사용됩니다. 지금은 인터럽트의 세부사항에 들어가지 않고 다음 포스트에서 논의할 것입니다.

다시 코드로 옵니다. 두번째 줄에서 바이트`0x80`(비활성화된 바이트)를 `0x70`(CMOS 주소 레지스터)에 쓰는 것을 볼 수 있습니다. 그 후 `io_delay`함수 호출이 발생합니다. `io_delay`에서 다음과 같이 약간의 딜레이가 있습니다:

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```

`0x80`포트에 바이트를 출력하면 정확히 1마이크로 초가 지연되어야 합니다. 따라서 어떤 값(이 경우 AL)이라도 `0x80`포트에 쓸 수 있습니다. 이 딜레이 후에 `realmode_switch_hook`함수의 실행이 끝나고 다음 함수로 넘어갈 수 있습니다.

다음 함수는 `enable_a20`으로 [A20 line](http://en.wikipedia.org/wiki/A20_line)에서 활성화 할 수 있습니다. 이 함수는 [arch/x86/boot/a20.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/a20.c)에 정의되었으며 다른 방법으로 A20 게이트를 활성화하려 합니다. 첫번째로 `a20_test_short`함수는 `a20_test`함수와 함께 A20의 활성화 여부를 확인합니다:

```C
static int a20_test(int loops)
{
	int ok = 0;
	int saved, ctr;

	set_fs(0x0000);
	set_gs(0xffff);

	saved = ctr = rdfs32(A20_TEST_ADDR);

        while (loops--) {
		wrfs32(++ctr, A20_TEST_ADDR);
		io_delay();	/* Serialize and make delay constant */
		ok = rdgs32(A20_TEST_ADDR+0x10) ^ ctr;
		if (ok)
			break;
	}

	wrfs32(saved, A20_TEST_ADDR);
	return ok;
}
```

우선 `FS`레지스터에 `0x0000`을 넣고 `GS`레지스터에 `0xffff`를 넣습니다. 다음으로 주소`A20_TEST_ADDR`(`0x200`)에서 값을 읽고 이 값을 변수`saved`와 `ctr`에 넣습니다.

다음, `wrfs32`함수로 업데이트된 `ctr`값을 `fs:A20_TEST_ADDR` 또는 `fs:0x200`에 넣습니다. 1마이크로초 뒤 주소 `A20_TEST_ADDR+0x10`에서 `GS`레지스터의 값을 읽습니다. `a20` 라인이 비활성화된 경우 주소가 겹치며,  `a20`라인이 0이 아닌 경우 이미 A20라인이 활성화 된 것입니다.

A20이 비활성화된 경우 `a20.c`에서 다른 방법을 찾아 활성화해야 합니다. 예를 들어, `AH=0x2041`를 써서 `0x15`BIOS 인터럽트를 호출할 수 있습니다.

`enable_a20`함수가 실패로 끝나면 에러 메시지를 출력하고 `die`함수를 호출합니다. 첫 소스 코드 파일[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)을 기억해보십시오.

```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

A20 게이트가 성공적으로 활성화되면 `reset_coprocessor`함수가 호출됩니다:

```C
outb(0, 0xf0);
outb(0, 0xf1);
```

이 함수는 `0xf0`에 `0`을 적어서 수학 보조프로세서를 지우고 `0xf1`에 `0`을 적어서 초기화합니다.

그 다음 `mask_all_interrupts`함수가 호출됩니다:

```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```

이 마스크는 1차 PIC의 IRQ2를 제외한 2차 PIC(Programmable Interrupt Controller)와 1차 PIC의  모든 인터럽트를 마스크합니다.

모든 준비 후에 실제 보호 모드로 전환되는 것을 볼 수 있습니다.

인터럽트 설명자 테이블 설정
--------------------------------------------------------------------------------

이제 `setup_idt`함수로 인터럽트 디스크립터 테이블(IDT)을 설정합니다.

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

인터럽트 디스크립터 테이블을 설정합니다(인터럽트 핸들러 등을 설명). 현재는 IDT가 설치되지 않았지만(나중에 볼 예정), `lidtl` 명령과 IDT를 불러오면 됩니다. `null_idt`는 주소와 IDT의 크기를 포함하지만 현재는 0입니다. `null_idt`는 `gdt_ptr`구조체로 다음과 같이 정의됩니다:

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

여기서 16비트 길이(`len`)의 IDT와 이에 대한 32비트 포인터를 볼 수 있습니다(IDT와 인터럽션에 대해 자세한 내용은 다음 포스트에서 볼 수 있습니다). ` __attribute__((packed))`는 `gdt_ptr`가 요구하는 최소한의 크기를 의미합니다. 따라서 `gdt_ptr`의 크기는 여기서도 6바이트 또는 48비트입니다. (다음에 `GDTR`레지스터에서 `gdt_ptr`포인터를 부르고 이전 포스트에서 이것이 48비트라고 한 것을 기억할 수도 있습니다).

글로벌 디스크립터 테이블 설정
--------------------------------------------------------------------------------

다음은 글로벌 디스크립터 테이블(GDT)의 설정입니다. GDT를 설정한 `setup_gdt`함수를 볼 수 있습니다(포스트 [커널 부팅 과정. 파트 2.](linux-bootstrap-2.md#protected-mode)에서 읽을 수 있습니다). `boot_gdt`배열 함수의 정의는 세그먼트 3개의 정의를 포함합니다:

```C
static const u64 boot_gdt[] __attribute__((aligned(16))) = {
	[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
	[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
};
```

코드, 데이터 및 TSS(Task State Segment)에서 현재 작업 상태 세그먼트를 사용하지 않을 것입니다. 주석 행에서 볼 수 있듯이 이것은 Intel VT를 적절히하기 위해 추가되었습니다(관심이 있는 경우 [여기](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31)에서 설명하는 커밋을 찾을 수 있습니다). `boot_gdt`을 봅시다. 우선 `__attribute__((aligned(16)))`속성이 있습니다. 이는 이 구조체가 16바이트로 정렬되는 것을 의미합니다.

간단한 예제를 봅시다:

```C
#include <stdio.h>

struct aligned {
	int a;
}__attribute__((aligned(16)));

struct nonaligned {
	int b;
};

int main(void)
{
	struct aligned    a;
	struct nonaligned na;

	printf("Not aligned - %zu \n", sizeof(na));
	printf("Aligned - %zu \n", sizeof(a));체

	return 0;
}
```

기술적으로 `int`필드를 포함하는 구조체는 4바이트여야 하지만 `aligned`구조체는 메모리에 저장하기 위해 16바이트가 필요합니다.

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

`GDT_ENTRY_BOOT_CS`는 인덱스 2개, `GDT_ENTRY_BOOT_DS`는 `GDT_ENTRY_BOOT_CS + 1` 등이 있습니다. 첫번째는 필수 널 디스크립터(index - 0)여서 2에서 시작하고 두번째는 사용하지 않습니다(index -1).

`GDT_ENTRY`는 flags, base, limit 및 GDT 항목을 작성하는 매크로입니다. 예를 들어, 코드 세크먼트 항목을 살펴봅시다. `GDT_ENTRY`는 다음 값을 갖습니다:

* base  - 0
* limit - 0xfffff
* flags - 0xc09b

이것은 무엇을 의미하는가? 세크먼트의 기본 주소는 0이고, limit(세그먼트의 크기)는 `0xfffff`(1MB)입니다. flags를 봅시다. flags는 `0xc09b`이고 다음과 같습니다:

```
1100 0000 1001 1011
```

이진법으로 모든 비트의 의미를 이해해봅시다. 모든 비트를 왼쪽에서 오른쪽으로 살펴봅시다.

* 1    - (G) granularity bit
* 1    - (D) 16비트 세그먼트인 경우; 1 = 32비트 세그먼트
* 0    - (L) 1인 경우 64비트 모드에서 실행
* 0    - (AVL) 시스템 소프트웨어에서 사용할 수 있는
* 0000 - 디스크립터에서 4비트의 길이는 19:16비트
* 1    - (P) 메모리에서 세그먼트의 존재
* 00   - (DPL) - 권한 레벨, 0이 가장 높은 권한
* 1    - (S) 시스템 세그먼트가 아닌 코드나 데이터 세그먼트
* 101  - 세그먼트 타입 실행/읽기/
* 1    - 액세스 된 비트

이전 [포스트](linux-bootstrap-2.md) 또는 [인텔® 64 and IA-32 아키텍쳐 소프트웨어 개발자 메뉴얼 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)에서 모든 비트에 대한 자세한 내용을 읽을 수 있습니다.

다음과 같이 GDT의 길이를 얻습니다:

```C
gdt.len = sizeof(boot_gdt)-1;
```

`boot_gdt`의 크기를 얻고 1을 뺍니다(GDT의 마지막 유효주소).

다음으로 GDT의 포인터를 얻습니다:

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

`boot_gdt`의 주소를 얻고 왼쪽으로 4비트 이동한 데이터 세그먼트의 주소를 추가합니다(현재 리얼모드인 것을 기억하십시오).

마지막으로 GDT를 GDTR레지스터에 로드하는 `lgdtl`명령을 실행합니다:

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

보호 모드로의 실제 전환
--------------------------------------------------------------------------------

이것이 `go_to_protected_mode`함수의 끝입니다. IDT 및 GDT를 로드하고 인터럽트를 비활성화했으며 이제 CPU를 보호모드로 전환할 수 있습니다. 마지막 단계는 `protected_mode_jump`함수와 두 매개변수를 호출하는 것입니다.

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

[arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S)에 정의되어 있습니다.

두 매개변수가 하는 일은:

* 보호 모드 진입 포인트의 주소
* `boot_params`의 주소

`protected_mode_jump`의 내부를 봅시다. 위에 쓴 것처럼 `arch/x86/boot/pmjump.S`를 찾을 수 있습니다. 첫번째 매개변수는 `eax`레지스터에 있고 두번째는 `edx`에 있습니다.

먼저, `boot_params`의 주소에 `esi`레지스터를 넣고 코드 세그먼트 레지스터`cs`에 `bx`를 넣습니다. 그런 다음 `bx`를 4비트 씩 이동해 레이블`2`로 지정된 메모리위치(즉, `(cs << 4) + in_pm32`를 32비트 모드로 전환하고 옮길 물리 주소)에 추가하고 레이블`1`로 이동합니다. 따라서 레이블`2`의 `in_pm32`은 `(cs << 4) + in_pm32`로 덮어 쓰여집니다.

다음으로 데이터 세그먼트와 작업 상태 세그먼트를 `cx`와 `di`레지스터에 넣습니다:

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

위에서 읽은 것처럼 `GDT_ENTRY_BOOT_CS`는 인덱스 2개를 가졌으며 모든 GDT항목이 8바이트입니다. 따라서 `CS`는 `2 * 8 = 16`이 되고 `__BOOT_DS`는 24입니다.

다음으로 `CR0`제어 레지스터에서 `PE`(보호 활성화) 비트를 설정합니다:

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

보호 모드로 이동합니다:

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS
```

위치:

* `0x66`은 16비트와 32비트를 혼합할 수 있는 피연산자 크기 접두사입니다. 
* `0xea` - opcode로 이동
* `in_pm32`은 보호모드에서 세그먼트 오프셋이며 실제 모드에서 파생된 값`(cs << 4) + in_pm32`이 있습니다.
* `__BOOT_CS`는 이동하려는 코드 세그먼트 입니다.

다음은 드디어 보호 모드입니다:

```assembly
.code32
.section ".text32","ax"
```

보호 모드에서 수행된 첫 번째 단계를 살펴봅시다. 먼저 다음과 같이 데이터 세그먼트를 설정합니다:

```assembly
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

관심이 있다면 `cx`레지스터의 `$__BOOT_DS`에 저장한 것을 기억할 것입니다. 이제 `cs`(`cs`는 이미 `__BOOT_CS`)이외의 모든 세그먼트 레지스터를 채웁니다.

그리고 디버깅 목적으로 유효한 스택을 설정합니다:

```assembly
addl	%ebx, %esp
```

32비트 진입 포인트로 이동하기 전 마지막 단계는 범용 레지스터를 지우는 것입니다:

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

그리고 마지막으로 32비트 진입 포인트로 이동합니다:

```
jmpl	*%eax
```

`eax`는 32비트 진입 주소를 포함하는 것을 기억하십시오(`protected_mode_jump`의 첫번째 매개변수로 전달).

끝났습니다. 보호 모드에 들어왔고 진입 포인트는 멈춥니다. 다음에 일어날 일은 다음 파트에서 보겠습니다.

결론
--------------------------------------------------------------------------------

리눅스 커널 내부에 대한 세번째 파트의 끝입니다. 다음 파트에서는 보호 모드에서 시작하여 [long mode](http://en.wikipedia.org/wiki/Long_mode)로 전환하는 첫 번째 단계를 살펴보겠습니다.

질문이나 제안 사항이 있다면 코멘트를 남기거나 [트위터](https://twitter.com/0xAX)로 보내주세요.

**영어는 모국어가 아니며 모든 불편한 점은 정말 죄송합니다. 실수를 발견하면 [linux-insides](https://github.com/0xAX/linux-internals)에서 수정 사항이 포함된 PR을 보내주십시오.**

링크
--------------------------------------------------------------------------------

* [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
* [VESA BIOS 확장](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
* [자료 구조 정렬](http://en.wikipedia.org/wiki/Data_structure_alignment)
* [마스크 불가능 인터럽트](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [A20](http://en.wikipedia.org/wiki/A20_line)
* [GCC 지정 초기화](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
* [GCC 타입 속성](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
* [이전 파트](linux-bootstrap-2.md)
