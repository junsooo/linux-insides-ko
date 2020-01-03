커널 부팅 과정. Part 2
================================================================================

커널 구성의 첫 단계
--------------------------------------------------------------------------------

우리는 [지난 시간](linux-bootstrap-1.md)에 리눅스 커널의 내부로 진입을 시작했고 커널 구성 코드의 일부를 살펴봤습니다. 우리는 [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)의 `main`함수를 처음으로 호출하는 데에서 멈췄었습니다. (C로 짜여진 첫 번째 함수죠)

이번 시간엔, 커널 구성 코드를 더 파헤쳐보고 아래 것들에 대해 알아봅시다.
* `protected mode`란 무엇인가
* `protected mode`로의 이행
* 힙과 콘솔의 초기화
* 메모리 탐지, CPU 유효성 검사 그리고 키보드 초기화
* 그리고 많고 많은 것들

그럼 가시죠.

보호 모드(Protected mode)
--------------------------------------------------------------------------------

네이티브 Intel64 [Long Mode](http://en.wikipedia.org/wiki/Long_mode)로 가기 전에, 커널은 CPU를 보호 모드로 전환해야만 합니다.

[보호 모드](https://ko.wikipedia.org/wiki/%EB%B3%B4%ED%98%B8_%EB%AA%A8%EB%93%9C)([protected mode](https://en.wikipedia.org/wiki/Protected_mode))가 뭘까요? 보호 모드는 1982년에 x86 아키텍처에 처음 추가되었고, [80286](http://en.wikipedia.org/wiki/Intel_80286) 프로세서부터 Intel 64와 long mode가 등장하기 전까지 인텔 프로세들의 메인 모드였습니다.

[리얼 모드](https://ko.wikipedia.org/wiki/%EB%A6%AC%EC%96%BC_%EB%AA%A8%EB%93%9C)([Real mode](http://wiki.osdev.org/Real_Mode))에서 옮겨온 가장 큰 이유는 RAM에 굉장히 한정적인 접근만 가능했기 때문이었습니다. 이전 시간에서 얘기했던 것처럼, 오직 2<sup>20</sup> 바이트 또는 1 메가바이트만 가능했고, 어떤 때는 리얼 모드에서 오로지 RAM의 640 킬로바이트만 사용가능했습니다.

보호 모드는 많은 변화를 가져왔지만 그 중 메인은 메모리 관리의 변화였습니다. 20비트 주소 버스는 32비트 주소 버스로 교체되었습니다. 이는 실제 모드에서 1메가바이트만 접근가능했던 것과는 달리 4기가바이트에 달하는 메모리에 접근 가능하게 해주었습니다. 또한, [페이징]( [https://ko.wikipedia.org/wiki/%ED%8E%98%EC%9D%B4%EC%A7%95](https://ko.wikipedia.org/wiki/페이징) )([paging](http://en.wikipedia.org/wiki/Paging)) 지원이 추가되어, 페이징으로 다음 구역을 읽을 수 있게 되었습니다.

보호 모드의 메모리 관리는 거의 독립적인 두 부분으로 나뉩니다.

* 분할 (세그먼트)
* 페이징

여기서는 분할에 대해서만 다루고, 페이징은 다음 시간에 다루도록 하겠습니다.

이전 시간에서 읽었듯, 리얼 모드에서 주소는 두 부분으로 구성됩니다.

* 세그먼트의 베이스 주소
* 세그먼트 베이스로 부터의 오프셋

그리고 이 두 부분을 알고 있다면 다음과 같이 물리적 주소를 알 수 있습니다.

```
PhysicalAddress = Segment Selector * 16 + Offset
```

보호 모드에서 메모리 세그먼트 방식은 완전히 재구성되었습니다. 64 킬로바이트의 고정 크기 세그먼트는 사라졌습니다. 대신에, 각 세그먼트의 크기와 위치는 _세그먼트 디스크립터(Segment Descriptor)_라는 자료구조에 기록되었습니다.  세그먼트 디스크립터는 `Global Descriptor Table` (GDT)라고 불리는 자료구조에 저장되었습니다.

GDT는 메모리 내에 존재하는 구조체입니다. GDT의 자리는 메모리 내에 고정되어 있지 않아서 그 주소가 특별한 `GDTR` 레지스터에 저장됩니다. 나중에 리눅스 커널 코드에서 어떻게 GDT가 로드되는지 살펴볼 겁니다. `lgdt`명령어가 베이스 주소와 GDT의 제한(크기)를 `GDTR` 레지스터에 가져오는 곳에 GDT를 메모리로 로드하기 위한 동작이 다음과 같이 있을 것입니다.

```assembly
lgdt gdt
```

 `GDTR` 는 48비트 레지스터이며 두 부분으로 이루어저 있습니다.

 * GDT의 크기 (16-bit) ;
 * GDT의 주소 (32-bit) .

위에서 언급한 것처럼, GDT는 메모리 세그먼트를 기록하는 `세그먼트 디스크립터`를 포함하고 있습니다.  각각의 디스크립터는 64비트의 크기를 갖고 있습니다. 세그먼트 디스크립터의 일반적인 구성은 다음과 같습니다:

```
 63         56         51   48    45           39        32 
------------------------------------------------------------
|             | |B| |A|       | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMIT |P|DPL|S|  TYPE | BASE 23:16 |
|             | |D| |L| 19:16 | |   | |1|C|R|A|            |
------------------------------------------------------------

 31                         16 15                         0 
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |       LIMIT 15:0           |
|                             |                            |
------------------------------------------------------------
```

리얼 모드에서 넘어오니 좀 무서워보일 수 있지만 걱정 마세요, 사실 쉽습니다. 예를 들어 LIMIT 15:0 는 Limit의 0-15번째 비트가 디스크립터의 시작 부분에 위치한다는 것을 의미합니다. 나머지 부분들은 LIMIT 19:16에 들어있고, 이는 디스크립터의 48-51번째 비트에 위치해 있습니다. 그래서 Limit의 크기는 0-19 즉, 20 비트가 되는 것입니다. 이에 대해 조금 더 자세히 살펴봅시다.

1. Limit[20 비트] 는 비트 0-15 와 48-51로 나누어집니다. 이는 `length_of_segment - 1` 값을 나타내고, `G`(Granularity) 비트에 의해 좌우됩니다.

  * 만약 `G` (비트 55) 가 0이고 segment limit도 0이면, 세그먼트의 크기는 1 Byte입니다.
  * 만약 `G` 가 1이고 segment limit가 0이면, 세그먼트의 크기는 4096 Byte입니다.
  * 만약 `G` 가 0이고 segment limit가 0xfffff이면, 세그먼트의 크기는 1 MByte입니다.
  * 만약 `G` 가 1이고 segment limit가 0xfffff이면, 세그먼트의 크기는 4 GByte입니다.

 이말인 즉슨,
  * 만약 G가 0이면, Limit 은 1 바이트로 해석되고, 세그먼트의 최대 크기는 1 메가바이트가 됩니다.
  * 만약 G가 1이면, Limit 은 4096 Bytes = 4 KBytes = 1 Page 로 해석되고 세그먼트의 최대 크기는 4 기가바이트가 됩니다. 사실, G가 1일 때, Limit의 값은 왼쪽으로 12비트 이동(shift)됩니다. 그러므로 20 bit + 12 bit = 32 bit이고 2<sup>32</sup> = 4 Gigabytes입니다.

2. Base[32-bits] 는 16-31번째, 32-39번째, 그리고 56-63번째 비트들 사이에서 쪼개집니다. 이것은 세그먼트의 시작 위치의 물리적 주소를 나타냅니다.

3. Type/Attribute[5-bits] 는 40-44번째 비트로 나타내어집니다. 이것은 세그먼트의 종류와 어떻게 접근 가능한지를 나타냅니다.
  * 44번째 비트의 `S` 플래그는 디스크립터의 종류를 특정합니다. 만약 `S`가 0이면 이 세그먼트는 시스템 세그먼트이고, 만약 `S`가 1이면 이것은 코드 혹은 데이터 세그먼트입니다. (스택 세그먼트는 데이터 세그먼트이고, 읽기/쓰기 세그먼트여야 합니다.).

특정 세그먼트가 코드 세그먼트인지 데이터 세그먼트인지 판별하기 위해서는, 그것의 Ex 속성(43번 비트)을 확인하면 됩니다 (위 다이어그램에는 0으로 표기되어있음). 만약 0이면, 그 세그먼트는 데이터 세그먼트이고, 그렇지 않다면 그건 코드 세그먼트입니다.

세그먼트는 다양한 타입이 될 수 있는데 이는 다음과 같습니다.

```
--------------------------------------------------------------------------------------
|           Type Field        | Descriptor Type | Description                        |
|-----------------------------|-----------------|------------------------------------|
| Decimal                     |                 |                                    |
|             0    E    W   A |                 |                                    |
| 0           0    0    0   0 | Data            | Read-Only                          |
| 1           0    0    0   1 | Data            | Read-Only, accessed                |
| 2           0    0    1   0 | Data            | Read/Write                         |
| 3           0    0    1   1 | Data            | Read/Write, accessed               |
| 4           0    1    0   0 | Data            | Read-Only, expand-down             |
| 5           0    1    0   1 | Data            | Read-Only, expand-down, accessed   |
| 6           0    1    1   0 | Data            | Read/Write, expand-down            |
| 7           0    1    1   1 | Data            | Read/Write, expand-down, accessed  |
|                  C    R   A |                 |                                    |
| 8           1    0    0   0 | Code            | Execute-Only                       |
| 9           1    0    0   1 | Code            | Execute-Only, accessed             |
| 10          1    0    1   0 | Code            | Execute/Read                       |
| 11          1    0    1   1 | Code            | Execute/Read, accessed             |
| 12          1    1    0   0 | Code            | Execute-Only, conforming           |
| 14          1    1    0   1 | Code            | Execute-Only, conforming, accessed |
| 13          1    1    1   0 | Code            | Execute/Read, conforming           |
| 15          1    1    1   1 | Code            | Execute/Read, conforming, accessed |
--------------------------------------------------------------------------------------
```

위 표에서 볼 수 있듯, 첫 번째 비트(43번 비트)가  `0`이면 _데이터_ 세그먼트이고 `1`이면 _코드_ 세그먼트입니다. 다음 세 비트 (40, 41, 42)는 `EWA`(*E*xpansion *W*ritable *A*ccessible) 이거나 `CRA`(*C*onforming *R*eadable *A*ccessible)입니다.
  * 만약 E(42번 비트)가 0이면, 위로 확장하고, 1이면 아래로 확장합니다. [여기](http://www.sudleyplace.com/dpmione/expanddown.html)서 더 알아보세요.
  * 만약 (데이터 세그먼트인 경우에) W(41번 비트)가 1이라면, 쓰기 엑세스가 허용됩니다. 반대로 0일 경우엔, 읽기 전용이 됩니다.  데이터 세그먼트에서는 항상 읽기 액세스가 허용된다는 점에 유의하세요. 
  * A(40번 비트)는 프로세서에서 세그먼트에 액세스할 수 있는지 여부를 제어합니다. 
  * C(43번 비트)는 호환 비트입니다(코드 선택기의 경우). C가 1이면 낮은 수준 권한(예: 사용자)에서도 세그먼트 코드를 실행할 수 있습니다. C가 0이면 동등한 권한 레벨에서만 실행할 수 있습니다. 
  * R(41번 비트)은 코드 세그먼트에 대한 읽기 액세스를 제어합니다. 이 값이 1이면 세그먼트를 읽을 수 있습니다. 코드 세그먼트에 대해서는 쓰기 액세스 권한이 부여되지 않습니다. 

4. DPL[2비트] (Descriptor 권한 수준)은 45-46번 비트로 구성됩니다. 이것은 세그먼트의 권한 수준을 정의합니다. 0-3의 값을 가질 수 있는데 여기서 0이 가장 높은 권한 수준입니다. 

5. P 플래그(47번 비트)는 세그먼트가 메모리에 있는지의 여부를 나타냅니다. P가 0이면 세그먼트가 _invalid_로 표시되고 프로세서가 이 세그먼트에서 읽기를 거부합니다. 

6. AVL 플래그(52번 비트) - 사용도 가능하고 이를 위해 자리도 예약된 비트이지만 Linux에서는 무시됩니다. 

7.  L 플래그(53번 비트)는 코드 세그먼트에 네이티브 64비트 코드가 포함되어 있는지 여부를 나타냅니다. 플래그가 설정되어 있는 경우, 해당 코드 세그먼트는 64비트 모드로 실행됩니다. 

8.  D/B 플래그(54번 비트)(기본/빅 플래그)는 피연산자 크기(예: 16/32비트)를 나타냅니다. 설정된 경우 피연산자 크기는 32비트입니다. 그렇지 않으면 16비트입니다. 

세그먼트 레지스터에는 실제 모드와 같은 세그먼트 셀렉터가 포함되어 있습니다. 그러나 보호 모드에서는 세그먼트 셀렉터가 다르게 처리됩니다. 각 세그먼트 디스크립터에는 다음과 같이 16비트 구조의 연결된 세그먼트 셀렉터가 있습니다.

```
 15             3 2  1     0
-----------------------------
|      Index     | TI | RPL |
-----------------------------
```

각각의 구역은...
* **Index**는 GDT 내부에 있는 디스크립터의 인덱스 번호를 저장합니다.
* **TI**(Table Indicator) 는 디스크립터를 찾을 위치를 나타냅니다. 0이면 GDT(Global Descriptor Table)에서 디스크립터를 찾습니다. 1이면 LDT(Local Descriptor Table)에서 찾습니다.
* 그리고 **RPL**에는 요청자의 권한 수준이 들어있습니다.

모든 세그먼트 레지스터에는 각각 보이는 부분과 숨겨진 부분이 존재합니다.
* Visible - 세그먼트 셀렉터가 여기 저장됩니다.
* Hidden -  base, limit, attributes, 그리고 flags와 같은 정보를 담고 있는 세그먼트 디스크립터가 여기 저장됩니다. 

보호 모드에서 물리적 주소를 얻으려면 다음 단계들이 필요합니다.

* 세그먼트 셀렉터가 세그먼트 레지스터 중 하나에 로드되어야 합니다.
* CPU는 셀렉터의 `GDT 주소 + 인덱스` 오프셋에서 세그먼트 디스크립터를 찾은 다음 세그먼트 레지스터의 *숨겨진* 부분에 디스크립터를 로드합니다.
* 페이징이 비활성화된 경우, 세그먼트의 선형 주소 또는 물리적 주소는 다음 수식을 따릅니다: 기본 주소(이전 단계에서 얻은 디스크립터에서 찾을 수 있음) + 오프셋

도식으로 나타내면 다음과 같습니다.

![linear address](images/linear_address.png)

리얼 모드에서 보호 모드로 전환하기 위한 알고리즘은 다음과 같습니다.

* 인터럽트 비활성화
* `lgdt` 지침에 따라 GDT를 설명(describe)하고 로드
* CR0 (Control Register 0)에서 PE (Protection Enable) 비트 설정
* 보호 모드 코드로 점프

다음 부분에서 리눅스 커널에서 보호 모드로 완전히 전환되는 것을 볼 수 있지만, 보호 모드로 이동하기 전에 좀 더 준비가 필요합니다.

[arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)를 보면, 키보드 초기화, 힙 초기화 등을 수행하는 루틴을 볼 수 있습니다. 자세히 살펴봅시다.

부팅 매개 변수를 "zeropage"에 복사
--------------------------------------------------------------------------------

"main.c"의`main` 루틴부터 시작합시다. `main`에서 가장 먼저 호출되는 함수는 [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)입니다. 이 함수는 커널 설정 헤더를 [arch/x86/include/uapi/asm/bootparam.h](https://gitub.com/torvalds/linux/v4.16/archclude/uapi/asm/param.h) 헤더 파일에 정의된 해당 필드에 복사합니다.

boot_params 구조에는 'struct setup_header hdr' 필드가 포함되어 있습니다. 이 구조는 [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)에 정의된 것과 동일한 필드를 포함하며 부트 로더에 의해, 그리고 커널 컴파일/빌드 시간에 채워집니다. copy_boot_params는 다음 두 가지 작업을 수행합니다.

1. [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L280)의 `hdr`을 `boot_params` 구조체의 `setup_header` 필드에 복사합니다.
2. 커널이 이전 명령 행 프로토콜(command line protocol)로 로드 된 경우 커널 명령 행에 대한 포인터를 업데이트합니다.

copy_boot_params는 `memcpy` 함수([copy.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S)에 정의됨)를 사용하여 `hdr`를 복사합니다. 한 번 자세히 살펴봅시다.

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```

C 코드로 옮겨온지도 얼마 안되었는데 다시 어셈블리어군요 :) 가장 먼저, 여기서 `memcpy`와 다른 루틴들이 정의되어있는 것을 확인할 수 있습니다. `GLOBAL` 그리고 `ENDPROC`라는 두개의 메크로로 시작하고 끝납니다. `GLOBAL`은 [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/linkage.h) 에 기술되어 있고, 여기엔 `globl` 명령어와 라벨도 정의되어 있습니다. `ENDPROC`는 [include/linux/linkage.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/linkage.h)에 기술되어 있으며, `name`이라는 기호(심볼)을 함수 이름으로 표시하고 `name` 기호의 사이즈로 끝납니다.

`memcpy`의 구현은 간단합니다. 가장 먼저, `si` 및`di` 레지스터의 값이 `memcpy` 동안 변경되므로 값을 보존하기 위해 스택으로 푸시합니다. `arch/x86/Makefile`의`REALMODE_CFLAGS`에서 알 수 있듯이 커널 빌드 시스템은 GCC의`-mregparm = 3` 옵션을 사용하므로 함수는  `ax`,`dx` 및 `cx` 레지스터에서 처음의 세 매개변수를 가져옵니다. `memcpy`의 호출은 다음과 같습니다.

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```

즉,
* `ax`는 `boot_params.hdr`의 주소를 가질 것이고
* `dx`는 `hdr`의 주소를 가질 것이고
* `cx` 는`hdr` 의 크기를 바이트 단위로 나타낸 값을 가질 것입니다.

`memcpy`는`boot_params.hdr`의 주소를`di`에 넣고 `cx`를 스택에 저장합니다. 그런 다음 값을 2 번 오른쪽으로 이동하고 (혹은 4로 나누고) `si`의 주소에서`di`의 주소로 4 바이트를 복사합니다. 그 후, 우리는 `hdr`의 크기를 다시 복원하고, 그것을 4 바이트로 정렬하고 (남는 경우) 나머지 바이트를`si`의 주소에서`di`의 주소로 바이트 단위로 복사합니다. 이제`si`와`di`의 값이 스택에서 복원되었고 복사 작업이 완료되었습니다.

콘솔 초기화
--------------------------------------------------------------------------------

`hdr`이`boot_params.hdr`에 복사 된 후 다음 단계는  [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/early_serial_console.c)에 정의 된`console_init` 함수를 호출하여 콘솔을 초기화하는 것입니다.

함수는 명령 행에서 `earlyprintk` 옵션을 찾으려고 시도하고 만약 검색이 성공하면 직렬 포트(시리얼 포트)의 포트 주소와 보드(baud) 속도를 구문 분석(parse)하고 직렬 포트를 초기화합니다. `earlyprintk` 명령 행 옵션의 값은 다음 중 하나입니다.

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

시리얼 포트 초기화 후 다음과 같이 첫 번째 출력을 볼 수 있습니다.

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

`puts`의 정의는 [tty.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c)에 있습니다. 보시다시피`putchar` 함수를 호출하여 루프에서 문자별로 문자를 출력합니다. 'putchar'구현을 살펴 봅시다.

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```

`__attribute__((section(".inittext")))` 는 이 코드가 `.inittext` 섹션에 있음을 의미합니다. 링커 파일 [setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld)에서 이를 찾아볼 수 있습니다.

먼저 'putchar'는`\n` 기호(심볼)이 있는지 확인하고, 발견되면`\r`을 먼저 출력합니다. 그런 다음`0x10` 인터럽트 호출로 BIOS를 호출하여 VGA 화면에 문자를 출력합니다.

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```

여기서 `initregs`는 `biosregs` 구조체를 가져온 다음 `memset` 함수를 사용하여 먼저 biosregs를 0으로 채우고, 그리고 레지스터 값으로 채 웁니다.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```

[memset](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S#L36)의 구현을 살펴 봅시다.

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```

위에서 읽을 수 있듯이,이 함수는 `memcpy` 함수와 동일한 호출 규칙을 사용합니다. 즉, 함수는`ax`,`dx` 및`cx` 레지스터에서 매개 변수를 가져옵니다.

`memset`의 구현은 memcpy의 구현과 유사합니다. 스택에 `di` 레지스터의 값을 저장하고 `biosregs` 구조체의 주소를 저장하고 있는 'ax'의 값을 `di`에 넣습니다. 다음은 `movzbl` 명령으로 `dl`의 값을 `eax` 레지스터의 하위 2 바이트에 복사합니다. `eax`의 남은 2바이트는 0으로 채워집니다.

다음 명령어는 `eax`와`0x01010101`을 곱합니다. 이것이 필요한 이유는 `memset`은 동시에 4 바이트를 복사하기 때문입니다. 가령 예를 들어, 크기가 4 바이트 인 구조체에 memset으로`0x7` 값을 채우려고 한다면, `eax`에 `0x00000007`가 담길 것입니다. 그러므로 `eax`와 `0x01010101`을 곱하면 `0x07070707`을 얻게되고 이제 이 4 바이트를 구조체에 복사 할 수 있습니다. `memset`은 `rep; stosl` 명령어를 사용하여 `eax`를 `es:di`로 복사합니다

`memset` 함수의 나머지 부분은 `memcpy`와 거의 같은 기능을합니다.

`biosregs` 구조체가 `memset`으로 채워진 후에, `bios_putchar`는 문자를 출력하는 [0x10](http://www.ctyme.com/intr/rb-0106.htm) 인터럽트를 호출합니다. 그런 다음 직렬 포트가 초기화되었는지의 여부를 확인하고 [serial_putchar](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c)와 (명령어가 설정이 되어 있는 경우) `inb/outb` 명령어로 문자를 출력합니다.

힙 초기화
--------------------------------------------------------------------------------

스택 및 bss 섹션이 [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S)에서 준비된 후 (이전 [part](linux-bootstrap-1.md) 참조) 커널은 [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) 함수로 [힙](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)을 초기화해야합니다.

가장 먼저, `init_heap`은 커널 설정 헤더의 [`loadflags`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) 구조체에서 [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L24) 플래그를 확인하고 플래그가 설정된 경우 스택의 끝을 계산합니다.

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

다르게 말하자면, `stack_end = esp-STACK_SIZE`입니다.

그 다음은 'heap_end'계산입니다.

```C
     heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

이것은 `heap_end_ptr` 또는 `_end` +`512` (`0x200h`)를 의미합니다. 마지막으로 확인할 것은`heap_end`가 `stack_end`보다 큰지의 여부입니다. 만약 그렇다면, 둘을 같게 하기 위하여 `stack_end`가 `heap_end`에 할당되게 됩니다.

이제 힙이 초기화되었으며 `GET_HEAP` 메소드를 사용하여 힙을 사용할 수 있습니다. 우리는 그것이 왜 사용되는지, 어떻게 사용되고, 어떻게 구현되는지에 대해서는 다음 포스트에서 살펴 볼 것입니다.

CPU 유효성 검사(validation)
--------------------------------------------------------------------------------

다음 단계는 [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpu.c) 소스 코드 파일의 `validate_cpu` 함수를 통한 CPU 유효성 검사입니다.

`validate_cpu`는 [`check_cpu`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpucheck.c) 함수를 호출하고 CPU 수준과 필요한 CPU 수준을 전달해주고 커널이 올바른 CPU 레벨에서 시작되는지 확인합니다. 

```C
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```

`check_cpu` 함수는 x86_64 (64 비트) CPU의 경우 [long mode](http://en.wikipedia.org/wiki/Long_mode)의 존재 여부와 함께 CPU의 플래그를 확인하고 프로세서 공급 업체를 확인하여 가령 AMD일 경우 SSE + SSE2 끄기와 같이 특정 공급 업체를 위한 준비가 되어 있지 않은 경우, 해당 준비를 합니다.

바로 다음 단계에서, 셋업 코드가 CPU가 적합하다는 것을 확인 한 후에`set_bios_mode` 함수를 호출하는 것을 볼 수 있습니다. 보시다시피 이 함수는`x86_64` 모드에서만 구현됩니다.

```C
static void set_bios_mode(void)
{
#ifdef CONFIG_X86_64
	struct biosregs ireg;

	initregs(&ireg);
	ireg.ax = 0xec00;
	ireg.bx = 2;
	intcall(0x15, &ireg, NULL);
#endif
}
```

`set_bios_mode` 함수는 `0x15` BIOS 인터럽트를 실행하여 BIOS에 (`bx == 2` 인 경우) [long mode](https://en.wikipedia.org/wiki/Long_mode) 가 사용될 것임을 알려줍니다.

메모리 탐지
--------------------------------------------------------------------------------

다음 단계는`detect_memory` 함수을 통한 메모리 탐지입니다. 기본적으로 `detect_memory`는 CPU에 사용 가능한 RAM의 맵을 제공합니다. 함수는 메모리 탐지를 위해 `0xe820`,`0xe801` 및`0x88`과 같은 다른 프로그래밍 인터페이스를 사용합니다. 여기서는 **0xE820** 인터페이스의 구현만 살펴볼 겁니다.

[arch / x86 / boot / memory.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/memory.c) 소스 파일에서 `detect_memory_e820` 함수의 구현을 살펴 보겠습니다. 우선,`detect_memory_e820` 함수는 위에서 보았 듯이 `biosregs`구조체를 초기화하고`0xe820` 호출에 대한 특수 값으로 레지스터를 채웁니다.

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```

* `ax` 함수의 번호를 갖고 있습니다. (이 경우엔 0xe820)
* `cx` 는 메모리에 대한 정보를 담을 버퍼의 크기를 갖고 있습니다.
* `edx`는`SMAP` 매직 넘버를 포함해야합니다
* `es : di`는 메모리 데이터를 포함 할 버퍼의 주소를 포함해야합니다
* `ebx`는 0이어야합니다.

다음은 메모리에 관한 데이터가 수집되는 루프입니다. 이는 주소 할당 테이블에서 한 줄을 쓰는`0x15` BIOS 인터럽트에 대한 호출로 시작합니다. 다음 줄을 얻으려면 (루프 안에서 수행되는) 이 인터럽트를 다시 호출해야합니다. 다음 호출 전에 `ebx`는 이전에 반환 된 값을 포함해야합니다.

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

결과적으로 보면, 이 함수는 주소 할당 테이블에서 데이터를 수집하여 이 데이터를 `e820_entry` 배열에 씁니다.

* 메모리 세그먼트의 시작
* 메모리 세그먼트의 크기
* 메모리 세그먼트의 종류 (특정 세그먼트의 사용 가능한지 아니면 이미 예약되었는지의 여부)

다음과 같이, `dmesg` 출력에서 이 결과를 볼 수 있습니다.

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

키보드 초기화
--------------------------------------------------------------------------------

다음 단계는 [`keyboard_init`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) 함수를 호출하여 키보드를 초기화하는 것입니다. 가장 먼저,`keyboard_init`는 `initregs` 함수를 사용하여 레지스터를 초기화합니다. 그런 다음 [0x16](http://www.ctyme.com/intr/rb-1756.htm) 인터럽트를 호출하여 키보드 상태를 쿼리합니다.

```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```

그런 다음 [0x16](http://www.ctyme.com/intr/rb-1757.htm)을 다시 호출하여 반복 속도와 지연을 설정합니다.

```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

쿼리(Querying)
--------------------------------------------------------------------------------

다음 두 단계는 다른 매개 변수에 대한 쿼리입니다. 이러한 쿼리에 대한 자세한 내용은 다루지 않겠지만 나중에 다시 설명하겠습니다. 그럼 이 기능들을 간단히 살펴 봅시다.

첫 번째 단계는 `query_ist` 함수를 호출하여 [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) 정보를 얻는 것입니다. CPU 레벨을 확인하고 올바른 경우 `0x15`를 호출하여 정보를 얻고 결과를 `boot_params`에 저장합니다.

다음으로, [query_apm_bios](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/apm.c#L21) 함수가 BIOS에서 [고급 전원 관리](http://en.wikipedia.org/wiki/Advanced_Power_Management) 정보를 가져옵니다. `query_apm_bios`도 `0x15` BIOS 인터럽트를 호출하지만 추가로 `ah` =`0x53`으로 `APM` 설치를 확인합니다. `0x15`의 실행이 끝나면 `query_apm_bios` 함수는 `PM`서명 ( `0x504d`이어야 함), 캐리 플래그 ( `APM`이 지원되는 경우 0이어야 함) 및 `cx`레지스터의 값( 0x02 인 경우 보호 모드 인터페이스가 지원됨 )을 확인합니다.

다음으로 `0x15`를 다시 호출하지만 `ax = 0x5304`로 `APM` 인터페이스의 연결을 끊고 32 비트 보호 모드 인터페이스를 연결합니다. 마지막에는, BIOS에서 얻은 값으로 boot_params.apm_bios_info를 채웁니다.

`query_apm_bios`는`CONFIG_APM` 또는`CONFIG_APM_MODULE` 컴파일 타임 플래그가 구성 파일에 설정된 경우에만 실행됩니다.

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```

마지막은  BIOS에서 '확장 된 디스크 드라이브'정보를 쿼리하는 [`query_edd`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/edd.c#L122) 함수입니다. `query_edd`가 어떻게 구현되는지 살펴봅시다.

우선, 커널의 명령 행에서 [edd](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst) 옵션을 읽습니다. `off`로 설정되면 `query_edd`가 반환됩니다.

EDD가 활성화되면`query_edd`는 BIOS 지원 하드 디스크를 통해 다음 루프에서 EDD 정보를 쿼리합니다.

```C
for (devno = 0x80; devno < 0x80+EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
    }
```

여기서 `0x80`은 첫 번째 하드 드라이브이고 `EDD_MBR_SIG_MAX` 매크로의 값은 16입니다.  [edd_info](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/edd.h) 구조의 배열로 데이터를 수집합니다. `get_edd_info`는 `0x13` 인터럽트를 `0x41`인 `ah`로 호출하여 EDD가 존재하는지 확인하고, 만약 EDD가 존재하면 `get_edd_info`는 `0x13` 인터럽트를 한 번 더 호출하지만, 이번에는 `0x48`인 `ah`와 EDD 정보가 저장될 버퍼의 주소를 포함하는 `si`로 호출합니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널 내부의 두 번째 부분은 끝입니다. 다음 부분에서는 비디오 모드 설정과 보호 모드로 전환하기 전 준비과정의 나머지 부분들, 그리고 직접 전환하는 과정을 보도록 하겠습니다.

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견을 보내거나 핑 (Ping) 해주십시오.

**영어는 제 모국어가 아닙니다, 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수들을 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한국 번역](https://github.com/junsooo/linux-insides-ko)로 PR을 보내주세요.**

링크들
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [earlyprintk documentation](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/earlyprintk.txt)
* [Kernel Parameters](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst)
* [Serial console](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/serial-console.rst)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [EDD specification](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [TLDP documentation for Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (old)
* [Previous Part](linux-bootstrap-1.md)
