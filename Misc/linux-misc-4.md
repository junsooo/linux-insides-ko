사용자 공간에서 프로그램 시작 프로세스
================================================================================

소개
--------------------------------------------------------------------------------

[리눅스 내부](https://www.gitbook.com/book/0xax/linux-insides/details)는 주로 리눅스 커널 관련 내용을 설명했지만, 나는 주로 사용자 공간과 관련된 이 파트를 작성하기로 결정했습니다.

이미 [시스템 콜](https://en.wikipedia.org/wiki/System_call)의 네 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-4.html)에서 프로그램을 시작할 때 리눅스 커널이 무엇을 하는지 설명했습니다. 이 파트에서는 우리가 사용자 공간 관점에서 리눅스 시스템에서 프로그램을 실행할 때 발생하는 것을 탐구하려 합니다.

저는 당신에 대해서 잘 모르지만, 제 대학에서 배운 `C`프로그램은 `main`이라 불리는 함수를 실행하며 시작했습니다. 그리고 그것은 부분적으로 사실입니다. 새 프로그램을 작성할 때마다, 다음 코드 라인에서 프로그램을 시작합니다:

```C
int main(int argc, char *argv[]) {
	// Entry point is here
}
```

그러나 저레벨 프로그래밍에 관심이 있다면, 당신은 이미 `main`함수가 프로그램의 실제 진입 점이 아니라는 것을 알고있을 것입니다. 디버거에서 이 간단한 프로그램을 살펴본 다음에는 이것이 사실이라고 생각할 것입니다:

```C
int main(int argc, char *argv[]) {
	return 0;
}
```

이것을 컴파일하고 [gdb](https://www.gnu.org/software/gdb/)에서 실행합시다:

```
$ gcc -ggdb program.c -o program
$ gdb ./program
The target architecture is assumed to be i386:x86-64:intel
Reading symbols from ./program...done.
```

`files`인자로 gdb `info`서브커맨드를 실행해봅시다. `info files`은 다른 섹션이 차지하는 디버깅 대상 및 메모리 공간에 대한 정보를 출력합니다.

```
(gdb) info files
Symbols from "/home/alex/program".
Local exec file:
	`/home/alex/program', file type elf64-x86-64.
	Entry point: 0x400430
	0x0000000000400238 - 0x0000000000400254 is .interp
	0x0000000000400254 - 0x0000000000400274 is .note.ABI-tag
	0x0000000000400274 - 0x0000000000400298 is .note.gnu.build-id
	0x0000000000400298 - 0x00000000004002b4 is .gnu.hash
	0x00000000004002b8 - 0x0000000000400318 is .dynsym
	0x0000000000400318 - 0x0000000000400357 is .dynstr
	0x0000000000400358 - 0x0000000000400360 is .gnu.version
	0x0000000000400360 - 0x0000000000400380 is .gnu.version_r
	0x0000000000400380 - 0x0000000000400398 is .rela.dyn
	0x0000000000400398 - 0x00000000004003c8 is .rela.plt
	0x00000000004003c8 - 0x00000000004003e2 is .init
	0x00000000004003f0 - 0x0000000000400420 is .plt
	0x0000000000400420 - 0x0000000000400428 is .plt.got
	0x0000000000400430 - 0x00000000004005e2 is .text
	0x00000000004005e4 - 0x00000000004005ed is .fini
	0x00000000004005f0 - 0x0000000000400610 is .rodata
	0x0000000000400610 - 0x0000000000400644 is .eh_frame_hdr
	0x0000000000400648 - 0x000000000040073c is .eh_frame
	0x0000000000600e10 - 0x0000000000600e18 is .init_array
	0x0000000000600e18 - 0x0000000000600e20 is .fini_array
	0x0000000000600e20 - 0x0000000000600e28 is .jcr
	0x0000000000600e28 - 0x0000000000600ff8 is .dynamic
	0x0000000000600ff8 - 0x0000000000601000 is .got
	0x0000000000601000 - 0x0000000000601028 is .got.plt
	0x0000000000601028 - 0x0000000000601034 is .data
	0x0000000000601034 - 0x0000000000601038 is .bss
```

`Entry point: 0x400430`라인을 참고하십시오. 이제 우리는 프로그램 진입 점의 실제 주소를 알고 있습니다. 이 주소로 중단점을 작성하고, 프로그램을 실행해 어떤 일이 일어나는지 봅시다:

```
(gdb) break *0x400430
Breakpoint 1 at 0x400430
(gdb) run
Starting program: /home/alex/program 

Breakpoint 1, 0x0000000000400430 in _start ()
```

흥미롭습니다. 여기서 `main`함수의 실행이 보이지 않지만 우리는 다른 함수가 호출되는 것을 봤습니다. 이 함수는 `_start`이고 우리의 디버거가 우리에게 보여주듯이 프로그램의 실제 진입점입니다. 이 함수는 어디에서 왔을까요? 누가 `main`을 호출하고 언제 호출될까요? 다음 포스트에서 이 모든 질문에 대답할 것입니다. 

커널이 어떻게 새 프로그램을 시작하는지
--------------------------------------------------------------------------------

먼저, 다음과 같은 간단한 `C`프로그램을 살펴보겠습니다:

```C
// program.c

#include <stdlib.h>
#include <stdio.h>

static int x = 1;

int y = 2;

int main(int argc, char *argv[]) {
	int z = 3;

	printf("x + y + z = %d\n", x + y + z);

	return EXIT_SUCCESS;
}
```

이 프로그램이 예상대로 작동하는지 확인할 수 있습니다. 그것을 컴파일합시다:

```
$ gcc -Wall program.c -o sum
```

그리고 실행합니다: 

```
$ ./sum
x + y + z = 6
```

지금까지 모든 것이 꽤 좋아 보입니다. 당신은 이미 특별한 함수 계열 [exec*](http://man7.org/linux/man-pages/man3/execl.3.html)가 있다는 것을 알고 있을 것입니다. man페이지에서 읽으면:

> exex() 함수 계열은 현재 프로세스 이미지를 새 프로세스 이미지로 바꿉니다.

모든 `exec*`함수는 [execve](http://man7.org/linux/man-pages/man2/execve.2.html)시스템 호출에 대한 간단한 프론트엔드입니다. [시스템 콜](https://en.wikipedia.org/wiki/System_call)로 설명한 챕터의 네 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-4.html)를 읽어보면, [execve](http://linux.die.net/man/2/execve)시스템 호출이 [files/exec.c](https://github.com/torvalds/linux/blob/08e4e0d0456d0ca8427b2d1ddffa30f1c3e774d7/fs/exec.c#L1888)소스 코드 파일에서 정의된 것을 알 수 있습니다. 다음을 보십시오:

```C
SYSCALL_DEFINE3(execve,
		const char __user *, filename,
		const char __user *const __user *, argv,
		const char __user *const __user *, envp)
{
	return do_execve(getname(filename), argv, envp);
}
```

실행 파일 이름, 명령 라인 인자 및 환경 변수 세트를 사용합니다. 추측할 수 있듯이, 모든 것은 `do_execve`함수에 의해 수행됩니다. [여기](https://0xax.gitbooks.io/linux-insides/content/SysCall/linux-syscall-4.html)에서 이것에 관해 읽을 수 있기 때문에 `do_execve`함수의 구현에 대해서는 자세히 설명하지 않겠습니다. 그러나 간단히 말해, `do_execve`함수는 `filename`과 같은 것이 유효한지 많은 검사를 수행하고, 시작된 프로세스의 한계는 시스템 등을 초과하지 않습니다. 이러한 검사가 모두 끝나면, 이 함수는 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)형식으로 표현되는 실행파일을 분석하고 새로 실행되는 실행파일을 위한 메모리 디스크립터를 만들고 그것을 스택, 힙 등의 영역과 같은 적절한 값으로 채웁니다. 새 이진 이미지 설정이 완료되면 `start_thread`함수는 하나의 새 프로세스를 설정합니다. 이 함수는 아키텍쳐에 따라 다르며 [x86_64](https://en.wikipedia.org/wiki/X86-64)아키텍처의 경우 해당 정의는 [arch/x86/kernel/process_64.c](https://github.com/torvalds/linux/blob/08e4e0d0456d0ca8427b2d1ddffa30f1c3e774d7/arch/x86/kernel/process_64.c#L239)소스 코드 파일에 있습니다.

`start_thread`함수는 [세그먼트 레지스터](https://en.wikipedia.org/wiki/X86_memory_segmentation) 와 프로그램 실행 주소에 새로운 값을 설정합니다. 이제 새 프로세스를 시작할 준비가 되었습니다. 일단 [컨텍스트 스위치](https://en.wikipedia.org/wiki/Context_switch)가 수행되고 컨트롤은 레지스터의 새로운 값으로 사용자 공간으로 반환되고 새로운 실행이 실행되기 시작합니다.

그것이 커널 측면의 전부입니다. 리눅스 커널은 실행을 위해 이진 이미지를 준비하고 컨텍스트 전환 직후 실행을 시작해 완료되면 사용자 공간으로 컨트롤을 반환합니다. 그러나 그것은 `_start`가 어디에서 왔는지와 같은 우리의 질문에 대답이 되지 못합니다. 다음 문단에서 이 질문들에 대답해 봅시다.

사용자 공간에서 프로그램이 시작되는 방법
--------------------------------------------------------------------------------

이전 단락에서 실행 파일이 리눅스 커널에 의해 실행되도록 준비된 방법을 봤습니다. 동일하지만, 사용자 공간 측면에서 살펴봅시다. 우리는 이미 각 프로그램의 진입점이 `_start`함수인 것을 압니다. 하지만 이 함수는 어디에서 왔을까요? 라이브러리에서 왔을 수도 있습니다. 그러나 정확히 기억한다면 우리는 우리의 프로그램을 컴파일 하는 동안 어떤 라이브러리도 프로그램에 연결하지 않았습니다:

```
$ gcc -Wall program.c -o sum
```

`_start`는 [표준 라이브러리](https://en.wikipedia.org/wiki/Standard_library)에서 나온 것으로 추측할 수 있고 그것은 사실입니다. 우리의 프로그램을 다시 컴파일하고 `-v`옵션을 gcc에 전달해 `verbose mode`를 활성화하면, 긴 출력을 볼 수 있습니다. 전체 출력은 우리에게 흥미롭지 않습니다. 다음의 단계를 살펴봅시다:

우선, 우리의 프로그램은 다음 `gcc`와 같이 컴파일 되어야합니다:

```
$ gcc -v -ggdb program.c -o sum
...
...
...
/usr/libexec/gcc/x86_64-redhat-linux/6.1.1/cc1 -quiet -v program.c -quiet -dumpbase program.c -mtune=generic -march=x86-64 -auxbase test -ggdb -version -o /tmp/ccvUWZkF.s
...
...
...
```

`cc1`컴파일러는 우리의 `C`소스코드를 컴파일하고 `/tmp/ccvUWZkF.s`라 명명된 어셈블리 파일을 생성합니다. 그 다음 `GNU as`어셈블러를 사용하여 어셈블리 파일이 오브젝트 파일로 컴파일 되는 것을 알 수 있습니다:

```
$ gcc -v -ggdb program.c -o sum
...
...
...
as -v --64 -o /tmp/cc79wZSU.o /tmp/ccvUWZkF.s
...
...
...
```

마지막에 우리의 오브젝트 파일은 `collect2`와 연결됩니다:

```
$ gcc -v -ggdb program.c -o sum
...
...
...
/usr/libexec/gcc/x86_64-redhat-linux/6.1.1/collect2 -plugin /usr/libexec/gcc/x86_64-redhat-linux/6.1.1/liblto_plugin.so -plugin-opt=/usr/libexec/gcc/x86_64-redhat-linux/6.1.1/lto-wrapper -plugin-opt=-fresolution=/tmp/ccLEGYra.res -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s -plugin-opt=-pass-through=-lc -plugin-opt=-pass-through=-lgcc -plugin-opt=-pass-through=-lgcc_s --build-id --no-add-needed --eh-frame-hdr --hash-style=gnu -m elf_x86_64 -dynamic-linker /lib64/ld-linux-x86-64.so.2 -o test /usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../../../lib64/crt1.o /usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../../../lib64/crti.o /usr/lib/gcc/x86_64-redhat-linux/6.1.1/crtbegin.o -L/usr/lib/gcc/x86_64-redhat-linux/6.1.1 -L/usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../../../lib64 -L/lib/../lib64 -L/usr/lib/../lib64 -L. -L/usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../.. /tmp/cc79wZSU.o -lgcc --as-needed -lgcc_s --no-as-needed -lc -lgcc --as-needed -lgcc_s --no-as-needed /usr/lib/gcc/x86_64-redhat-linux/6.1.1/crtend.o /usr/lib/gcc/x86_64-redhat-linux/6.1.1/../../../../lib64/crtn.o
...
...
...
```

예, 우리는 링커에 전달되는 긴 커맨드 라인 옵션의 집합을 볼 수 있습니다. 다른 방법으로 가봅시다. 우리는 프로그램이 `stdlib`에 따르는 것을 알고 있습니다:

```
$ ldd program
	linux-vdso.so.1 (0x00007ffc9afd2000)
	libc.so.6 => /lib64/libc.so.6 (0x00007f56b389b000)
	/lib64/ld-linux-x86-64.so.2 (0x0000556198231000)
```

우리는 `printf`등에서 온 일부를 사용합니다. 그러나 전부는 아닙니다. 그것이 `-nostdlib`옵션을 컴파일러에 전달할 때 오류를 얻는 이유입니다:

```
$ gcc -nostdlib program.c -o program
/usr/bin/ld: warning: cannot find entry symbol _start; defaulting to 000000000040017c
/tmp/cc02msGW.o: In function `main':
/home/alex/program.c:11: undefined reference to `printf'
collect2: error: ld returned 1 exit status
```

다른 오류외에도, 우리는 `_start`심볼이 정의되지 않을 것을 볼 수 있습니다. 이제 우리는 `_start`함수가 표준 라이브러리에서 온 것을 확신합니다. 그러나 표준 라이브러리와 연결해도, 성공적으로 컴파일되지 않습니다:

```
$ gcc -nostdlib -lc -ggdb program.c -o program
/usr/bin/ld: warning: cannot find entry symbol _start; defaulting to 0000000000400350
```

컴파일러는 `/usr/lib64/libc.so.6`로 프로그램과 연결된 어떠한 정의되지 않은 표준 라이브러리 함수의 언급에 관해 불평하지 않지만, `_start`심볼이 아직 해결되지 않았습니다. `gcc`의 verbose 출력으로 돌아가 `collect2`매개변수를 봅시다. 우리가 볼 수 있는 중요한 것은 우리의 프로그램이 표준 라이브러리 뿐 아니라 일부 오브젝트 파일과도 연결되어 있다는 것입니다.  첫 번째 오브젝트 파일은 `/lib64/crt1.o`입니다. `objdump`로 오브젝트 파일의 내부를 보면, `_start`심볼을 볼 수 있습니다:

```
$ objdump -d /lib64/crt1.o 

/lib64/crt1.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <_start>:
   0:	31 ed                	xor    %ebp,%ebp
   2:	49 89 d1             	mov    %rdx,%r9
   5:	5e                   	pop    %rsi
   6:	48 89 e2             	mov    %rsp,%rdx
   9:	48 83 e4 f0          	and    $0xfffffffffffffff0,%rsp
   d:	50                   	push   %rax
   e:	54                   	push   %rsp
   f:	49 c7 c0 00 00 00 00 	mov    $0x0,%r8
  16:	48 c7 c1 00 00 00 00 	mov    $0x0,%rcx
  1d:	48 c7 c7 00 00 00 00 	mov    $0x0,%rdi
  24:	e8 00 00 00 00       	callq  29 <_start+0x29>
  29:	f4                   	hlt    
```

`crt1.o`는 공유된 오브젝트 파일로 실제 호출 대신 스텁만 표시됩니다. `_start`함수의 소스 코드를 살펴봅시다. 이 함수는 특정한 아키텍쳐로 , `_start`에 대한 구현은 [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=f1b961f5ba2d6a1ebffee0005f43123c4352fbf4;hb=HEAD)어셈블리 파일에 있습니다.

`_start`는 레지스터 [ABI](https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf)가 제안한 `ebp`의 제거에서 시작합니다.

```assembly
xorl %ebp, %ebp
```

그리고 종료 함수의 주소를 `r9`레지스터에 넣습니다:

```assembly
mov %RDX_LP, %R9_LP
```

[ELF](http://flint.cs.yale.edu/cs422/doc/ELF_Format.pdf)설명서에 설명됐습니다:

> 동적 링커나 프로세스 이미지를 빌드하고 재배치를 수행 한 후, 각 공유 객체는
> 일부 초기화 코드를 실행한 기회를 얻습니다.
> ...
> 마찬가지로, 공유 객체에는 종료 기능이 있을 수 있으며, 기본 기능이 종료 시퀀스를 시작한 후에 atexit (BA_OS)메커니즘으로 실행됩니다.

따라서 나중에 여섯 번째 인자로 `__libc_start_main`에 전달될 종료 함수의 주소를 `r9`레지스터에 넣어야합니다.  종료 함수의 주소는 처음에 `rdx`레지스터에 있습니다. 이외에 다른 레지스터 `rdx`와 `rsp`는 지정되지 않은 값을 포함합니다. 실제로 `_start`함수의 요점은 `__libc_start_main`를 호출하는 것입니다. 따라서 다음에 이 함수를 위한 준비를 합니다.

`__libc_start_main`함수의 특징은 [csu/libc-start.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=csu/libc-start.c;h=9a56dcbbaeb7ef85c495b4df9ab1d0b13454c043;hb=HEAD#l107)소스 코드 파일에 있습니다. 살펴봅시다:

```C
STATIC int LIBC_START_MAIN (int (*main) (int, char **, char **),
 			                int argc,
			                char **argv,
 			                __typeof (main) init,
			                void (*fini) (void),
			                void (*rtld_fini) (void),
			                void *stack_end)
```

이것은 프로그램의 `main`함수의 주소에서 `argc` 및 `argv`를 가집니다. `init` 및 `fini`함수 생성자와 프로그램의 소멸자입니다. `rtld_fini`는 프로그램을 종료하고 동적 섹션을 해방한 다음 호출되는 종료 함수입니다. 마지막으로 `__libc_start_main`의 매개변수는 프로그램의 스택 포인터입니다. `__libc_start_main`함수를 호출하기 전에, 이 모든 매개변수를 준비하여 전달해야합니다. [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=f1b961f5ba2d6a1ebffee0005f43123c4352fbf4;hb=HEAD)어셈블리 파일로 돌아가서 `__libc_start_main`함수가 호출되기 전에 어떤일이 발생하는지 계속 살펴봅시다.

스택에서 `__libc_start_main`함수를 위해 필요한 모든 인자를 얻을 수 있습니다. 처음에 `_start`가 호출되면 스택은 다음과 같습니다:

```
+-----------------+
|       NULL      |
+-----------------+ 
|       ...       |
|       envp      |
|       ...       |
+-----------------+ 
|       NULL      |
+------------------
|       ...       |
|       argv      |
|       ...       |
+------------------
|       argc      | <- rsp
+-----------------+ 
```

`ebp`레지스터를 지우고 `r9`레지스터에서 종료 함수의 주소를 저장한 다음, 스택에서 `rsi`레지스터로 요소를 pop합니다. 따라서 `rsp`는 `argv`배열을 가리키고 `rsi`는 프로그램에 전달된 커맨드 라인 인수의 카운트를 포함합니다:

```
+-----------------+
|       NULL      |
+-----------------+ 
|       ...       |
|       envp      |
|       ...       |
+-----------------+ 
|       NULL      |
+------------------
|       ...       |
|       argv      |
|       ...       | <- rsp
+-----------------+
```

그 다음 `argv`배열의 주소를 `rdx`레지스터로 옮깁니다

```assembly
popq %rsi
mov %RSP_LP, %RDX_LP
```

이 순간부터 우리는 `argc`와 `argv`를 가집니다. 우리는 여전히 생성자, 소멸자에 대한 포인터를 적절한 레지스터에 넣고, 스택에 포인터를 전달해야합니다. 먼저 다음 세 줄에서 우리는 [ABI](https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf)에서 제안한대로 스텍을 `16`바이트 경계에 정렬하고 garbage를 포함하는 `rax`를 푸시합니다:

```assembly
and  $~15, %RSP_LP
pushq %rax

pushq %rsp
mov $__libc_csu_fini, %R8_LP
mov $__libc_csu_init, %RCX_LP
mov $main, %RDI_LP
```

스택이 정렬된 후 우리는 스택의 주소를 푸시하고, 생성자와 소멸자의 주소를 `r8` alc `rcx`로 옮기고 `main`심볼의 주소를 `rdi`로 옮깁니다. 이 순간부터 우리는 [csu/libc-start.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=csu/libc-start.c;h=0fb98f1606bab475ab5ba2d0fe08c64f83cce9df;hb=HEAD)에서 `__libc_start_main`함수를 호출할 수 있습니다.

`__libc_start_main`함수를 보기 전에, `/lib64/crt1.o`를 넣고 우리의 프로그램에 다시 컴파일해봅시다:

```
$ gcc -nostdlib /lib64/crt1.o -lc -ggdb program.c -o program
/lib64/crt1.o: In function `_start':
(.text+0x12): undefined reference to `__libc_csu_fini'
/lib64/crt1.o: In function `_start':
(.text+0x19): undefined reference to `__libc_csu_init'
collect2: error: ld returned 1 exit status
```

이제 `__libc_csu_fini` 및 `__libc_csu_init`함수를 둘 다 찾을 수 없다는 또 다른 오류를 봅니다. 우리는 이 두 함수의 주소가 매개변수처럼 `__libc_start_main`로 전달되고 이 함수는 우리의 프로그램의 생성자와 소멸자인 것을 알고 있습니다. 하지만 `constructor`와 `destructor`가 `C`프로그램의 측면에서 무엇을 의미할까요? 우리는 이미 [ELF](http://flint.cs.yale.edu/cs422/doc/ELF_Format.pdf)설명서에서 인용문을 봤습니다:

> 동적 링커나 프로세스 이미지를 빌드하고 재배치를 수행 한 후, 각 공유 객체는
> 일부 초기화 코드를 실행한 기회를 얻습니다.
> ...
> 마찬가지로, 공유 객체에는 종료 기능이 있을 수 있으며, 기본 기능이 종료 시퀀스>를 시작한 후에 atexit (BA_OS)메커니즘으로 실행됩니다.


따라서 링커는 일반 섹션외에도 `.text`, `.data` 및 나머지와 같은 두 가지 특별한 섹션을 만듭니다: 

* `.init`
* `.fini`

`readelf`유틸을 사용해 찾을 수 있습니다:

```
$ readelf -e test | grep init
  [11] .init             PROGBITS         00000000004003c8  000003c8

$ readelf -e test | grep fini
  [15] .fini             PROGBITS         0000000000400504  00000504
```

이 두 섹션은 이진 이미지의 시작과 끝에 배치되며 각각 생성자와 소멸자라고 불리는 루틴을 포함합니다. 이 루틴의 요점은 프로그램의 실제 코드가 실행되기 전에 [errno](http://man7.org/linux/man-pages/man3/errno.3.html)처럼 전역변수의 초기화와 같은 초기화/완료 및 시스템 루틴 등등에 대한 메모리 할당 및 할당취소를 수행하는 것입니다.

이러한 함수의 이름에서 유추할 수 있듯이, `main`함수의 앞뒤에서 호출됩니다. `.init`과 `.fini`섹션의 정의는 `/lib64/crti.o`에 위치됐으며 오브젝트 파일을 추가하면:

```
$ gcc -nostdlib /lib64/crt1.o /lib64/crti.o  -lc -ggdb program.c -o program
```

어떠한 오류도 얻지 않습니다. 하지만 우리의 프로그램을 실행하고 어떤 일이 일어나s는지 봅시다:

```
$ ./program
Segmentation fault (core dumped)
```

네, 분할 오류가 있습니다. `objdump`를 이용해 `lib64/crti.o`의 내부를 살펴봅시다:

```
$ objdump -D /lib64/crti.o

/lib64/crti.o:     file format elf64-x86-64


Disassembly of section .init:

0000000000000000 <_init>:
   0:	48 83 ec 08          	sub    $0x8,%rsp
   4:	48 8b 05 00 00 00 00 	mov    0x0(%rip),%rax        # b <_init+0xb>
   b:	48 85 c0             	test   %rax,%rax
   e:	74 05                	je     15 <_init+0x15>
  10:	e8 00 00 00 00       	callq  15 <_init+0x15>

Disassembly of section .fini:

0000000000000000 <_fini>:
   0:	48 83 ec 08          	sub    $0x8,%rsp
```

위에서 쓴 것처럼, `/lib64/crti.o`오브젝트 파일은 `.init`와 `.fini`섹션의 정의를 포함하지만, 함수를 위한 스텁도 볼 수 있습니다. [sysdeps/x86_64/crti.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/crti.S;h=e9d86ed08ab134a540e3dae5f97a9afb82cdb993;hb=HEAD)소스 코드 파일에 위치한 소스 코드를 살펴봅시다:

```assembly
	.section .init,"ax",@progbits
	.p2align 2
	.globl _init
	.type _init, @function
_init:
	subq $8, %rsp
	movq PREINIT_FUNCTION@GOTPCREL(%rip), %rax
	testq %rax, %rax
	je .Lno_weak_fn
	call *%rax
.Lno_weak_fn:
	call PREINIT_FUNCTION
```

여기에는 `.init`섹션의 정의가 포함되었으며 어셈브릴 코드는 16바이트 스택 정렬을 하고 다음으로 우리는 `PREINIT_FUNCTION`의 주소를 옮기고 이것이 0이면 호출하지 않습니다:

```
00000000004003c8 <_init>:
  4003c8:       48 83 ec 08             sub    $0x8,%rsp
  4003cc:       48 8b 05 25 0c 20 00    mov    0x200c25(%rip),%rax        # 600ff8 <_DYNAMIC+0x1d0>
  4003d3:       48 85 c0                test   %rax,%rax
  4003d6:       74 05                   je     4003dd <_init+0x15>
  4003d8:       e8 43 00 00 00          callq  400420 <__libc_start_main@plt+0x10>
  4003dd:       48 83 c4 08             add    $0x8,%rsp
  4003e1:       c3                      retq
```

`PREINIT_FUNCTION`는 프로파일에 대한 설정을 하는 `__gmon_start__`에 있습니다. [sysdeps/x86_64/crti.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/crti.S;h=e9d86ed08ab134a540e3dae5f97a9afb82cdb993;hb=HEAD)는는 리턴 명령을 가지지 않았습니다. 실제로 세그멘테이션 오류가 발생하는 이유입니다. `_init`와 `_fini`의 프롤로그는 [sysdeps/x86_64/crtn.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/crtn.S;h=e9d86ed08ab134a540e3dae5f97a9afb82cdb993;hb=HEAD)어셈블리 파일에 있습니다:

```assembly
.section .init,"ax",@progbits
addq $8, %rsp
ret

.section .fini,"ax",@progbits
addq $8, %rsp
ret
```

컴파일에 추가하면, 우리의 프로그램은 성공적으로 컴파일되고 실행됩니다!

```
$ gcc -nostdlib /lib64/crt1.o /lib64/crti.o /lib64/crtn.o  -lc -ggdb program.c -o program

$ ./program
x + y + z = 6
```

결론
--------------------------------------------------------------------------------

이제 `_start`함수로 돌아가 프로그램의 `main`이 호출되기 전에 전체 호출 체인을 시도해봅시다.

`_start`는 항상 디폴트 `ld`스크립트로 사용되는 연결된 프로그램에서 `.text`섹션의 시작에 배치됩니다:

```
$ ld --verbose | grep ENTRY
ENTRY(_start)
```

`_start`함수는 [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=f1b961f5ba2d6a1ebffee0005f43123c4352fbf4;hb=HEAD)어셈블리 파일에 정의되었으며 `__libc_start_main`함수가 호출되기 전에 스택에서 `argc/argv`가져오기, 스택 준비와 같은 준비를 합니다. [csu/libc-start.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=csu/libc-start.c;h=0fb98f1606bab475ab5ba2d0fe08c64f83cce9df;hb=HEAD)소스 코드 파일에서 `__libc_start_main`함수는 `main`과 다음 전에 호출된 응용프로그램의 생성자와 소멸자의 등록을 합니다. 스레딩을 시작하고, 필요한 경우 스택 canary 설정과 같은 일부 보안 관련 작업을 수행하고 초기화 관련 루틴을 호출하고 마지막에 응용프로그램의 `main`함수를 호출하고 결과와 함께 종료됩니다:

```C
result = main (argc, argv, __environ MAIN_AUXVEC_PARAM);
exit (result);
```

그것이 전부입니다.

링크
--------------------------------------------------------------------------------

* [system call](https://en.wikipedia.org/wiki/System_call)
* [gdb](https://www.gnu.org/software/gdb/)
* [execve](http://linux.die.net/man/2/execve)
* [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
* [x86_64](https://en.wikipedia.org/wiki/X86-64)
* [segment registers](https://en.wikipedia.org/wiki/X86_memory_segmentation)
* [context switch](https://en.wikipedia.org/wiki/Context_switch)
* [System V ABI](https://software.intel.com/sites/default/files/article/402129/mpx-linux64-abi.pdf)
