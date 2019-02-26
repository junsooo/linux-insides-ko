리눅스 커널에서의 시스템 콜 Part 1.
================================================================================

인트로
--------------------------------------------------------------------------------

지금부터 [linux-insides](https://0xax.gitbooks.io/linux-insides/content/) 책에서의 새로운 챕터가 시작됩니다. 제목에서 알 수 있듯이 이 챕터에서는 리눅스 커널의 [시스템 콜](https://en.wikipedia.org/wiki/System_call) 개념에 대해 설명할 것입니다. 이 장의 주제 선택은 우발적인 것이 아닙니다. 이전 [챕터](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)에서는 인터럽트와 인터럽트 핸들링에 대해 배웠습니다. 시스템 콜의 개념은 앞의 인터럽트 개념과 매우 유사합니다. 이는 시스템 콜을 구현하는 가장 일반적인 방법이 소프트웨어 인터럽트를 이용하는 것이기 때문입니다. 이 챕터에서는 시스템 콜 개념과 관련된 여러 측면을 공부할 것입니다. 예를 들어, 유저 스페이스(사용자 공간userspace)에서 시스템 호출이 발생할 때 어떤 일이 일어나는지 배울겁니다. 추가로, 리눅스 커널에서의 시스템 콜 핸들러, 예로 [VDSO](https://en.wikipedia.org/wiki/VDSO) 및 [vsyscall](https://lwn.net/Articles/446528/) 개념과 그 밖의 여러 가지 핸들러의 구현을 볼 것입니다.

리눅스 시스템 콜 구현을 공부하기 전에 시스템 콜에 대한 몇 가지 이론을 알고있는 것이 좋습니다. 다음 단락에서 해보죠.

시스템 콜이 뭐예요?
--------------------------------------------------------------------------------

시스템 콜은 커널의 서비스를 제공받기 위한 유저 스페이스의 요청일 뿐입니다. 맞습니다. 운영 체제 커널은 많은 서비스를 제공합니다. 프로그램이 파일에 뭔가를 쓰거나 읽으려 할 때, [소켓](https://en.wikipedia.org/wiki/Network_socket) 연결을 위해 listen을 시작할 때, 디렉토리를 생성하거나 삭제할 때, 심지어는 해당 작업을 끝낼 때 프로그램이 시스템 호출을 사용합니다. 다시 말하면, 시스템 콜은 유저 스페이스의 프로그램이 일부 요청들을 처리하기 위해 호출하는 커널 스페이스에 작성되어 있는 [C언어](https://en.wikipedia.org/wiki/C_%28programming_language%29) 함수일 뿐입니다.

리눅스 커널은 이러한 함수들의 세트를 제공하며 각 아키텍처에서 자체적인 세트를 제공합니다. 예를 들어, [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처는 [322](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl) 개의 시스템 콜을 제공하고 [x86](https://en.wikipedia.org/wiki/X86) 아키텍처는 [358](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_32.tbl)개의 다른 시스템 콜을 제공합니다. 다시 말하지만, 시스템 호출은 단지 함수일 뿐입니다. 어셈블리 프로그래밍 언어로 작성된 간단한 Hello world 예제를 살펴보시죠.

```assembly
.data

msg:
    .ascii "Hello, world!\n"
    len = . - msg

.text
    .global _start

_start:
	movq  $1, %rax
    movq  $1, %rdi
    movq  $msg, %rsi
    movq  $len, %rdx
    syscall

    movq  $60, %rax
    xorq  %rdi, %rdi
    syscall
```

위 명령을 다음과 같이 컴파일 할 수 있습니다.

```
$ gcc -c test.S
$ ld -o test test.o
```

그리고 다음과 같이 실행할 수 있습니다.

```
./test
Hello, world!
```

좋습니다. 우리가 여기서 무엇을 알 수 있을까요? 이 간단한 코드는 리눅스 `x86_64` 아키텍처용 `Hello World` 어셈블리 프로그램에 해당합니다. 우리는 여기서 두 개의 섹션을 확인할 수 있습니다.

* `.data`
* `.text`

첫 번째 섹션 - `.data` 에는 프로그램의 초기 상태 데이터가 저장되어 있습니다.(`Hello world` 문자열과 그 길이) 두 번째 섹션 - `.text` 에는 프로그램의 실제 코드가 저장되어 있습니다. 프로그램의 코드를 두 파트로 나누어보죠: 첫 번째 파트는 처음 `syscall` 명령어의 앞 부분, 두 번째 파트는 첫 번째와 두 번째 `syscall` 명령어 사이로 나누어봅시다. 일단 첫번째로, `syscall` 인스트럭션은 일반적으로 코드에서 어떤 일을 할까요? 그건 [64-ia-32-architectures-software-developer-vol-2b-manual](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)에서 읽을 수 있습니다.

```
SYSCALL invokes an OS system-call handler at privilege level 0. It does so by
loading RIP from the IA32_LSTAR MSR (after saving the address of the instruction
following SYSCALL into RCX). (The WRMSR instruction ensures that the
IA32_LSTAR MSR always contain a canonical address.)
...
...
...
SYSCALL loads the CS and SS selectors with values derived from bits 47:32 of the
IA32_STAR MSR. However, the CS and SS descriptor caches are not loaded from the
descriptors (in GDT or LDT) referenced by those selectors.

Instead, the descriptor caches are loaded with fixed values. It is the respon-
sibility of OS software to ensure that the descriptors (in GDT or LDT) referenced
by those selector values correspond to the fixed values loaded into the descriptor
caches; the SYSCALL instruction does not ensure this correspondence.
```

요약하면, `syscall` 인스트럭션은 `MSR_LSTAR` [Model specific register](https://en.wikipedia.org/wiki/Model-specific_register)(Long system target address register)에 저장된 주소로 점프하는 역할을 합니다. 커널은 시스템이 시작할 때 `MSR_LSTAR` 레지스터에 시스템 콜 핸들러 함수의 주소를 쓰는 것 뿐만 아니라, 시스템 콜 핸들러 내부에서 실제로 각 시스템 콜을 핸들링 하기 위한 커스텀 함수를 제공해야 합니다.
여기서 커스텀 함수란 [arch/x86/entry/entry_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/entry_64.S#L98)에 정의된 `entry_SYSCALL_64`를 뜻합니다. 이 시스템 핸들링 함수의 주소는 커널이 시작하는 도중 [arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c#L1335)에서 `MSR_LSTAR` 레지스터에 쓰여집니다.

```C
wrmsrl(MSR_LSTAR, entry_SYSCALL_64);
```

좋아요. `syscall` 인스트럭션이 주어진 시스템 콜의 핸들러를 호출해요. 그런데 실제로 어떤 핸들러를 호출해야 하는지 어떻게 알 수 있을까요? 사실 해당 정보는 범용 [레지스터](https://en.wikipedia.org/wiki/Processor_register)에서 가져옵니다. [시스템 콜 테이블](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl)에서 볼 수 있듯이, 각 시스템 콜은 고유한 번호를 가지고 있습니다. 우리 예시에서 첫번째 시스템 콜은 `write` 시스템 콜이고, 주어진 파일에 데이터를 쓰는 역할을 합니다. 한 번 시스템 콜 테이블에서 `write` 시스템 콜의 위치를 찾아보세요. [write](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl#L10) 시스템 콜이 숫자 `1`에 해당되는 것을 확인하셨을 겁니다. 우리는 이 숫자를 `rax` 레지스터를 통해 넘길 것입니다. 그 다음으로 `write` 시스템 콜을 수행하기 위해 필요한 세 개의 인자(parameter)를 넘기기 위해서는 각각 `%rdi`, `%rsi`, `%rdx` 범용 레지스터를 이용할겁니다. 다음을 확인해보세요.

* `%rdi` : [파일 디스크립터](https://en.wikipedia.org/wiki/File_descriptor) (`1` 은 [stdout](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29) 을 의미함)
* `%rsi` : string에 대한 포인터
* `%rdx` : 데이터의 크기

흠.. 당신이 제대로 본 게 맞아요. 위 내용은 시스템 콜에 필요한 인자에 대해 설명하고 있어요. 제가 위에서 말했던 것처럼, 시스템 콜은 단지 커널 스페이스의 `C` 함수일 뿐이예요. 위의 예제에서는 첫번째 시스템 콜이 `write` 인 것입니다. 이 시스템 콜은 [fs/read_write.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/read_write.c)소스 코드에 정의가 되어있어요. 그리고 이렇게 생겼죠.

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	...
	...
	...
}
```

다시 말하면 이렇게 생겼죠.

```C
ssize_t write(int fd, const void *buf, size_t nbytes);
```

지금은 `SYSCALL_DEFINE3` 매크로를 보고 너무 걱정하지 마세요. 나중에 다시 돌아올 겁니다.

위 예시의 두번째 파트는 거의 똑같지만, 사실 다른 시스템 콜을 부릅니다. 이 경우에는 [exit](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl#L69) 시스템 콜을 부릅니다. 이 시스템 콜은 딱 하나의 인자만 받습니다.

* `%rdi`: 리턴값(Return value)

그리고 프로그램이 종료되는 방식을 처리하죠. 한 번 위의 예시 프로그램을 [strace](https://en.wikipedia.org/wiki/Strace) 유틸리티로 실행해보면 실제로 시스템 콜이 동작하는 것을 확인할 수 있습니다.

```
$ strace test
execve("./test", ["./test"], [/* 62 vars */]) = 0
write(1, "Hello, world!\n", 14Hello, world!
)         = 14
_exit(0)                                = ?

+++ exited with 0 +++
```

`strace` output의 첫번째 줄에서, [execve](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl#L68) 시스템 콜이 우리의 프로그램을 실행하는 것을 확인할 수 있습니다. 두 번째와 세 번째는 우리가 프로그램에서 사용했던 `write` 와 `exit` 시스템 콜입니다. 위 예시에서 범용 레지스터를 통해 매개 변수를 전달한다는 것을 알아두세요. 레지스터 순서는 우연적인 것이 아니라 [x86-64 콜링 컨벤션](https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions)에 의해 정의됩니다. 이 컨벤션과 `x86_64` 아키텍쳐의 다른 컨벤션에 관련해서는 바로 뒤의 특별한 문서에서 다루고 있습니다. - [System V Application Binary Interface. PDF](https://github.com/hjl-tools/x86-psABI/wiki/x86-64-psABI-r252.pdf). 일반적으로, 함수의 인자는 레지스터에 저장이 되거나 스택에 push 됩니다. 함수의 처음 6개의 매개 변수에 대해 저장되는 순서는 다음과 같습니다.

* `rdi`
* `rsi`
* `rdx`
* `rcx`
* `r8`
* `r9`

만약 함수가 6개 이상의 인자를 가진다면 남은 인자들은 스택에 저장이 됩니다.

우리는 코드에서 직접 시스템 콜을 사용하지 않습니다. 프로그램이 뭔가를 출력하거나 파일에 접근해 읽기, 쓰기를 원할 때 내부적으로 시스템 콜을 사용합니다.

예를 들어보죠.

```C
#include <stdio.h>

int main(int argc, char **argv)
{
   FILE *fp;
   char buff[255];

   fp = fopen("test.txt", "r");
   fgets(buff, 255, fp);
   printf("%s\n", buff);
   fclose(fp);

   return 0;
}
```

리눅스 커널에는 `fopen`, `fgets`, `printf`, `fclose` 시스템 콜은 없지만, `open`, `read`, `write`, `close` 시스템 콜은 있습니다. 저는 아마 여러분이 `fopen`, `fgets`, `printf`, `fclose` 함수가 `C` [표준 라이브러리](https://en.wikipedia.org/wiki/GNU_C_Library)에 정의되어 있다는 것을 알 것이라고 생각합니다. 사실 이 함수들은 시스템 콜을 위한 wrapper일 뿐입니다. 다시 말하지만, 우리는 코드에서 시스템 콜을 직접 호출하지 않고 표준 라이브러리에 정의된 [wrapper](https://en.wikipedia.org/wiki/Wrapper_function) 함수를 사용합니다. 사실 그 주된 이유는 간단합니다. 시스템 콜이 아주 신속하게 실행되어야 하기 때문입니다. 빨리 실행되어야 하기 때문에 그 크기는 작아야 합니다. 표준 라이브러리는 시스템 콜을 수행할 때 알맞은 인자를 넣어서 수행하고, 또 시스템 콜을 실제로 수행하기 전에 관련된 여러 검사를 진행합니다. 위의 예시 프로그램을 다음과 같이 컴파일 해봅시다. 

```
$ gcc test.c -o test
```

그리고 [ltrace](https://en.wikipedia.org/wiki/Ltrace) 유틸리티를 이용해 분석해봅시다.

```
$ ltrace ./test
__libc_start_main([ "./test" ] <unfinished ...>
fopen("test.txt", "r")                                             = 0x602010
fgets("Hello World!\n", 255, 0x602010)                             = 0x7ffd2745e700
puts("Hello World!\n"Hello World!

)                                                                  = 14
fclose(0x602010)                                                   = 0
+++ exited (status 0) +++
```

`ltrace` 유틸리티는 프로그램의 유저 스페이스에서의 함수 call 내용들을 알려줍니다. `fopen` 함수는 주어진 텍스트 파일을 열고, `fgets` 함수는 파일의 내용을 읽어서 `buf` 버퍼로 읽어들입니다. 또 `puts` 함수는 버퍼의 내용을 `stdout`에 출력하고, `fclose` 함수는 파일 디스크립터를 통해 주어진 파일을 닫습니다. 그리고 이미 말했던 것처럼, 이 함수들은 모두 적절한 시스템 콜을 부릅니다. 예를 들어, `puts` 함수는 `write` 시스템 콜을 내부에서 부릅니다. `ltrace` 유틸리티에 `-S` 옵션을 추가하면 해당 내용을 볼 수 있습니다.

```
write@SYS(1, "Hello World!\n\n", 14) = 14
```

맞아요. 시스템콜은 어디에나 있습니다. 각 프로그램은 파일을 열고 읽고 쓰거나 네트워크 연결을 하거나, 메모리를 할당받거나 수많은 다른 일들을 할 수 있어야 합니다. 그리고 이런 것들은 오직 커널만이 제공해줄 수 있습니다. [Proc](https://en.wikipedia.org/wiki/Procfs) 파일 시스템은 `/proc/${pid}/syscall` 형식의 특별한 파일들을 포함하고 있습니다. 이 파일들은 시스템 콜 넘버와, 프로세스에서 현재까지 실행된 시스템 콜의 인자 레지스터 값들을 포함합니다. 예를 들어 pid 1을 가지는 [systemd](https://en.wikipedia.org/wiki/Systemd) 프로세스를 확인해보시죠.

```
$ sudo cat /proc/1/comm
systemd

$ sudo cat /proc/1/syscall
232 0x4 0x7ffdf82e11b0 0x1f 0xffffffff 0x100 0x7ffdf82e11bf 0x7ffdf82e11a0 0x7f9114681193
```

넘버 `232`를 가지는 시스템 콜 [epoll_wait](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl#L241)는 [epoll](https://en.wikipedia.org/wiki/Epoll) 파일 디스크립터에서 I/O 이벤트를 기다리는 역할을 합니다. 추가로 이 부분을 작성하는데 사용하고 있는 `emacs` 에디터를 예시로 들어보죠.

```
$ ps ax | grep emacs
2093 ?        Sl     2:40 emacs

$ sudo cat /proc/2093/comm
emacs

$ sudo cat /proc/2093/syscall
270 0xf 0x7fff068a5a90 0x7fff068a5b10 0x0 0x7fff068a59c0 0x7fff068a59d0 0x7fff068a59b0 0x7f777dd8813c
```

넘버 `270`를 가지는 시스템 콜 [sys_pselect6](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl#L279)은 `emacs`가 여러 파일 디스크립터들을 모니터링 할 수 있도록 허가해줍니다. 

이제 우리는 시스템 콜이 무엇이고, 왜 우리가 필요로 하는지에 대해 조금 알게 되었습니다. 이제 프로그램에서 사용된 `write` 시스템 콜을 좀 더 자세히 살펴보죠.

Write 시스템 콜 구현하기
--------------------------------------------------------------------------------

리눅스 커널의 소스 코드에서 write 시스템 콜의 구현을 직접 살펴봅시다. 우리가 이미 알고 있듯이 `write` 시스템 콜은 [fs/read_write.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/read_write.c) 소스 파일에 정의되어 있고 다음과 같이 작성되어 있습니다.

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_write(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}

	return ret;
}
```

우선, `SYSCALL_DEFINE3` 매크로는 [include/linux/syscalls.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/syscalls.h) 헤더 파일에 정의되어 있고,  `sys_name(...)` 함수의 정의를 확장합니다. 아래의 매크로를 한 번 보시죠.

```C
#define SYSCALL_DEFINE3(name, ...) SYSCALL_DEFINEx(3, _##name, __VA_ARGS__)

#define SYSCALL_DEFINEx(x, sname, ...)                \
        SYSCALL_METADATA(sname, x, __VA_ARGS__)       \
        __SYSCALL_DEFINEx(x, sname, __VA_ARGS__)
```

여기서 볼 수 있듯이, `SYSCALL_DEFINE3` 매크로는 `name` 인자를 제공 받습니다. 이 인자는 시스템 콜의 이름과 인자가 변할 수 있는 갯수를 드러냅니다. 이 매크로는 `SYSCALL_DEFINEx` 매크로를 확장한 것에 불과합니다. 해당 매크로는 주어진 시스템 콜의 인자 갯수와, 미래에 정해질 시스템 콜의 이름을 `_##name` stub을 통해 제공 받습니다. (`##`과 뒤의 토큰을 잇는 과정에 대해서는 [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)의 해당 [문서](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)에서 더 찾아볼 수 있습니다.) 그 다음으로 `SYSCALL_DEFINEx` 매크로 자체에 대해 알아보죠. 이 매크로는 아래의 두 매크로를 확장시킨 것입니다.

* `SYSCALL_METADATA`;
* `__SYSCALL_DEFINEx`.

첫 번째 매크로 `SYSCALL_METADATA`의 구현은 `CONFIG_FTRACE_SYSCALLS` 커널 설정 옵션에 의존합니다. 이 옵션의 이름에서 알 수 있듯이, 이 옵션은 tracer가 시스템 콜 엔트리와 종료 이벤트를 따라다닐 수 있도록 설정합니다. 만약 이 커널 설정 옵션이 켜져있으면 `SYSCALL_METADATA` 매크로는 [include/trace/syscall.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/trace/syscall.h) 헤더 파일에 정의된 `syscall_metadata` 구조를 초기화합니다. `syscall_metadata` 구조는 시스템 콜의 이름, 시스템 콜 [테이블](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl)에서의 넘버, 시스템 콜의 인자 수, 인자의 타입과 다양한 정보를 포함합니다.

```C
#define SYSCALL_METADATA(sname, nb, ...)                             \
	...                                                              \
	...                                                              \
	...                                                              \
    struct syscall_metadata __used                                   \
              __syscall_meta_##sname = {                             \
                    .name           = "sys"#sname,                   \
                    .syscall_nr     = -1,                            \
                    .nb_args        = nb,                            \
                    .types          = nb ? types_##sname : NULL,     \
                    .args           = nb ? args_##sname : NULL,      \
                    .enter_event    = &event_enter_##sname,          \
                    .exit_event     = &event_exit_##sname,           \
                    .enter_fields   = LIST_HEAD_INIT(__syscall_meta_##sname.enter_fields), \
             };                                                                            \

    static struct syscall_metadata __used                           \
              __attribute__((section("__syscalls_metadata")))       \
             *__p_syscall_meta_##sname = &__syscall_meta_##sname;
```

`CONFIG_FTRACE_SYSCALLS` 커널 옵션을 커널을 설정할 때 켜지 않으면 `SYSCALL_METADATA` 매크로는 빈 스트링을 확장합니다.

```C
#define SYSCALL_METADATA(sname, nb, ...)
```

두 번째 매크로 `__SYSCALL_DEFINEx`는 아래 5개의 함수들의 정의를 확장합니다.

```C
#define __SYSCALL_DEFINEx(x, name, ...)                                 \
        asmlinkage long sys##name(__MAP(x,__SC_DECL,__VA_ARGS__))       \
                __attribute__((alias(__stringify(SyS##name))));         \
                                                                        \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__));  \
                                                                        \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__));      \
                                                                        \
        asmlinkage long SyS##name(__MAP(x,__SC_LONG,__VA_ARGS__))       \
        {                                                               \
                long ret = SYSC##name(__MAP(x,__SC_CAST,__VA_ARGS__));  \
                __MAP(x,__SC_TEST,__VA_ARGS__);                         \
                __PROTECT(x, ret,__MAP(x,__SC_ARGS,__VA_ARGS__));       \
                return ret;                                             \
        }                                                               \
                                                                        \
        static inline long SYSC##name(__MAP(x,__SC_DECL,__VA_ARGS__))
```

첫번째로 `sys##name`은 `sys_system_call_name`의 이름을 가지는 시스템 콜 핸들러 함수의 정의입니다. `__SC_DECL` 매크로는 `__VA_ARGS__` 인자들을 제공받습니다. 해당 매크로를 통해 인자의 타입을 결정할 수 없기 때문에, 입력받는 인자의 타입과 이름들을 합쳐버립니다. 그리고 `__MAP` 매크로는 `__SC_DECL` 매크로를 `__VA_ARGS__` 인자들에 적용시킵니다. `__SYSCALL_DEFINEx` 매크로에 의해 생성된 다른 함수들은 보안 취약점 [CVE-2009-0029](https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2009-0029)에 취약하지 않도록 패치를 진행해야 합니다. 일단 여기서는 그 자세한 내용에 대해 다루지는 않겠습니다. 어쨌든 좋아요.`SYSCALL_DEFINE3` 매크로의 결과로, 우리는 다음과 같은 함수를 얻게 되었습니다.

```C
asmlinkage long sys_write(unsigned int fd, const char __user * buf, size_t count);
```

이제 시스템 콜의 정의에 대해 조금 알게 되었으니, `write` 시스템 콜의 구현에 대해 다시 알아봅시다. 이 시스템 콜의 구현을 다시 살펴보아요.

```C
SYSCALL_DEFINE3(write, unsigned int, fd, const char __user *, buf,
		size_t, count)
{
	struct fd f = fdget_pos(fd);
	ssize_t ret = -EBADF;

	if (f.file) {
		loff_t pos = file_pos_read(f.file);
		ret = vfs_write(f.file, buf, count, &pos);
		if (ret >= 0)
			file_pos_write(f.file, pos);
		fdput_pos(f);
	}

	return ret;
}
```

코드에서 볼 수 있듯이, 이 시스템 콜은 3개의 인자를 제공 받습니다.

* `fd`    - 파일 디스크립터;
* `buf`   - 데이터를 작성할 버퍼;
* `count` - 버퍼에 작성할 데이터의 길이;

그리고 주어진 장치나 파일으로부터 데이터를 제공받아 유저가 선언한 버퍼에 데이터를 작성합니다. 두번째 인자 `buf`는 `__user` 속성으로 정의됩니다. 해당 속성을 이용하는 주된 이유는 [sparse](https://en.wikipedia.org/wiki/Sparse) 유틸리티를 이용해 리눅스 커널 코드를 체크하기 위함입니다. 이것은 [include/linux/compiler.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/compiler.h) 헤더 파일에 정의되어 있으며 리눅스 커널의 `__CHECKER__` 정의에 따라 달라집니다. 해당 정의는 `sys_write` 시스템 콜의 유용한 메타데이터에 관한 모든 정보를 포함하고 있습니다. 
이 시스템 콜이 어떻게 구현되어 있는지 이해해봅시다. 먼저 함수의 첫번째 줄에서는, `fd` 구조체 타입을 가지는 변수 `f`에 `fdget_pos` 함수의 인자로 unsigned int 타입을 가지는 `fd`를 넣고 계산한 함수의 리턴 값을 삽입합니다. `fdget_pos` 함수는 같은 [소스](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/read_write.c) 파일에 정의되어 있고 `__to_fd` 함수의 실행 결과를 확장합니다.

```C
static inline struct fd fdget_pos(int fd)
{
        return __to_fd(__fdget_pos(fd));
}
```

`fdget_pos` 함수는 숫자로 주어진 파일 디스크립터를 `fd` 구조체로 바꾸는 역할을 합니다. `fdget_pos` 함수는 여러번의 긴 함수 호출을 통해 현재 프로세스의 파일 디스크립터 테이블 `current->files`를 얻습니다. 그리고 해당 테이블에서 fd 넘버에 일치하는 파일 디스크립터를 찾습니다. 해당 파일 디스크립터 넘버에 대한 `fd` 구조체를 찾으면 변수 f에 해당 구조체를 대입합니다. 만약 찾지 못하면 `-EBADF` 값을 함수에서 반환하고 함수를 종료합니다. 그 다음으로 `file_pos_read` 함수를 호출해서, 파일의 `f_pos` 필드를 반환하여 파일의 현재 위치를 획득합니다.(역주 : 파일의 현재 위치란 파일을 write 하기 시작하는 위치를 말함)

```C
static inline loff_t file_pos_read(struct file *file)
{
        return file->f_pos;
}
```

그 다음으로 `vfs_write` 함수를 호출합니다. `vfs_write` 함수는 [fs/read_write.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/fs/read_write.c) 소스 파일에 정의되어 있고, 인자로 주어진 버퍼를 주어진 파일의 위치에서부터 파일에 쓰는 역할을 합니다. 여기서는 `vfs_write` 함수의 자세한 내용에 대해 다루지 않을 것입니다. 이 함수는 `system call` 개념보다는 [가상 파일 시스템](https://en.wikipedia.org/wiki/Virtual_file_system) 개념과 깊은 관련이 있고, 그에 따라 다른 챕터에서 이에 대해 배울 것입니다. `vfs_write` 함수를 끝마치면 이 함수의 결과를 확인하고 만약 성공적으로 함수가 종료했으면 `file_pos_write` 함수를 통해 파일의 현재 위치를 실제로 `write` 한만큼 변경합니다.

```C
if (ret >= 0)
	file_pos_write(f.file, pos);
```

즉, `f_pos` 을 주어진 파일과 `pos` 변수에 따라 업데이트합니다.

```C
static inline void file_pos_write(struct file *file, loff_t pos)
{
        file->f_pos = pos;
}
```

`write` 시스템 콜 핸들러의 마지막 부분에서 다음 함수를 호출하는 것을 확인할 수 있습니다.

```C
fdput_pos(f);
```

이 함수는, 여러 쓰레드에서 파일 디스크립터를 공유할 때 파일의 위치 값을 동시에 작성하지 못하도록 하는 `f_pos_lock` 뮤텍스를 해제하는 역할을 합니다.

좋아요. 이게 끝입니다.

우리는 리눅스 커널에서 제공하는 `write` 시스템 콜의 부분적인 구현을 살펴보았습니다. 물론 저는 `write` 시스템 콜 내부 구현의 일정 부분에 대한 내용은 생략했습니다. 왜냐하면 위에서 서술한 것처럼, 이 챕터에서는 시스템 콜에 관련된 내용에만 집중하고 [가상 파일 시스템](https://en.wikipedia.org/wiki/Virtual_file_system)과 같은 내용은 공부하지 않을 것이기 때문입니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널의 시스템 콜 개념을 다루는 첫번째 파트를 마칩니다. 지금까지 시스템 콜의 이론적인 부분에 대해 살펴보았고, 다음 파트에서는 시스템 콜에 연관된 리눅스 커널 코드를 보면서 이 주제에 대해 좀 더 깊이 파고들겁니다. 

(원작자) 질문이나 제안할 사항이 있으면 [이슈](https://github.com/0xAX/linux-insides/issues/new)를 남기거나, 트위터 [0xAX](https://twitter.com/0xAX)를 핑하거나, [이메일](anotherworldofworld@gmail.com)을 남기세요.

**(원작자) 영어가 제 모국어가 아니어서 정말 죄송하게 생각합니다. 내용에서 이상한 부분을 찾으면 [linux-insides](https://github.com/0xAX/linux-insides) 레포지토리에 풀 리퀘스트를 올려주세요.**

**(번역자) 번역 중 오역이나 제안할 사항이 있으면 언제든 [linux-insides-ko](https://github.com/0xAX/linux-insides) 레포지토리에 풀 리퀘스트나 이슈를 남겨주세요.**

링크
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [vdso](https://en.wikipedia.org/wiki/VDSO)
* [vsyscall](https://lwn.net/Articles/446528/)
* [general purpose registers](https://en.wikipedia.org/wiki/Processor_register)
* [socket](https://en.wikipedia.org/wiki/Network_socket)
* [C programming language](https://en.wikipedia.org/wiki/C_%28programming_language%29)
* [x86](https://en.wikipedia.org/wiki/X86)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [x86-64 calling conventions](https://en.wikipedia.org/wiki/X86_calling_conventions#x86-64_calling_conventions)
* [System V Application Binary Interface. PDF](http://www.x86-64.org/documentation/abi.pdf)
* [GCC](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
* [Intel manual. PDF](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [system call table](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/entry/syscalls/syscall_64.tbl)
* [GCC macro documentation](https://gcc.gnu.org/onlinedocs/cpp/Concatenation.html)
* [file descriptor](https://en.wikipedia.org/wiki/File_descriptor)
* [stdout](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_.28stdout.29)
* [strace](https://en.wikipedia.org/wiki/Strace)
* [standard library](https://en.wikipedia.org/wiki/GNU_C_Library)
* [wrapper functions](https://en.wikipedia.org/wiki/Wrapper_function)
* [ltrace](https://en.wikipedia.org/wiki/Ltrace)
* [sparse](https://en.wikipedia.org/wiki/Sparse)
* [proc file system](https://en.wikipedia.org/wiki/Procfs)
* [Virtual file system](https://en.wikipedia.org/wiki/Virtual_file_system)
* [systemd](https://en.wikipedia.org/wiki/Systemd)
* [epoll](https://en.wikipedia.org/wiki/Epoll)
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/Interrupts/index.html)
