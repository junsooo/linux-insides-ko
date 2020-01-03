소개
---------------

[linux-insides](https://0xax.gitbooks.io/linux-insides/content/) 책을 쓰는 중에 [링커](https://en.wikipedia.org/wiki/Linker_%28computing%29) 스크립트 및 링커 관련 주제와 관련된 질문이 있는 많은 이메일을 받았습니다. 그래서 나는 링커의 일부 측면과 객체 파일의 링크를 다루기 위해 이것을 작성하기로 결정했습니다.

Wikipedia에서 `링커` 페이지를 열면 다음과 같은 정의가 나타납니다:

>컴퓨터 과학에서 링커 또는 링크 편집기는 컴파일러에서 생성 된 하나 이상의 객체 파일을 하나의 실행 파일, 라이브러리 파일 또는 다른 객체 파일로 결합하는 컴퓨터 프로그램입니다.

당신이 C에서 최소한 하나의 프로그램을 작성했다면, 확장자가 `*.o` 인 파일을 보게 될 것입니다. 이러한 파일은 [목적 파일](https://en.wikipedia.org/wiki/Object_file)입니다. 목적 파일은 다른 목적 파일 또는 라이브러리의 데이터와 함수을 참조하는 자리 표시 자 주소 및 자체 기능 및 데이터 목록이 있는 머신 코드 및 데이터 블록입니다. 링커의 주요 목적은 각 목적 파일의 코드와 데이터를 수집/처리하여 최종 실행 파일 또는 라이브러리로 변환하는 것 입니다. 이 포스트에서는 이 프로세스의 모든 측면을 살펴볼 것입니다. 시작합니다.

링킹 과정
---------------

다음과 같은 구조로 간단한 프로젝트를 만들어 봅시다:

```
*-linkers
*--main.c
*--lib.c
*--lib.h
```

우리의 `main.c` 소스 코드 파일은 다음을 포함합니다:

```C
#include <stdio.h>

#include "lib.h"

int main(int argc, char **argv) {
	printf("factorial of 5 is: %d\n", factorial(5));
	return 0;
}
```

`lib.c` 파일은 다음을 포함합니다:

```C
int factorial(int base) {
	int res,i = 1;

	if (base == 0) {
		return 1;
	}

	while (i <= base) {
		res *= i;
		i++;
	}

	return res;
}
```

그리고 `lib.h` 파일은 다음을 포함합니다:

```C
#ifndef LIB_H
#define LIB_H

int factorial(int base);

#endif
```

이제 다음과 같이 `main.c` 소스 코드 파일만 컴파일 해 봅시다:

```
$ gcc -c main.c
```

`nm` 유틸리티를 사용하여 출력된 오브젝트 파일을 살펴보면 다음 출력을 볼 수 있습니다:

```
$ nm -A main.o
main.o:                 U factorial
main.o:0000000000000000 T main
main.o:                 U printf
```

`nm` 유틸리티를 사용하면 주어진 목적 파일에서 심볼 목록을 볼 수 있습니다. 세 개의 열로 구성됩니다. 첫 번째는 주어진 목적 파일의 이름과 해석 된 심볼의 주소입니다. 두 번째 열에는 주어진 기호의 상태를 나타내는 문자가 포함됩니다. 이 경우 `U`는 `정의되지 않음`을 의미하고 `T`는 심볼이 객체의 `.text` 섹션에 배치됨을 나타냅니다. `nm` 유틸리티는 여기서 `main.c` 소스 코드 파일에 세 개의 심볼이 있음을 보여줍니다:

* `factorial` - `lib.c` 소스 코드 파일에 정의 된 계승 함수 여기서는 `main.c` 소스 코드 파일만 컴파일했기 때문에 `정의되지 않음` 으로 표시되어 있으며, 현재는 `lib.c` 파일의 코드에 대해서는 아무것도 모릅니다.;
* `main` - 메인 함수;
* `printf` - [glibc](https://en.wikipedia.org/wiki/GNU_C_Library) 라이브러리의 함수 `main.c`는 지금도 그것에 대해 아무것도 모른다.

지금까지 `nm`의 출력에서 무엇을 이해할 수 있습니까? `main.o` 목적 파일은 주소 `0000000000000000`에 로컬 기호 `main`(연결된 후 올바른 주소로 채워짐)과 두 개의 해석되지 않은 기호를 포함합니다. `main.o` 목적 파일의 디스 어셈블리 출력에서이 모든 정보를 볼 수 있습니다:

```
$ objdump -S main.o

main.o:     file format elf64-x86-64
Disassembly of section .text:

0000000000000000 <main>:
   0:	55                   	push   %rbp
   1:	48 89 e5             	mov    %rsp,%rbp
   4:	48 83 ec 10          	sub    $0x10,%rsp
   8:	89 7d fc             	mov    %edi,-0x4(%rbp)
   b:	48 89 75 f0          	mov    %rsi,-0x10(%rbp)
   f:	bf 05 00 00 00       	mov    $0x5,%edi
  14:	e8 00 00 00 00       	callq  19 <main+0x19>
  19:	89 c6                	mov    %eax,%esi
  1b:	bf 00 00 00 00       	mov    $0x0,%edi
  20:	b8 00 00 00 00       	mov    $0x0,%eax
  25:	e8 00 00 00 00       	callq  2a <main+0x2a>
  2a:	b8 00 00 00 00       	mov    $0x0,%eax
  2f:	c9                   	leaveq
  30:	c3                   	retq   
```

여기서 우리는 두 개의 `callq`연산에만 관심이 있습니다. 두 개의 `callq` 오퍼레이션에는 링커 스텁 또는 함수 이름 및 다음 명령으로의 오프셋이 포함됩니다. 이 스텁은 함수의 실제 주소로 업데이트됩니다. 다음 `objdump` 출력에서 이러한 함수의 이름을 볼 수 있습니다:

```
$ objdump -S -r main.o

...
  14:	e8 00 00 00 00       	callq  19 <main+0x19>
  15: R_X86_64_PC32	               factorial-0x4
  19:	89 c6                	mov    %eax,%esi
...
  25:	e8 00 00 00 00       	callq  2a <main+0x2a>
  26:   R_X86_64_PC32	               printf-0x4
  2a:	b8 00 00 00 00       	mov    $0x0,%eax
...
```

`objdump` 유틸리티의 `-r` 또는 `--reloc`플래그는 파일의 `relocation` 항목을 인쇄합니다. 이제 재배치 과정을 자세히 살펴 보겠습니다.

재배치
------------

재배치는 기호 참조를 기호 정의와 연결하는 프로세스입니다. `objdump` 출력에서 이전 코드 조각을 보자:

```
  14:	e8 00 00 00 00       	callq  19 <main+0x19>
  15:   R_X86_64_PC32	               factorial-0x4
  19:	89 c6                	mov    %eax,%esi
```

첫 번째 줄의 `e8 00 00 00 00`에 주목하십시오. `e8`은 `call`의 [opcode](https://en.wikipedia.org/wiki/Opcode)이며, 나머지 줄은 상대 오프셋입니다. 따라서 `e8 00 00 00 00`에는 1 바이트 연산 코드와 4 바이트 주소가 포함됩니다. `00 00 00 00`은 4 바이트입니다. `x86_64` (64 비트) 시스템에서 주소가 8 바이트 일 수있는 경우 왜 4 바이트입니까? 실제로 우리는 `gcc` 맨 페이지에서 `-mcmodel = small`로 `main.c`소스 코드 파일을 컴파일했습니다!:

```
-mcmodel=small

작은 코드 모델에 대한 코드를 생성하십시오. 프로그램 및 해당 심볼은 주소 공간의 하위 2GB에 연결되어야합니다. 포인터는 64 비트이다. 프로그램은 정적으로 또는 동적으로 링크 될 수 있습니다. 이것은 기본 코드 모델입니다.
```

물론 우리는 `main.c`를 컴파일 할 때이 옵션을 `gcc`에 전달하지 않았지만 기본값입니다. 우리는 프로그램이 위의 `gcc` 수동 추출에서 2GB의 주소 공간에 연결될 것임을 알고 있습니다. 따라서 4 바이트이면 충분합니다. 따라서 우리는 `call` 명령어의 opcode와 알 수 없는 주소를 가지고 있습니다. 실행 파일에 대한 모든 의존성을 가진 `main.c`를 컴파일 한 다음, 계승 호출을 살펴보면:

```
$ gcc main.c lib.c -o factorial | objdump -S factorial | grep factorial

factorial:     file format elf64-x86-64
...
...
0000000000400506 <main>:
	40051a:	e8 18 00 00 00       	callq  400537 <factorial>
...
...
0000000000400537 <factorial>:
	400550:	75 07                	jne    400559 <factorial+0x22>
	400557:	eb 1b                	jmp    400574 <factorial+0x3d>
	400559:	eb 0e                	jmp    400569 <factorial+0x32>
	40056f:	7e ea                	jle    40055b <factorial+0x24>
...
...
```

우리가 이전의 출력에서 볼 수 있듯이, `main` 함수의 주소는 `0x0000000000400506`입니다. 왜 `0x0`에서 시작하지 않습니까? 표준 C 프로그램이 `glibc` C 표준 라이브러리와 연결되어 있다는 것을 이미 알고있을 것입니다 (`-nostdlib`가 `gcc`에 전달되지 않았다고 가정). 프로그램의 컴파일 된 코드에는 프로그램이 시작될 때 프로그램에서 데이터를 초기화하는 생성자 함수가 포함되어 있습니다. 이 함수들은 프로그램이 시작되기 전에, 또는 다른 말로 `main` 함수가 호출되기 전에 호출되어야합니다. 초기화 및 종료 기능이 작동하려면 어셈블러 코드에서 컴파일러 출력해야 뭔가 그 기능이 적절한 시간에 호출되도록합니다. 이 프로그램의 실행은 특수 `.init` 섹션에있는 코드에서 시작합니다. objdump 출력의 시작 부분에서 이것을 볼 수 있습니다:

```
objdump -S factorial | less

factorial:     file format elf64-x86-64

Disassembly of section .init:

00000000004003a8 <_init>:
  4003a8:       48 83 ec 08             sub    $0x8,%rsp
  4003ac:       48 8b 05 a5 05 20 00    mov    0x2005a5(%rip),%rax        # 600958 <_DYNAMIC+0x1d0>
```

그것이 `glibc` 코드에 상대적인 `0x00000000004003a8` 주소에서 시작한다는 것은 아닙니다. `readelf`를 실행하여 [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) 출력에서도 확인할 수 있습니다:

```
$ readelf -d factorial | grep \(INIT\)
 0x000000000000000c (INIT)               0x4003a8
 ```

따라서 `main` 함수의 주소는 `0000000000400506`이며 `.init` 섹션에서 오프셋됩니다. 출력에서 알 수 있듯이 `factorial` 함수의 주소는 `0x0000000000400537`이고 `factorial` 함수 호출을 위한 이진 코드는 이제 `e8 18 00 00 00`입니다. 우리는 이미 `e8`이 `call` 명령어의 opcode이고, 다음 `18 00 00 00` (`x86_64`의 주소는 리틀 엔디안으로 표시되므로 `00 00 00 18`입니다) `callq`에서 `factorial` 함수로:

```python
>>> hex(0x40051a + 0x18 + 0x5) == hex(0x400537)
True
```

따라서 우리는 `call` 명령어의 주소에 `0x18`과 `0x5`를 추가합니다. 오프셋은 다음 명령어의 주소에서 측정됩니다. 우리의 호출 명령은 5 바이트 길이 (`e8 18 00 00 00`)이고 `0x18`은 `factorial` 함수 이후 호출의 오프셋입니다. 컴파일러는 일반적으로 프로그램 주소가 0에서 시작하여 각 객체 파일을 만듭니다. 그러나 프로그램이 여러 목적 파일로 생성되면 파일이 겹칩니다.

이 섹션에서 본 것은 재배치 프로세스입니다. 이 프로세스는 프로그램의 여러 부분에 로드 주소를 할당하고 할당 된 주소를 반영하도록 프로그램의 코드와 데이터를 조정합니다.

이제 링커 및 재배치에 대해 조금 알고 있으므로 목적 파일을 연결하여 링커에 대해 자세히 알아볼 차례입니다.

GNU 링커
-----------------

제목에서 알 수 있듯이이 게시물에서 [GNU 링커](https://en.wikipedia.org/wiki/GNU_linker) 또는 `ld`를 사용하겠습니다. 물론 우리는 `gcc`를 사용하여 `factorial` 프로젝트를 연결할 수 있습니다:

```
$ gcc main.c lib.o -o factorial
```

그 후에 우리는 실행 파일 인 `factorial`을 얻게됩니다:

```
./factorial
factorial of 5 is: 120
```

그러나 `gcc`는 목적 파일을 링크하지 않습니다. 대신 `GNU ld` 링커의 래퍼 인 `collect2`를 사용합니다:

```
~$ /usr/lib/gcc/x86_64-linux-gnu/4.9/collect2 --version
collect2 version 4.9.3
/usr/bin/ld --version
GNU ld (GNU Binutils for Debian) 2.25
...
...
...
```

좋습니다, 우리는 gcc를 사용할 수 있고 우리를 위해 프로그램의 실행 파일을 생성 할 것입니다. 그러나 동일한 목적으로 `GNU ld` 링커를 사용하는 방법을 살펴 보자. 우선, 이 객체 파일들을 다음 예제와 연결해 봅시다:

```
ld main.o lib.o -o factorial
```

시도하면 다음과 같은 오류가 발생합니다:

```
$ ld main.o lib.o -o factorial
ld: warning: cannot find entry symbol _start; defaulting to 00000000004000b0
main.o: In function `main':
main.c:(.text+0x26): undefined reference to `printf'
```

여기서 우리는 두 가지 문제를 볼 수 있습니다:

* 링커에서 `_start` 기호를 찾을 수 없습니다;
* 링커는`printf` 기능에 대해 아무것도 모릅니다.

우선 프로그램을 실행하기 위해 필요한이 `_start` 엔트리 심볼이 무엇인지 이해하려고합니까? 프로그래밍을 배우기 시작했을 때 나는 `main` 기능이 프로그램의 진입 점이라는 것을 알게되었습니다. 여러분도 이것을 배웠다고 생각합니다 :) 그러나 그것은 실제로 진입 점이 아니며, 대신 `_start`입니다. `_start` 기호는 `crt1.o` 오브젝트 파일에 정의되어 있습니다. 다음 명령으로 찾을 수 있습니다:

```
$ objdump -S /usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o

/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <_start>:
   0:	31 ed                	xor    %ebp,%ebp
   2:	49 89 d1             	mov    %rdx,%r9
   ...
   ...
   ...
```

이 목적 파일을 첫 번째 인수로 `ld` 명령에 전달합니다(위 참조). 이제 연결을 시도하고 결과를 살펴 보겠습니다.

```
ld /usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o \
main.o lib.o -o factorial

/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o: In function `_start':
/tmp/buildd/glibc-2.19/csu/../sysdeps/x86_64/start.S:115: undefined reference to `__libc_csu_fini'
/tmp/buildd/glibc-2.19/csu/../sysdeps/x86_64/start.S:116: undefined reference to `__libc_csu_init'
/tmp/buildd/glibc-2.19/csu/../sysdeps/x86_64/start.S:122: undefined reference to `__libc_start_main'
main.o: In function `main':
main.c:(.text+0x26): undefined reference to `printf'
```

불행히도 더 많은 오류가 나타납니다. 여기서는 정의되지 않은 `printf`와 또 다른 세 가지 정의되지 않은 참조에 대한 오래된 오류를 볼 수 있습니다:

* `__libc_csu_fini`
* `__libc_csu_init`
* `__libc_start_main`


`_start` 기호는 `glibc` 소스 코드의 [sysdeps/x86_64/start.S](https://sourceware.org/git/?p=glibc.git;a=blob;f=sysdeps/x86_64/start.S;h=0d27a38e9c02835ce17d1c9287aa01be222e72eb;hb=HEAD) 어셈블리 파일에 정의 되어있습니다. 다음 어셈블리 코드 라인을 찾을 수 있습니다:

```assembly
mov $__libc_csu_fini, %R8_LP
mov $__libc_csu_init, %RCX_LP
...
call __libc_start_main
```

여기서 우리는 진입 점의 주소를 프로그램이 실행될 때 실행을 시작하는 코드와 프로그램이 종료 될 때 실행되는 코드를 포함하는 `.init` 및 `.fini` 섹션으로 전달합니다. 그리고 결국 우리는 프로그램에서 `main` 함수의 호출을 보게됩니다. 이 세 가지 기호는 [csu/elf-init.c](https://sourceware.org/git/?p=glibc.git;a=blob;f=csu/elf-init.c;hb=1d4bbc54bd4f7d85d774871341b49f4357af1fb7) 소스 코드 파일 입니다. 다음 두 목적 파일:

* `crtn.o`;
* `crti.o`.

.init 및 .fini 섹션에 대한 함수 prologs/epilogs를 정의하십시오 (각각`_init` 및`_fini` 기호 사용).

`crtn.o` 목적 파일에는 다음 `.init` 및 `.fini` 섹션이 포함됩니다:

```
$ objdump -S /usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crtn.o

0000000000000000 <.init>:
   0:	48 83 c4 08          	add    $0x8,%rsp
   4:	c3                   	retq   

Disassembly of section .fini:

0000000000000000 <.fini>:
   0:	48 83 c4 08          	add    $0x8,%rsp
   4:	c3                   	retq   
```

그리고 `crti.o` 객체 파일에는 `_init`와 `_fini` 기호가 들어 있습니다. 이 두 목적 파일과 다시 연결해 봅시다:

```
$ ld \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crti.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crtn.o main.o lib.o \
-o factorial
```

어쨌든 같은 오류가 발생합니다. 이제 `-lc` 옵션을 `ld`에 전달해야합니다. 이 옵션은 `$ LD_LIBRARY_PATH` 환경 변수에 존재하는 경로에서 표준 라이브러리를 검색합니다. `-lc` 옵션으로 다시 연결해 봅시다:

```
$ ld \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crti.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crtn.o main.o lib.o -lc \
-o factorial
```

마지막으로 실행 파일을 얻지만 실행하려고 하면 이상한 결과가 나타납니다:

```
$ ./factorial
bash: ./factorial: No such file or directory
```

여기의 문제는 무엇일까요? [readelf](https://sourceware.org/binutils/docs/binutils/readelf.html) 유틸리티를 사용하여 실행 파일을 살펴 보겠습니다:

```
$ readelf -l factorial

Elf file type is EXEC (Executable file)
Entry point 0x4003c0
There are 7 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000400040 0x0000000000400040
                 0x0000000000000188 0x0000000000000188  R E    8
  INTERP         0x00000000000001c8 0x00000000004001c8 0x00000000004001c8
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
  LOAD           0x0000000000000000 0x0000000000400000 0x0000000000400000
                 0x0000000000000610 0x0000000000000610  R E    200000
  LOAD           0x0000000000000610 0x0000000000600610 0x0000000000600610
                 0x00000000000001cc 0x00000000000001cc  RW     200000
  DYNAMIC        0x0000000000000610 0x0000000000600610 0x0000000000600610
                 0x0000000000000190 0x0000000000000190  RW     8
  NOTE           0x00000000000001e4 0x00000000004001e4 0x00000000004001e4
                 0x0000000000000020 0x0000000000000020  R      4
  GNU_STACK      0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x0000000000000000 0x0000000000000000  RW     10

 Section to Segment mapping:
  Segment Sections...
   00     
   01     .interp
   02     .interp .note.ABI-tag .hash .dynsym .dynstr .gnu.version .gnu.version_r .rela.dyn .rela.plt .init .plt .text .fini .rodata .eh_frame
   03     .dynamic .got .got.plt .data
   04     .dynamic
   05     .note.ABI-tag
   06     
```

이상한 줄에 주의하십시오:

```
  INTERP         0x00000000000001c8 0x00000000004001c8 0x00000000004001c8
                 0x000000000000001c 0x000000000000001c  R      1
      [Requesting program interpreter: /lib64/ld-linux-x86-64.so.2]
```

`elf` 파일의 `.interp` 섹션은 프로그램 인터프리터의 경로 이름을 가지고 있거나 다른 말로 `.interp` 섹션은 동적 링커의 이름 인 `ascii`문자열을 포함합니다. 동적 링커는 라이브러리의 내용을 디스크에서 RAM으로 복사하여 실행 파일이 실행될 때 필요한 공유 라이브러리를 로드하고 링크하는 Linux의 일부입니다. `readelf` 명령의 출력에서 볼 수 있듯이 `x86_64` 아키텍처의 `/lib64/ld-linux-x86-64.so.2`파일에 위치합니다. 이제 `ld-linux-x86-64.so.2` 경로를 가진 `-dynamic-linker` 옵션을 `ld` 호출에 추가하고 다음 결과를 보게됩니다:

```
$ gcc -c main.c lib.c

$ ld \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crti.o \
/usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crtn.o main.o lib.o \
-dynamic-linker /lib64/ld-linux-x86-64.so.2 \
-lc -o factorial
```

이제 일반 실행 파일로 실행할 수 있습니다:

```
$ ./factorial

factorial of 5 is: 120
```

작동합니다! 첫 번째 줄에서 우리는 `main.c`와 `lib.c` 소스 코드 파일을 목적 파일로 컴파일합니다. `gcc`를 실행 한 후에 `main.o`와 `lib.o`를 얻습니다:

```
$ file lib.o main.o
lib.o:  ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
main.o: ELF 64-bit LSB relocatable, x86-64, version 1 (SYSV), not stripped
```

그리고 나서 프로그램의 목적 파일을 필요한 시스템 목적 파일 및 라이브러리와 연결합니다. 우리는 `gcc` 컴파일러와 `GNU ld` 링커로 C 프로그램을 컴파일하고 링크하는 간단한 예제를 보았습니다. 이 예에서는 `GNU linker`의 몇 가지 명령 행 옵션을 사용했지만 `-o`,`-dynamic-linker` 등보다 훨씬 더 많은 명령 행 옵션을 지원합니다. 또한 `GNU ld`는 연결 프로세스를 제어할 수있는 고유 한 언어 다음 두 단락에서 살펴볼 것입니다.

GNU 링커의 유용한 명령 행 옵션
----------------------------------------------

이미 쓴 것처럼 `GNU linker`의 매뉴얼에서 볼 수 있듯이, 명령 행 옵션이 많이 있습니다. 이 포스트에서 `-o <output>`옵션을 보았습니다: `-o <output>`-`ld`에게 링크 결과로 `output`라는 출력 파일을 생성하도록 지시합니다.`-l <name>`는 동적 링커의 이름을 지정하는 `-dynamic-linker`라는 이름으로 지정된 아카이브 또는 목적 파일. 물론 `ld`는 훨씬 더 많은 명령 행 옵션을 지원합니다.

가장 유용한 명령 행 옵션은 `@ file`입니다. 이 경우 `file`은 명령 줄 옵션을 읽을 파일 이름을 지정합니다. 예를 들어 `linker.ld`라는 이름으로 파일을 생성하고 이전 예제의 명령 줄 인자를 넣고 다음과 같이 실행할 수 있습니다:

```
$ ld @linker.ld
```

다음 명령 행 옵션은 `-b` 또는 `--format`입니다. 이 명령 행 옵션은 입력 오브젝트 파일 `ELF`, `DJGPP/COFF` 등의 형식을 지정합니다. 동일한 목적이지만 출력 파일에 대한 명령 행 옵션이 있습니다: `--oformat = output-format`.

다음 명령 행 옵션은 `--defsym`입니다. 이 명령 행 옵션의 전체 형식은 `--defsym = symbol = expression`입니다. 식으로 지정된 절대 주소를 포함하는 출력 파일에 전역 기호를 만들 수 있습니다. 이 명령 행 옵션이 유용 할 수있는 다음과 같은 경우를 찾을 수 있습니다. Linux 커널 소스 코드 및 ARM 아키텍처의 커널 압축 해제와 관련된 Makefile-[arch/arm/boot/compressed/Makefile]( https://github.com/torvalds/linux/blob16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/arm/boot/compressed/Makefile), 우리는 다음 정의를 찾을 수 있습니다:

```
LDFLAGS_vmlinux = --defsym _kernel_bss_size=$(KBSS_SZ)
```

이미 알고 있듯이 출력 파일의 `.bss` 섹션 크기로 `_kernel_bss_size` 심볼을 정의합니다. 이 기호는 커널 압축 해제 중에 실행될 첫 번째 [어셈블리 파일](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/arm/boot/compressed/head.S)에서 사용됩니다:

```assembly
ldr r5, =_kernel_bss_size
```

다음 명령 행 옵션은 공유 라이브러리를 만들 수 있는 `공유` 입니다. `-M` 또는 `-map <filename>`명령 행 옵션은 심볼에 대한 정보와 함께 링크 맵을 인쇄합니다. 우리의 경우 :

```
$ ld -M @linker.ld
...
...
...
.text           0x00000000004003c0      0x112
 *(.text.unlikely .text.*_unlikely .text.unlikely.*)
 *(.text.exit .text.exit.*)
 *(.text.startup .text.startup.*)
 *(.text.hot .text.hot.*)
 *(.text .stub .text.* .gnu.linkonce.t.*)
 .text          0x00000000004003c0       0x2a /usr/lib/gcc/x86_64-linux-gnu/4.9/../../../x86_64-linux-gnu/crt1.o
...
...
...
 .text          0x00000000004003ea       0x31 main.o
                0x00000000004003ea                main
 .text          0x000000000040041b       0x3f lib.o
                0x000000000040041b                factorial
```

물론 `GNU 링커`는 표준 명령 행 옵션 인 `--help`와 `--version`을 지원하여 `ld`와 그 버전의 사용법에 대한 일반적인 도움말을 출력합니다. 이것이 `GNU 링커`의 명령 행 옵션에 관한 것입니다. 물론 `ld` 유틸리티가 지원하는 전체 명령 행 옵션이 아닙니다. 매뉴얼에서 `ld` 유틸리티의 전체 문서를 찾을 수 있습니다.

제어 언어 링커
----------------------------------------------

앞에서 쓴 것처럼 `ld`는 자신의 언어를 지원합니다. AT & T의 링크 편집기 명령 언어 구문의 상위 세트로 작성된 링커 명령 언어 파일을 허용하여 링크 프로세스에 대한 명시적이고 완전한 제어를 제공합니다. 세부 사항을 살펴 봅시다.

우리가 통제할 수 있는 링커 언어로:

* input files;
* output files;
* file formats
* addresses of sections;
* etc...

링커 제어 언어로 작성된 명령은 일반적으로 링커 스크립트라는 파일에 배치됩니다. `-T` 명령 행 옵션을 사용하여 `ld`에 전달할 수 있습니다. 링커 스크립트의 주요 명령은 `SECTIONS` 명령입니다. 각 링커 스크립트는 이 명령을 포함해야하며 출력 파일의 `맵`을 결정합니다. 특수 변수 `.`는 출력의 현재 위치를 포함합니다. 간단한 어셈블리 프로그램을 작성하고 링커 스크립트를 사용하여이 프로그램의 링크를 제어하는 방법을 살펴 보겠습니다. 이 예제에서는 hello world 프로그램을 사용합니다:

```assembly
.data
        msg:    .ascii  "hello, world!\n"

.text

.global _start

_start:
        mov    $1,%rax
        mov    $1,%rdi
        mov    $msg,%rsi
        mov    $14,%rdx
        syscall

        mov    $60,%rax
        mov    $0,%rdi
        syscall
```

다음 명령으로 컴파일하고 연결할 수 있습니다:

```
$ as -o hello.o hello.asm
$ ld -o hello hello.o
```

우리의 프로그램은 두 개의 섹션으로 구성됩니다: `.text`는 프로그램 코드를 포함하고 `.data`는 초기화 된 변수를 포함합니다. 간단한 링커 스크립트를 작성하고 `hello.asm` 어셈블리 파일을 링크 해 봅시다. 우리의 스크립트는 다음과 같습니다:

```
/*
 * Linker script for the factorial
 */
OUTPUT(hello)
OUTPUT_FORMAT("elf64-x86-64")
INPUT(hello.o)

SECTIONS
{
	. = 0x200000;
	.text : {
	      *(.text)
	}

	. = 0x400000;
	.data : {
	      *(.data)
	}
}
```

처음 세 줄에는 `C` 스타일로 작성된 주석이 있습니다. 그 후 `OUTPUT`과 `OUTPUT_FORMAT` 명령은 실행 파일의 이름과 형식을 지정합니다. 다음 명령 `INPUT`은 `ld` 링커에 대한 입력 파일을 지정합니다. 그런 다음, 이미 작성한 것처럼 모든 링커 스크립트에 있어야 하는 기본 `SECTIONS` 명령을 볼 수 있습니다. `SECTIONS` 명령은 출력 파일에 있을 섹션의 세트와 순서를 나타냅니다. `SECTIONS` 명령의 시작 부분에서 다음 줄을 볼 수 있습니다 `.= 0x200000`. 나는 이미 `.` 명령이 출력의 현재 위치를 가리키고 있다고 썼다. 이 줄은 코드가 주소 `0x200000`과 줄에로드되어야한다고 말합니다 `. = 0x400000`은 데이터 섹션이 주소 `0x400000`에 로드되어야한다고 말합니다. 다음의 두 번째 줄 `. = 0x200000`는 `.text`를 출력 섹션으로 정의합니다. 내부에 `* (. text)`표현이 있습니다. `*`기호는 모든 파일 이름과 일치하는 와일드 카드입니다. 다시 말해, `* (. text)`표현식은 모든 입력 파일의 모든 `.text` 입력 섹션을 나타냅니다. 이 예에서는 `hello.o (.text)`로 다시 쓸 수 있습니다. 다음 위치 카운터 이후 `. = 0x400000`, 우리는 데이터 섹션의 정의를 볼 수 있습니다.

다음 명령으로 컴파일하고 연결할 수 있습니다:

```
$ as -o hello.o hello.S && ld -T linker.script && ./hello
hello, world!
```

`objdump` 유틸리티로 내부를 살펴보면 `.text` 섹션이 `0x200000` 주소에서 시작하고 `.data` 섹션이 `0x400000` 주소에서 시작한다는 것을 알 수 있습니다:

```
$ objdump -D hello

Disassembly of section .text:

0000000000200000 <_start>:
  200000:	48 c7 c0 01 00 00 00 	mov    $0x1,%rax
  ...

Disassembly of section .data:

0000000000400000 <msg>:
  400000:	68 65 6c 6c 6f       	pushq  $0x6f6c6c65
  ...
```

우리가 이미 본 명령 외에도 몇 가지 다른 것들이 있습니다. 첫 번째는 주어진 표현식이 0이 아닌 ASSERT (exp, message)입니다. 0이 아니면 오류 코드와 함께 링커를 종료하고 주어진 오류 메시지를 인쇄하십시오. [linux-insides](https://0xax.gitbooks.io/linux-insides/content/) 서적에서 Linux 커널 부팅 프로세스에 대해 읽은 경우 Linux 커널의 설정 헤더에 오프셋이 있음을 알고 `0x1f1`. Linux 커널의 링커 스크립트에서 다음을 확인할 수 있습니다.

```
. = ASSERT(hdr == 0x1f1, "The setup header has the wrong offset!");
```

`INCLUDE filename` 명령을 사용하면 현재 링커 스크립트 심볼을 외부 링커 스크립트 심볼에 포함시킬 수 있습니다. 링커 스크립트에서 심볼에 값을 할당 할 수 있습니다. `ld`는 몇 가지 할당 연산자를 지원합니다:

* symbol = expression   ;
* symbol += expression  ;
* symbol -= expression  ;
* symbol *= expression  ;
* symbol /= expression  ;
* symbol <<= expression ;
* symbol >>= expression ;
* symbol &= expression  ;
* symbol |= expression  ;

알 수 있듯이 모든 연산자는 C 할당 연산자입니다. 예를 들어 링커 스크립트에서 다음과 같이 사용할 수 있습니다:

```
START_ADDRESS = 0x200000;
DATA_OFFSET   = 0x200000;

SECTIONS
{
	. = START_ADDRESS;
	.text : {
	      *(.text)
	}

	. = START_ADDRESS + DATA_OFFSET;
	.data : {
	      *(.data)
	}
}
```

이미 언급했듯이 링커 스크립트 언어의 식 구문은 C 식의 구문과 동일합니다. 이 외에도 링크의 제어 언어는 다음과 같은 내장 함수를 지원합니다:

* `ABSOLUTE` - 주어진 표현식의 절대 값을 반환;
* `ADDR` - 섹션을 가져 와서 주소를 반환;
* `ALIGN` - 주어진 표현식 이후 다음 표현식의 경계에 의해 정렬 된 위치 카운터 (`.` 연산자)의 값을 반환;
* `DEFINED` - 주어진 심볼이 전역 심볼 테이블에 있으면 `1`을, 다른 방법으로 `0`을 반환;
* `MAX` and `MIN` - 주어진 두 표현식의 최대 값과 최소값을 반환;
* `NEXT` - give 표현식의 배수 인 할당되지 않은 다음 주소를 반환;
* `SIZEOF` - 지정된 명명 된 섹션의 크기를 바이트 단위로 반환.

그게 답니다.

결론
-----------------

이것은 링커에 대한 게시물의 끝입니다. 우리는 이 글에서 링커란 무엇이고 왜 필요한지, 어떻게 사용하는지 등 링커에 대해 많은 것을 배웠습니다.

질문이나 제안이 있으시면 Twitter에 [email](kuleshovmail@gmail.com) 또는 ping [me](https://twitter.com/0xAX)를 작성하십시오.

영어는 모국어가 아니며 불편을 끼쳐 드려 죄송합니다. 당신이 실수를 발견하면 이메일을 통해 알려주거나 PR을 보내주십시오.

링크
-----------------

* [Book about Linux kernel insides](https://0xax.gitbooks.io/linux-insides/content/)
* [linker](https://en.wikipedia.org/wiki/Linker_%28computing%29)
* [object files](https://en.wikipedia.org/wiki/Object_file)
* [glibc](https://en.wikipedia.org/wiki/GNU_C_Library)
* [opcode](https://en.wikipedia.org/wiki/Opcode)
* [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format)
* [GNU linker](https://en.wikipedia.org/wiki/GNU_linker)
* [My posts about assembly programming for x86_64](https://0xax.github.io/categories/assembler/)
* [readelf](https://sourceware.org/binutils/docs/binutils/readelf.html)
