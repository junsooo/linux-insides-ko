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
컴퓨터나 노트북에 있는 마법의 전원 버튼을 누르면, 컴퓨터와 노트북은 작동하기 시작합니다. 그 후에는 메인보드가 파워 서플라이에 신호를 보냅니다. 신호를 받고 나면, 파워 서플라이는 컴퓨터에 적절한 양의 전력을 제공하기 시작합니다. 메인보드가 Power good signal을 받고나면, 메인보드는 CPU 시작을 시도합니다. CPU는 모든 레지스터에 남아있던 데이터를 초기화하고, 각각에 미리 정의된 값들을 설정합니다. 

80386 CPU 이상에서 컴퓨터 리셋 이후, CPU 레지스터에 설정될 미리 정의된 값 들은 다음을 따릅니다:

```
IP          0xfff0
CS selector 0xf000
CS base     0xffff0000
```

프로세서는 리얼 모드에서 작동을 시작합니다. 조금만 뒤로 돌아가서 이 모드에서의 세그먼테이션에 대해 이해해봅시다. 리얼모드는 모든 x86 호환 프로세서에서 지원합니다, 이는 8086에서부터 인텔 64bit CPU가 나오기 까지의 모든 CPU가 해당됩니다. `8086` 프로세서는 20비트 주소 버스를 가지고 있었고, 이는 곧 `0-0xFFFFF` 또는 `1 메가바이트` 주소 공간으로 동작한다는 뜻입니다. 그러나 8086은 오직 `2^16 - 1` 또는 `0xffff`를 최대 주소로 갖는 `16비트` 레지스터밖에 가지고 있지 않았습니다.

메모리 세그먼테이션은 사용 가능한 모든 주소 공간을 이용하는데 쓰입니다. 모든 메모리는 `65535` 바이트 (64 KB)의 일정한 크기를 가진 세그먼트로 작게 나누어집니다. 16비트 레지스터로 `64 KB` 이상의 주소는 만들 수 없기 때문에 이러한 대체 방법을 고안하게 되었습니다.

주소는 두 파트로 구성됩니다: 기준 주소를 가지고 있는 세그먼트 셀렉터, 그리고 기준 주소로부터의 오프셋. 리얼모드에서는, 세그먼트 셀렉터의 관련된 기준 주소는 `세그먼트 셀렉터 * 16` 입니다. 따라서 메모리의 물리 주소를 얻기 위해서는, 세그먼트 셀렉터에 16을 곱하고 오프셋을 더해주어야 합니다:

```
물리주소 = 세그먼트 셀렉터 * 16 + 오프셋
```

예를 들어, 만약 `CS:IP` 가 `0x2000:0x0010`이면, 해당하는 물리주소는 아래와 같습니다:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

하지만, 만약 우리가 `0xffff:0xffff`와 같이 제일 큰 세그먼트 셀렉터와 오프셋을 갖는다면, 결과는 아래와 같을 것 입니다:
But, if we take the largest segment selector and offset, `0xffff:0xffff`, then the resulting address will be:

```python
>>> hex((0xffff << 4) + 0xffff)
'0x10ffef'
```

`65520`바이트는 1메가 바이트를 넘어갑니다. 따라서 1메가바이트만 접근 가능한 리얼모드 에서는, `0x10ffef`는 A20 라인이 비활성화 되어 있을 `0x00ffef` 가 되어버립니다.
which is `65520` bytes past the first megabyte. Since only one megabyte is accessible in real mode, `0x10ffef` becomes `0x00ffef` with the [A20 line](https://en.wikipedia.org/wiki/A20_line) disabled.
 
좋습니다, 이제 우리는 리얼모드와 이 모드에서의 메모리 주소 지정에 대해서 조금이나마 알게 되었습니다. 다시 리셋 이후의 레지스터 값에 대한 논의로 돌아갑시다.
Ok, now we know a little bit about real mode and memory addressing in this mode. Let's get back to discussing register values after reset.

`CS` 레지스터는 두 파트로 구성됩니다: 보이는 세그먼트 셀렉터, 그리고 숨겨진 기준 주소. 기준 주소는 일반적으로 세그먼트 셀렉터 값에 16을 곱하여 형성되지만, 하드웨어가 리셋되는 동안 CS 레지스터의 세그먼트 셀렉터에는 `0xf0000`가 로드되고, 기준 주소에는 `0xffff0000`이 로드됩니다; 프로세서는 `CS`의 값이 바뀌기 전까지는 이 특별한 기준 주소를 사용합니다.
The `CS` register consists of two parts: the visible segment selector, and the hidden base address. While the base address is normally formed by multiplying the segment selector value by 16, during a hardware reset the segment selector in the CS register is loaded with `0xf000` and the base address is loaded with `0xffff0000`; the processor uses this special base address until `CS` is changed.

시작 주소는 EIP 레지스터의 값에 기준 주소를 더하는 것으로 형성됩니다.
The starting address is formed by adding the base address to the value in the EIP register:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

We get `0xfffffff0`, which is 16 bytes below 4GB. This point is called the [reset vector](https://en.wikipedia.org/wiki/Reset_vector). This is the memory location at which the CPU expects to find the first instruction to execute after reset. It contains a [jump](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29) (`jmp`) instruction that usually points to the BIOS entry point. For example, if we look in the [coreboot](https://www.coreboot.org/) source code (`src/cpu/x86/16bit/reset16.inc`), we will see:
우리는 4GB 아래의 16바이트인 `0xfffffff0`을 얻었습니다. 이 지점은 [리셋 벡터]라고 불립니다. 이곳은 리셋 후 실행해야할 첫 번째 명령어가 있다고 CPU가 예상하는 곳입니다. 이 명령은 [점프] (`jmp`) 명령을 포함하고 있으며, 보통 바이오스 진입 지점으로 쓰입니다. 예를 들어, [coreboot]의 소스코드 (`src/cpu/x86/16bit/reset16.inc`) 를 보겠습니다.:

```assembly
    .section ".reset", "ax", %progbits
    .code16
.globl	_start
_start:
    .byte  0xe9
    .int   _start16bit - ( . + 2 )
    ...
```

Here we can see the `jmp` instruction [opcode](http://ref.x86asm.net/coder32.html#xE9), which is `0xe9`, and its destination address at `_start16bit - ( . + 2)`.
여기서 우리는 `0xe9`, 즉 `jmp` 명령을 찾을 수 있습니다, 그리고 이에 대한 목적지 주소는 `_start16bit - ( . + 2)` 입니다.

We can also see that the `reset` section is `16` bytes and that is compiled to start from `0xfffffff0` address (`src/cpu/x86/16bit/reset16.ld`):
우리는 또한 `rest` 섹션의 `16` 바이트를 볼 수 있습니다. 그리고 그것은 `0xfffffff0`에서 시작하기 위해 컴파일 됩니다. 

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

Now the BIOS starts; after initializing and checking the hardware, the BIOS needs to find a bootable device. A boot order is stored in the BIOS configuration, controlling which devices the BIOS attempts to boot from. When attempting to boot from a hard drive, the BIOS tries to find a boot sector. On hard drives partitioned with an [MBR partition layout](https://en.wikipedia.org/wiki/Master_boot_record), the boot sector is stored in the first `446` bytes of the first sector, where each sector is `512` bytes. The final two bytes of the first sector are `0x55` and `0xaa`, which designates to the BIOS that this device is bootable.
이제 바이오스가 시작되었습니다; 하드웨어 초기화와 검사가 끝난 후, 바이오스는 부팅 가능한 디바이스를 필요로 합니다. 부팅 순서는 BIOS 설정에 저장되어 있으며, BIOS가 부팅을 시도하는 장치를 제어합니다. 하드 드라이브로 부터 부트를 시도할때, 바이오스는 부트 섹터를 찾으려 시도합니다. MBR 형식으로 파티션된 하드 드라이브에서, 각 섹터가 `512` 바이트일때, 부트 섹터는 첫 섹터의 첫 `446` 바이트에 저장됩니다. 첫 섹터의 마지막 두 바이트들은 `0x55`와 `0xaa` 입니다. 이는 BIOS가 부팅 가능한 장치를 식별하기 위해서 디자인 되었습니다.

예를 들어:
For example:

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

빌드하고 실행하면:
Build and run this with:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

This will instruct [QEMU](http://qemu.org) to use the `boot` binary that we just built as a disk image. Since the binary generated by the assembly code above fulfills the requirements of the boot sector (the origin is set to `0x7c00` and we end with the magic sequence), QEMU will treat the binary as the master boot record (MBR) of a disk image.

You will see:

![Simple bootloader which prints only `!`](http://oi60.tinypic.com/2qbwup0.jpg)

In this example, we can see that the code will be executed in `16-bit` real mode and will start at `0x7c00` in memory. After starting, it calls the [0x10](http://www.ctyme.com/intr/rb-0106.htm) interrupt, which just prints the `!` symbol; it fills the remaining `510` bytes with zeros and finishes with the two magic bytes `0xaa` and `0x55`.

You can see a binary dump of this using the `objdump` utility:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

A real-world boot sector has code for continuing the boot process and a partition table instead of a bunch of 0's and an exclamation mark :) From this point onwards, the BIOS hands over control to the bootloader.

**NOTE**: As explained above, the CPU is in real mode; in real mode, calculating the physical address in memory is done as follows:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

just as explained above. We have only 16-bit general purpose registers; the maximum value of a 16-bit register is `0xffff`, so if we take the largest values, the result will be:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

where `0x10ffef` is equal to `1MB + 64KB - 16b`. An [8086](https://en.wikipedia.org/wiki/Intel_8086) processor (which was the first processor with real mode), in contrast, has a 20-bit address line. Since `2^20 = 1048576` is 1MB, this means that the actual available memory is 1MB.

In general, real mode's memory map is as follows:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table
0x00000400 - 0x000004FF - BIOS Data Area
0x00000500 - 0x00007BFF - Unused
0x00007C00 - 0x00007DFF - Our Bootloader
0x00007E00 - 0x0009FFFF - Unused
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory
0x000B0000 - 0x000B7777 - Monochrome Video Memory
0x000B8000 - 0x000BFFFF - Color Video Memory
0x000C0000 - 0x000C7FFF - Video ROM BIOS
0x000C8000 - 0x000EFFFF - BIOS Shadow Area
0x000F0000 - 0x000FFFFF - System BIOS
```

In the beginning of this post, I wrote that the first instruction executed by the CPU is located at address `0xFFFFFFF0`, which is much larger than `0xFFFFF` (1MB). How can the CPU access this address in real mode? The answer is in the [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map) documentation:

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

At the start of execution, the BIOS is not in RAM, but in ROM.

Bootloader
--------------------------------------------------------------------------------

There are a number of bootloaders that can boot Linux, such as [GRUB 2](https://www.gnu.org/software/grub/) and [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). The Linux kernel has a [Boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt) which specifies the requirements for a bootloader to implement Linux support. This example will describe GRUB 2.

Continuing from before, now that the `BIOS` has chosen a boot device and transferred control to the boot sector code, execution starts from [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). This code is very simple, due to the limited amount of space available, and contains a pointer which is used to jump to the location of GRUB 2's core image. The core image begins with [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), which is usually stored immediately after the first sector in the unused space before the first partition. The above code loads the rest of the core image, which contains GRUB 2's kernel and drivers for handling filesystems, into memory. After loading the rest of the core image, it executes the [grub_main](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/kern/main.c) function.

The `grub_main` function initializes the console, gets the base address for modules, sets the root device, loads/parses the grub configuration file, loads modules, etc. At the end of execution, the `grub_main` function moves grub to normal mode. The `grub_normal_execute` function (from the `grub-core/normal/main.c` source code file) completes the final preparations and shows a menu to select an operating system. When we select one of the grub menu entries, the `grub_menu_execute_entry` function runs, executing the grub `boot` command and booting the selected operating system.

As we can read in the kernel boot protocol, the bootloader must read and fill some fields of the kernel setup header, which starts at the `0x01f1` offset from the kernel setup code. You may look at the boot [linker script](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) to confirm the value of this offset. The kernel header [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) starts from:

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

The bootloader must fill this and the rest of the headers (which are only marked as being type `write` in the Linux boot protocol, such as in [this example](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354)) with values which it has either received from the command line or calculated during boot. (We will not go over full descriptions and explanations for all fields of the kernel setup header now, but we shall do so when we discuss how the kernel uses them; you can find a description of all fields in the [boot protocol](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156).)

As we can see in the kernel boot protocol, the memory will be mapped as follows after loading the kernel:

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
