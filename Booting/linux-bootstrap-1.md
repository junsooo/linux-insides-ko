커널 부팅 과정. Part 1
================================================================================

부트로더에서 커널까지
--------------------------------------------------------------------------------

만약 당신이 제 이전 [블로그 게시물](https://0xax.github.io/categories/assembler/)들을 계속 읽어왔다면, 제가 꽤 오랜 시간동안 로우레벨 프로그래밍을 해왔다는 것을 알 수 있었을 것입니다. 저는 x86_64 리눅스에 대한 어셈블리 프로그래밍에 대한 여러 글들을 작성해왔고, 그와 동시에 리눅스 커널 소스 코드에 빠져들게 되었습니다.

저는 로우레벨이 어떻게 내부적으로 동작하는지, 프로그램이 컴퓨터에서 어떻게 실행이 되는지, 어떻게 그것들이 메모리에 적재되는지, 커널이 프로세스와 메모리 관리를 어떻게 하는지, 네트워크 스택이 로우레벨에서 어떻게 동작하는지, 그리고 이외의 많은 것들을 이해하는데 큰 관심을 가지고 있습니다. 그래서 저는 **x86_64** 아키텍쳐 리눅스 커널에 대해 게시물 시리즈를 작성하기로 결정하였습니다.

제가 전문적인 커널 해커가 아니고 일하면서 커널 코드를 작성하지 않는다는 것을 알아두세요. 이건 취미일 뿐이고, 저는 단지 로우레벨과 이것들이 어떻게 동작하는지 흥미가 있기 때문에 하는 것입니다. 만약에 글을 읽다가 헷갈리는 것이 생기거나 궁금한 사항 등이 생기면 트위터에서 [0xAX](https://twitter.com/0xAX)를 핑하거나 [이메일](anotherworldofworld@gmail.com)을 보내거나 아니면 [이슈]를 만들어주세요. 감사히 받겠습니다.

모든 게시물들은 [github repo](https://github.com/0xAX/linux-insides)에서 볼 수 있습니다. 그리고 제 영어 실력이나 게시물 내용에서 이상한 것을 찾으면 언제든 풀 리퀘스트를 보내주세요.

*이 게시물은 공식 문서가 아니며, 단지 지식을 배우고 공유하기 위한 것임을 알아두세요.*
**요구 지식**

* C언어를 이해할 수 있는 능력
* 어셈블리어(AT&T 문법)를 이해할 수 있는 능력

어쨌든, 이런 도구를 처음으로 배우기 시작했으면, 그런 분들을 위해 앞으로의 게시물들에서 제가 여러 파트를 설명할 수 있도록 노력할 것입니다.
자, 이제 간단한 소개가 끝났습니다. 이제 리눅스 커널과 로우레벨에 뛰어들 수 있겠군요.

제가 리눅스 커널 버전 3.18 때부터 이 게시물을 작성하기 시작했는데, 그 이후로 많은 부분들이 바뀌었습니다. 변경 사항이 생기면 그에 따라 게시물들을 업데이트하겠습니다.

마법의 컴퓨터 전원 버튼을 누르면 무슨 일이 벌어지나요?
--------------------------------------------------------------------------------

비록 이 시리즈의 포스트들이 리눅스 커널에 대한 것일지라도, 우리는 바로 커널 코드부터 시작하지 않을 것입니다. 적어도 이 단락에서는요. 

노트북 또는 데스크톱 컴퓨터에 있는 마법의 전원 버튼을 누르면, 컴퓨터와 노트북은 바로 작동하기 시작합니다. 메인보드는 파워 서플라이에 신호를 보냅니다. 신호를 받고 나면, 파워 서플라이는 컴퓨터에 적절한 양의 전력을 제공하기 시작합니다. 메인보드가 [파워 양호 신호(Power good signal)] (https://en.wikipedia.org/wiki/Power_good_signal) 을 받고 나면, 메인보드는 CPU 시작을 시도합니다. CPU는 모든 레지스터에 남아있던 데이터를 초기화하고, 각각에 미리 정의된 값들을 설정합니다. 

80386 CPU 이상에서 컴퓨터 리셋 이후 CPU 레지스터에 들어갈 미리 정의된 값들은 다음을 따릅니다:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

프로세서는 [리얼 모드](https://ko.wikipedia.org/wiki/리얼_모드) 에서 작동을 시작합니다. 조금 뒤로 물러나서, 이 모드에서의 [메모리 세그먼트 방식](https://ko.wikipedia.org/wiki/메모리_세그먼트)을 조금만 이해해봅시다. 
리얼모드는 [8086](https://ko.wikipedia.org/wiki/인텔_8086)에서부터 최신의 인텔 64bit CPU까지, 모든 x86 호환 프로세서에서 지원됩니다. 
`8086` 프로세서는 20비트 주소 버스를 가지고 있어서, `0-0xFFFFF` 또는 `1 메가바이트` 주소 공간으로 동작할 수 있었습니다. 하지만 8086은 오직 `2^16 - 1`, 또는 `0xffff`(64 KB)의 최대 주소를 갖는 `16비트` 레지스터밖에 가지고 있지 않았습니다.

[메모리 세그먼트 방식](https://ko.wikipedia.org/wiki/메모리_세그먼트)은 사용 가능한 모든 주소 공간을 이용하는데 쓰였습니다. 모든 메모리는 `65535` 바이트(64 KB) 의 일정한 크기를 가진 작은 세그먼트로 나누어집니다. 16비트 레지스터로는 `64 KB` 이상의 주소를 만들 수 없기 때문에 이러한 대체 방법을 고안하게 되었습니다.

주소는 두 파트로 구성됩니다: 주소를 가지고 있는 세그먼트 셀렉터, 그리고 기준 주소로부터의 오프셋. 리얼모드에서 세그먼트 셀렉터의 관련하는 기준 주소는 `세그먼트 셀렉터 * 16` 입니다. 따라서 메모리의 물리 주소를 얻기 위해서는, 이것에 세그먼트 셀렉터에 16을 곱하고 오프셋을 더해주어야 합니다:

```
물리주소 = 세그먼트 셀렉터 * 16 + 오프셋
```

예를 들어, 만약 `CS:IP` 가 `0x2000:0x0010`이면, 해당하는 물리주소는 아래와 같습니다:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

하지만, 만약 우리가 `0xffff:0xffff`와 같이 가장 큰 세그먼트 셀렉터와 오프셋을 갖는다면, 결과는 이렇게 될 것 입니다:

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

`65520` 바이트가 첫 메가바이트를 넘어가 버립니다. 따라서 오직 1MB까지만 접근 가능한 리얼모드에서, [A20 라인](https://en.wikipedia.org/wiki/A20_line)이 비활성화 되어 있을때 `0x10ffef`는 `0x00ffef`가 되어버립니다.
 
좋습니다, 이제 우리는 리얼모드와 이 모드에서의 메모리 주소 지정에 대해서 조금 알게 되었습니다. 다시 리셋 이후의 레지스터 값에 대한 논의로 돌아갑시다.

`CS` 레지스터는 두 파트로 구성됩니다: 보이는 세그먼트 셀렉터(visible segment selector), 그리고 숨겨진 기준 주소(hidden base address). 기준 주소는 일반적으로 세그먼트 셀렉터 값에 16을 곱하여 형성되지만, 하드웨어가 리셋되는 동안에는 CS 레지스터의 세그먼트 셀렉터에 `0xf0000`가 로드되고, 기준 주소에는 `0xffff0000`이 로드됩니다; 프로세서는 `CS`의 값이 바뀌기 전까지는 이 특별한 기준 주소를 사용합니다.

시작 주소는 EIP 레지스터의 값에 기준 주소를 더함으로써 형성됩니다.


```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

우리는 4GB의 16바이트 아래인 `0xfffffff0`을 얻었습니다. 이 지점은 [리셋 벡터](https://en.wikipedia.org/wiki/Reset_vector)라고 불립니다. 이곳은 CPU가 리셋 후 실행할 첫 번째 명령어가 있을 것이라 예상하는 메모리 위치입니다. 이 명령은 보통 BIOS 진입 지점을 가리키고 있는 [점프](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) 명령을 포함하고 있습니다. 예로써 [coreboot](https://www.coreboot.org/)의 소스코드(`src/cpu/x86/16bit/reset16.inc`)를 보면, 우리는 다음과 같은 코드를 볼 수 있습니다.:

```assembly
    .section ".reset", "ax", %progbits
    .code16
.globl	_start
_start:
    .byte  0xe9
    .int   _start16bit - ( . + 2 )
    ...
```

여기서 우리는 `jmp` 명령어의 [기계어 코드](http://ref.x86asm.net/coder32.html#xE9)인 `0xe9`를 볼 수 있습니다, 그리고 목적지 주소는 `_start16bit - ( . + 2)` 입니다.

우리는 또한 `rest` 섹션의 `16` 바이트를 볼 수 있습니다. 그리고 그것은 `0xfffffff0` 주소에서 시작하기 위해 컴파일 됩니다. (`src/cpu/x86/16bit/reset16.ld`):

```
SECTIONS {
    /* Trigger an error if I have an unuseable start address */
    _bogus = ASSERT(_start16bit >= 0xffff0000, "_start16bit too low. Please report.");
    _ROMTOP = 0xfffffff0;
    . = _ROMTOP;
    .reset . : {
        *(.reset);
        . = 15;
        BYTE(0x00);
    }
}
```

이제 바이오스가 시작되었습니다; 하드웨어 초기화와 검사가 끝난 후에, 바이오스는 부팅 가능한 디바이스를 찾아야 합니다. 부팅 순서는 BIOS 설정에 저장되어 있으며, BIOS가 부팅을 시도하는 장치를 제어합니다. 하드 드라이브로부터 부팅을 시도할때, 바이오스는 부트 섹터를 찾으려 시도합니다. [MBR 파티션 레이아웃](https://ko.wikipedia.org/wiki/마스터_부트_레코드) 으로 파티션된 하드 드라이브에서 각 섹터가 `512` 바이트일때, 부트 섹터는 첫 섹터의 첫 `446` 바이트에 저장됩니다. 첫 섹터의 마지막 두 바이트는 `0x55`와 `0xaa` 입니다. 이는 BIOS에게 부팅 가능한 장치라는 것을 알려주기 위해 디자인 되었습니다.

예를 들어:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

이렇게 빌드하고 실행하면:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

이를 통해 QEMU에게 디스크 이미지로 방금 우리가 만든 부팅 바이너리를 사용하도록 지시할 수 있습니다. 어셈블리 코드에 의해 생성된 바이너리는 부트 섹터의 요구사항(위치 카운터가 `0x7c00` 설정 되었으며, 매직 시퀀스로 끝남)을 충족하기 때문에, QEMU는 이 바이너리를 디스크 이미지의 마스터 부트 레코드 (MBR)로 다룹니다.

당신은 아래와 같이 보게 될것입니다:
You will see:

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

이 예제에서, 우리는 코드가 `16-bit` 리얼 모드에서 실행되며, 메모리의 `0x7c00`에서 시작한다는 것을 알 수 있습니다. 시작하고 난 후, 코드는 단순히 `!`를 출력하는 [0x10](http://wwww.ctyme.com/intr/rb-0106.htm) 인터럽트를 호출합니다; 나머지 `510` 바이트를 0으로 채우고, 매직 바이트 `0xaa`와 `0x55`로 끝납니다.

당신은 `objdump` 유틸리티를 사용하여 바이너리 덤프를 볼 수 있습니다.:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

실제 부트 섹터는 많은 0과 감탄사 대신에 부팅 프로세스와 파티션 테이블을 계속하기 위한 코드를 가지고 있습니다. 이 시점부터 BIOS는 부트로더에게 제어를 넘겨줍니다.

**주**: 위에 설명되어 있듯이, CPU는 리얼모드에 있습니다; 리얼모드에서는, 물리주소를 계산하기 위해 아래의 공식을 따릅니다:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

위에 설명한 것과 똑같습니다. 우리는 16비트 범용 레지스터밖에 가지고 있지 않습니다.; 16비트 레지스터의 최대 값은 `0xffff`입니다. 따라서 우리가 가장 큰 값을 가진다면, 결과는 다음과 같이 될 것입니다:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

여기서 `0x10ffef`는 `1MB + 64KB - 16b` 와 같습니다. [8086]((https://ko.wikipedia.org/wiki/인텔_8086) 프로세서 (리얼 모드를 지원하는 첫 번째 프로세서)는 그에 반해서, 20비트 주소 라인을 가지고 있었습니다. 따라서 `2^20 = 104876` 은 1MB 이므로, 사실상 사용 가능한 메모리가 1MB 라는 뜻이 됩니다.

일반적으로, 리얼 모드의 메모리 맵은 다음과 같습니다:

```
0x00000000 - 0x000003FF - 리얼 모드 인터럽트 벡터 테이블
0x00000400 - 0x000004FF - BIOS 데이터 구역 
0x00000500 - 0x00007BFF - 사용되지 않음 (Unused)
0x00007C00 - 0x00007DFF - 우리의 부트로더 
0x00007E00 - 0x0009FFFF - 사용되지 않음 (Unused)
0x000A0000 - 0x000BFFFF - 비디오 램 메모리 (VRAM)
0x000B0000 - 0x000B7777 - 흑백 비디오 메모리 
0x000B8000 - 0x000BFFFF - 컬러 비디오 메모리
0x000C0000 - 0x000C7FFF - 비디오 롬 BIOS (Video ROM BIOS)
0x000C8000 - 0x000EFFFF - BIOS 그림자 구역 (BIOS Shadow Area)
0x000F0000 - 0x000FFFFF - 시스템 BIOS
```

이 글의 시작에서, 저는 CPU에서 실행되는 첫 번째 명령어가 `0xFFFFFFF0`에 위치한다고 했었습니다, 이는 `0xFFFFF` (1MB) 보다 더 큰 주소입니다. CPU는 리얼모드에서 어떻게 이 주소에 접근하는 것일까요? 정답은 [coreboot](https://wwww.coreboot.org/Developer_Manual/Memory_map) 문서에 있습니다.

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 킬로바이트 롬이 주소 공간에 맵핑됨 (128 kilobyte ROM mapped into address space)
```

실행이 시작되었을 때, BIOS는 RAM이 아닌 ROM에 위치합니다.


부트로더
--------------------------------------------------------------------------------
[GRUB 2](https://wwww.gnu.org/software/grub/)와 [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project)와 같이 리눅스를 부팅시킬 수 있는 많은 부트로더가 있습니다. 리눅스 커널은 리눅스 지원 시행을 위해 부트로더의 요구사항을 명시해놓은 [부트 프로토콜(Boot protocol)](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt)을 가지고 있습니다. 이 예제에서는 GRUB2를 설명 할 것입니다.

계속하기 전에, `BIOS`가 부팅 장치를 선택하고 부트 섹터 코드로 제어권을 넘겨준 지금, 실행은 [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD)에서 시작합니다. 이 코드는 공간의 제한으로 인해 매우 간단하며, GRUB2 코어 이미지의 위치로 점프하는데 사용하기 위한 포인터를 포함하고 있습니다. 코어 이미지는 [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD)로 시작하는데, 이 이미지는 보통 첫 번째 파티션 이전의 사용되지 않는 공간(unused space) 첫 번째 섹터 바로 뒤에 저장됩니다. 위의 코드는 GRUB2의 커널과 파일 시스템 처리를 위한 드라이버를 포함하고 있는 나머지 코드 이미지를 메모리에 로드합니다. 나머지 코드의 로딩 이후에는 [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) 함수를 실행시킵니다.

`grub_main` 함수는 콘솔을 초기화하고, 모듈을 위한 기준 주소를 얻고, 루트 디바이스 설정하며, grub 설정 파일 로드/분석하고, 모듈 로드 등을 합니다. 실행이 끝나면 `grub_main` 함수가 grub를 normal mode로 이동시킵니다. `grub_normal_execute` 함수 (`grub-core/normal/main.c`의 소스코드 파일에 위치)는 최종 준비를 완료하고 운영체제 선택을 위한 메뉴를 보여줍니다. 우리가 grub 메뉴 항목들 중 하나를 선택하면 `grub_menu_excecute_entry` 함수가 실행되어 grub `boot` 명령을 실행하고 선택한 운영체제를 부팅합니다.

우리가 커널 부트 프로토콜(kernel boot protocol)에서 읽을 수 있듯이, 부트로더는 반드시 커널 설정 코드(kernel setup code)의 `0x01f1` 오프셋에서 시작하는 커널 설정 헤더(kernel setup header)의 일부 필드를 읽고 채워야 합니다. 당신은 부트 [링커 스크립트](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) 에서 이 오프셋의 값을 확인할 수 있습니다. 커널 헤더[arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)는 다음과 같이 시작합니다.

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

부트로더는 반드시 이 것과 헤더의 나머지 부분(Linux 부트 프로토콜에서 오직 `wrtie` 타입으로 표시된 것만, [이 예시와 같이](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354)) 을 커맨드 라인으로부터 받았거나 부팅 중에 계산된 값으로 채워야 합니다. (지금은 커널 설정 헤더의 모든 필드에 대한 모든 설명을 하지는 않을 것입니다, 하지만 우리는 커널이 이것을 어떻게 사용하는지에 대해 논의할 때 할 것입니다; 모든 필드에 대한 설명은 [부트 프로토콜](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156)에서 찾을 수 있습니다.)


커널 부트 프로토콜에서 볼 수 있듯이, 커널이 로딩된 후 메모리는 이렇게 맵핑 될 것입니다.
-------------------------------------------------------------------------------------------------------------------
```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Leave as much as possible unused
         ~                        ~
         | Command line           | (Can also be below the X+10000 mark)
X+10000  +------------------------+
         | Stack/heap             | For use by the kernel real-mode code.
X+08000  +------------------------+
         | Kernel setup           | The kernel real-mode code.
         | Kernel boot sector     | The kernel legacy boot sector.
       X +------------------------+
         | Boot loader            | <- Boot sector entry point 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

So, when the bootloader transfers control to the kernel, it starts at:

```
X + sizeof(KernelBootSector) + 1
```

where `X` is the address of the kernel boot sector being loaded. In my case, `X` is `0x10000`, as we can see in a memory dump:

![kernel first address](http://oi57.tinypic.com/16bkco2.jpg)

The bootloader has now loaded the Linux kernel into memory, filled the header fields, and then jumped to the corresponding memory address. We can now move directly to the kernel setup code.

The Beginning of the Kernel Setup Stage
--------------------------------------------------------------------------------

Finally, we are in the kernel! Technically, the kernel hasn't run yet; first, the kernel setup part must configure stuff such as the decompressor and some memory management related things, to name a few. After all these things are done, the kernel setup part will decompress the actual kernel and jump to it. Execution of the setup part starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) at the [_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292) symbol.

It may looks a little bit strange at first sight, as there are several instructions before it. A long time ago, the Linux kernel used to have its own bootloader. Now, however, if you run, for example,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

then you will see:

![Try vmlinuz in qemu](http://oi60.tinypic.com/r02xkz.jpg)

Actually, the file `header.S` starts with the magic number [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (see image above), the error message that displays and, following that, the [PE](https://en.wikipedia.org/wiki/Portable_Executable) header:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

It needs this to load an operating system with [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) support. We won't be looking into its inner workings right now and will cover it in upcoming chapters.

The actual kernel setup entry point is:

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (grub2 and others) knows about this point (at an offset of `0x200` from `MZ`) and makes a jump directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

The kernel setup entry point is:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

Here we can see a `jmp` instruction opcode (`0xeb`) that jumps to the `start_of_setup-1f` point. In `Nf` notation, `2f`, for example, refers to the local label `2:`; in our case, it is the label `1` that is present right after the jump, and it contains the rest of the setup [header](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156). Right after the setup header, we see the `.entrytext` section, which starts at the `start_of_setup` label.

This is the first code that actually runs (aside from the previous jump instructions, of course). After the kernel setup part receives control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the kernel real mode, i.e., after the first 512 bytes. This can be seen in both the Linux kernel boot protocol and the grub2 source code:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

In my case, the kernel is loaded at `0x10000` address. This means that segment registers will have the following values after kernel setup starts:

```
gs = fs = es = ds = ss = 0x10000
cs = 0x10200
```

After the jump to `start_of_setup`, the kernel needs to do the following:

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)

Let's look at the implementation.

Aligning the Segment Registers 
--------------------------------------------------------------------------------

First of all, the kernel ensures that the `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, `grub2` loads kernel setup code at address `0x10000` by default and `cs` at `0x10200` because execution doesn't start from the start of file, but from the jump here:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

which is at a `512` byte offset from [4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46). We also need to align `cs` from `0x10200` to `0x10000`, as well as all other segment registers. After that, we set up the stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack, followed by the address of the [6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterward, `ds` and `cs` will have the same values.

Stack Setup
--------------------------------------------------------------------------------

Almost all of the setup code is in preparation for the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575) is checking the `ss` register value and making a correct stack if `ss` is wrong:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

* `ss` has a valid value `0x1000` (as do all the other segment registers beside `cs`)
* `ss` is invalid and the `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and the `CAN_USE_HEAP` flag is not set (see below)

Let's look at all three of these scenarios in turn:

* `ss` has a correct address (`0x1000`). In this case, we go to label [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we set the alignment of `dx` (which contains the value of `sp` as given by the bootloader) to `4` bytes and a check for whether or not it is zero. If it is zero, we put `0xfffc` (4 byte aligned address before the maximum segment size of 64 KB) in `dx`. If it is not zero, we continue to use the value of `sp` given by the bootloader (0xf7f4 in my case). After this, we put the value of `ax` into `ss`, which stores the correct segment address of `0x1000` and sets up a correct `sp`. We now have a correct stack:

![stack](http://oi58.tinypic.com/16iwcis.jpg)

* In the second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) is a bitmask header which is defined as:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and, as we can read in the boot protocol:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (minimum stack size, `1024` bytes) to it. After this, if `dx` is not carried (it will not be carried, `dx = _end + 1024`), jump to label `2` (as in the previous case) and make a correct stack.

![stack](http://oi62.tinypic.com/dr7b5w.jpg)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

![minimal stack](http://oi60.tinypic.com/28w051y.jpg)

BSS Setup
--------------------------------------------------------------------------------

The last two steps that need to happen before we can jump to the main C code are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking the "magic" signature. First, signature checking:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is reported.

If the magic number matches, knowing we have a set of correct segment registers and a stack, we only need to set up the BSS section before jumping into the C code.

The BSS section is used to store statically allocated, uninitialized data. Linux carefully ensures this area of memory is first zeroed using the following code:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First, the [__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) address is moved into `di`. Next, the `_end + 3` address (+3 - aligns to 4 bytes) is moved into `cx`. The `eax` register is cleared (using a `xor` instruction), and the bss section size (`cx`-`di`) is calculated and put into `cx`. Then, `cx` is divided by four (the size of a 'word'), and the `stosl` instruction is used repeatedly, storing the value of `eax` (zero) into the address pointed to by `di`, automatically increasing `di` by four, repeating until `cx` reaches zero). The net effect of this code is that zeros are written through all words in memory from `__bss_start` to `_end`:

![bss](http://oi59.tinypic.com/29m2eyr.jpg)

Jump to main
--------------------------------------------------------------------------------

That's all - we have the stack and BSS, so we can jump to the `main()` C function:

```assembly
    calll main
```

The `main()` function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). You can read about what this does in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about Linux kernel insides. If you have questions or suggestions, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com), or just create an [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part, we will see the first C code that executes in the Linux kernel setup, the implementation of memory routines such as `memset`, `memcpy`, `earlyprintk`, early console implementation and initialization, and much more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [8086](https://en.wikipedia.org/wiki/Intel_8086)
  * [80386](https://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [coreboot developer manual](https://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](https://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](https://en.wikipedia.org/wiki/Power_good_signal)
