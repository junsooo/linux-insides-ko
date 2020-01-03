리눅스 커널의 여러 데이터 구조
================================================================================

리눅스 커널에서의 비트 배열과 비트 연산
--------------------------------------------------------------------------------

서로 다른 [linked](https://en.wikipedia.org/wiki/Linked_data_structure) 와 [tree](https://en.wikipedia.org/wiki/Tree_%28data_structure%29) 기반의 데이터 구조 외에도 Linux 커널은 [bit arrays](https://en.wikipedia.org/wiki/Bit_array) 또는 `bitmap`에 [API](https://en.wikipedia.org/wiki/Application_programming_interface)를 제공합니다. 비트 배열은 Linux 커널에서 많이 사용되며 다음 소스 코드 파일에는 이러한 구조로 작업하기 위한 공통 `API`가 포함됩니다.

* [lib/bitmap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/bitmap.c)
* [include/linux/bitmap.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/bitmap.h)


이 두 파일 외에도 특정 아키텍처에 최적화 된 비트 연산을 제공하는 아키텍처 별 헤더 파일도 있습니다. 우리는 [x86_64](https://en.wikipedia.org/wiki/X86-64) 아키텍처를 고려하므로 우리의 경우 다음과 같습니다.

* [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h)

헤더 파일. 위에서 쓴 것처럼 `bitmap`은 Linux 커널에서 많이 사용됩니다. 예를 들어, `bit array`는 [hot-plug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt) cpu를 지원하는 시스템을 위한 온라인/오프라인 프로세서 세트를 저장하는 데 사용됩니다 (자세한 내용은 [cpumasks](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html) 부분에서 읽을 수 있음). 또 리눅스 커널 등을 초기화하는 동안 할당 된 [irqs](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 세트를 저장하는데에도 사용됩니다.

따라서 이 파트의 주요 목표는 Linux 커널에서 `bit array(비트 배열)`가 어떻게 구현되는지 확인하는 것입니다. 시작합시다.

비트 배열의 선언
================================================================================

비트 맵 조작에 대한 `API`를 살펴보기 전에 Linux 커널에서 이를 선언하는 방법을 알아야합니다. 자신의 비트 배열을 선언하는 두 가지의 일반적인 방법이 있습니다. 비트 배열을 선언하는 첫 번째 간단한 방법은 `unsigned long` 배열입니다. 예를 들면 다음과 같습니다. :

```C
unsigned long my_bitmap[8]
```

두 번째 방법은 [include/linux/types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/types.h) 헤더 파일에 정의 된 `DECLARE_BITMAP` 매크로를 사용하는 것입니다. :

```C
#define DECLARE_BITMAP(name,bits) \
    unsigned long name[BITS_TO_LONGS(bits)]
```

`DECLARE_BITMAP` 매크로는 두 가지 매개 변수를 사용합니다. :

* `name` - 비트맵의 이름;
* `bits` - 비트맵의 비트 수;

그리고 `BITS_TO_LONGS(bits)` 요소를 사용하여 `unsigned long` 배열의 정의로 확장합니다. 여기서 `BITS_TO_LONGS` 매크로는 주어진 비트 수를 `longs` 수로 변환합니다. 즉, `bits` 안의 `8` 바이트 요소의 수를 게산합니다.:

```C
#define BITS_PER_BYTE           8
#define DIV_ROUND_UP(n,d) (((n) + (d) - 1) / (d))
#define BITS_TO_LONGS(nr)       DIV_ROUND_UP(nr, BITS_PER_BYTE * sizeof(long))
```

따라서, 예를들어 `DECLARE_BITMAP(my_bitmap, 64)` 는 다음을 생성할 것입니다. :

```python
>>> (((64) + (64) - 1) / (64))
1
```

그리고 :

```C
unsigned long my_bitmap[1];
```

비트 배열을 선언 한 후에는 이제 사용할 수 있습니다.

아키텍처 별 비트 연산
================================================================================

우리는 이미 비트 배열의 조작을위한 [API](https://en.wikipedia.org/wiki/Application_programming_interface)를 제공하는 몇 가지 소스 코드와 헤더 파일을 보았습니다. 비트 배열의 가장 중요하고 널리 사용되는 API는 아키텍처에 따라 다르며 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h) 헤더 파일에서 이미 알고 있습니다.

우선 가장 중요한 두 가지 함수를 살펴 보겠습니다. :

* `set_bit`;
* `clear_bit`.

이 함수들이 무엇을 하는지는 설명 할 필요없다고 생각합니다. 그것은 이미 함수들의 이름에서부터 분명합니다. 함수들의 구현을 살펴 봅시다. [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h) 헤더 파일을 살펴보면 이러한 각 함수들은 [atomic](https://en.wikipedia.org/wiki/Linearizability)과 not의 두 가지 변형으로 나타납니다. 이러한 함수의 구현을 시작하기 전에 먼저 `atomic(원자적)` 연산 에 대해 어느정도 알아야합니다.

간단히 말해서 원자적 연산은 두 개 이상의 연산이 동일한 데이터에 대해 동시에 수행되지 않도록 하는 것입니다. `x86` 아키텍처는 원자 명령어 세트를 제공합니다. 예를 들어 [xchg](http://x86.renejeschke.de/html/file_module_x86_id_328.html) 명령어, [cmpxchg](http://x86.renejeschke.de/html/file_module_x86_id_41.html) 명령어 등 원자 명령어 외에 일부는 원자 명령어가 [lock](http://x86.renejeschke.de/html/file_module_x86_id_159.html) 명령어의 도움으로 원자적으로 만들 수 있습니다. 이제 원자 연산에 대해 아는 것이 충분하므로 `set_bit`와 `clear_bit` 함수의 구현을 고려할 수 있습니다.

우선, 이 함수의 `비원자적` 변형을 고려해 봅시다. 비원자 `set_bit` 와 `clear_bit`의 이름은 두개의 밑줄로 시작합니다. 우리가 이미 알고 있듯이 이러한 모든 함수는 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h) 헤더 파일에 정의되어 있으며 첫 번째 함수는 `__set_bit`입니다. :

```C
static inline void __set_bit(long nr, volatile unsigned long *addr)
{
	asm volatile("bts %1,%0" : ADDR : "Ir" (nr) : "memory");
}
```

보시다시피 두개의 매개변수를 가집니다. :

* `nr` - 비트 배열의 비트 수;
* `addr` -우리가 비트를 설정할 필요가 있는 비트 배열의 주소.

`addr` 파라미터는 `volatile` 키워드로 정의되어 주어진 주소에 의해 값이 변경 될 수 있음을 컴파일러에 알려줍니다. `__set_bit`의 구현은 매우 쉽습니다. 보시다시피 한 줄의 [inline assembler](https://en.wikipedia.org/wiki/Inline_assembler) 코드만 포함되어 있습니다. 우리의 경우 비트 배열에서 첫 번째 피연산자 (우리의 경우 `nr`)로 지정된 비트를 선택하고 선택된 비트의 값을 [CF](https://en.wikipedia.org/wiki/FLAGS_register) 플래그 레지스터에 저장하는 [bts](http://x86.renejeschke.de/html/file_module_x86_id_25.html)  명령어를 사용합니다. 이 비트를 설정하세요.

`nr`의 사용법을 볼 수도 있지만 여기에 `addr`이 있습니다. 당신은 이미 그 비밀이 `ADDR`에 있다고 추측 할 것입니다. `ADDR`은 동일한 헤더 코드 파일에 정의 된 매크로이며 주어진 주소와 `+m` 제약 조건을 포함하는 문자열로 확장됩니다. :

```C
#define ADDR				BITOP_ADDR(addr)
#define BITOP_ADDR(x) "+m" (* (volatile long * ) (x))
```

`+m` 외에도 `__set_bit` 함수에서 다른 제약 조건을 볼 수 있습니다. 그것들을 살펴보고 그들이 무엇을 의미하는지 이해해봅시다.

* `+m` - 메모리 피연산자를 나타냅니다.`+`는 주어진 피연산자가 입력 및 출력 피연산자임을 나타냅니다.
* `I` - 정수 상수를 나타냅니다.
* `r` - 레지스터 피연산자를 나타냅니다.

이 제약 조건 외에도 컴파일러가 이 코드가 메모리의 값을 변경한다는 것을 알려주는 `memory` 키워드도 볼 수 있습니다. 이게 끝입니다. 이제 동일한 기능이지만 `원자적` 변형을 살펴 보겠습니다. `비원자적` 변형보다 더 복잡해 보입니다. :

```C
static __always_inline void
set_bit(long nr, volatile unsigned long *addr)
{
	if (IS_IMMEDIATE(nr)) {
		asm volatile(LOCK_PREFIX "orb %1,%0"
			: CONST_MASK_ADDR(nr, addr)
			: "iq" ((u8)CONST_MASK(nr))
			: "memory");
	} else {
		asm volatile(LOCK_PREFIX "bts %1,%0"
			: BITOP_ADDR(addr) : "Ir" (nr) : "memory");
	}
}
```

우선 이 함수는 `__set_bit`와 동일한 파라미터 세트를 가지지만 추가로 `__always_inline` 속성으로 나타납니다. `__always_inline`은 [include/linux/compiler-gcc.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/compiler-gcc.h)에 정의 된 매크로입니다. `always_inline` 속성으로 확장합니다 :

```C
#define __always_inline inline __attribute__((always_inline))
```

이는 Linux 커널 이미지의 크기를 줄이기 위해 이 함수가 항상 인라인됨을 의미합니다. 이제 `set_bit` 함수의 구현을 이해하려고 해봅시다. 우선 `set_bit` 함수의 시작 부분에서 주어진 비트 수를 확인합니다. 동일한 [헤더](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h) 파일에 정의 된 `IS_IMMEDIATE` 매크로는 내장 [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection) 함수의 호출로 확장됩니다. :

```C
#define IS_IMMEDIATE(nr)		(__builtin_constant_p(nr))
```

`__builtin_constant_p` 내장 함수는 주어진 매개 변수가 컴파일 타임에 일정하다고 알려진 경우 `1`을 리턴하고 다른 경우 `0 `을 리턴합니다. 주어진 비트 수가 컴파일 타임 상수에 알려진 경우 비트를 설정하기 위해 느린  `bts ` 명령어를 사용할 필요가 없습니다. 주어진 비트와 마스크 된 비트 수를 포함하는 주기 주소에서 바이트에 [비트 단위 또는] 비트를 적용 할 수 있습니다. 높은 비트는 `1`이고 그 외의 비트는 0입니다. 주어진 비트 수가 컴파일 타임에 상수로 알려지지 않은 경우, 우리는 `__set_bit` 함수에서 했던 것과 동일하게 작업합니다. `CONST_MASK_ADDR` 매크로 :

```C
#define CONST_MASK_ADDR(nr, addr)	BITOP_ADDR((void *)(addr) + ((nr)>>3))
```

는 주어진 비트를 포함하는 바이트에 대한 오프셋과 함께 주소로 확장합니다. 예를 들어 우리는 `0x1000` 주소를 가지고 있고 비트 수는 `0x9`입니다. 따라서 `0x9`는 `1 byte + one bit` 이므로 주소는 `addr + 1`입니다 :

```python
>>> hex(0x1000 + (0x9 >> 3))
'0x1001'
```

`CONST_MASK` 매크로는 주어진 비트 수를 바이트로 나타내며, 높은 비트는 `1`이고 그 외의 비트는 `0`입니다. :

```C
#define CONST_MASK(nr)			(1 << ((nr) & 7))
```

```python
>>> bin(1 << (0x9 & 7))
'0b10'
```

결국 우리는 이 값들에 대해 비트 단위 `or`을 적용합니다. 예를 들어 주소가 `0x4097` 이고 `0x9` 비트를 설정해야하는 경우 :

```python
>>> bin(0x4097)
'0b100000010010111'
>>> bin((0x4097 >> 0x9) | (1 << (0x9 & 7)))
'0b100010'
```

`9 번째` 비트가 설정됩니다.

이러한 모든 작업에는 `LOCK_PREFIX` 가 표시되어 있으며 이 작업의 원자성을 보장하는 [lock](http://x86.renejeschke.de/html/file_module_x86_id_159.html) 명령으로 확장됩니다.

이미 알고 있듯이, 리눅스 커널은 `set_bit` 과 `__set_bit` 연산 외에도 원자 와 비원자 컨텍스트에서 비트를 지우는 두 가지 역함수를 제공합니다. 그것들은 `clear_bit`와 `__clear_bit` 입니다. 이 두 함수는 동일한 [헤더 파일](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h)에 정의되어 있으며 동일한 매개변수 세트를 사용합니다. 그러나 매개변수만 비슷한 것은 아닙니다. 일반적으로 이러한 함수는 `set_bit`, `__set_bit` 와 매우 유사합니다. 비원자적인 `__clear_bit` 함수의 구현을 살펴 봅시다. :

```C
static inline void __clear_bit(long nr, volatile unsigned long *addr)
{
	asm volatile("btr %1,%0" : ADDR : "Ir" (nr));
}
```

그렇습니다. 보시다시피, 동일한 매개변수 집합을 가지고 매우 유사한 인라인 어셈블러 블록을 포함합니다. `bts` 대신 [btr](http://x86.renejeschke.de/html/file_module_x86_id_24.html) 명령만 사용합니다. 함수 이름의 형태를 이해할 수 있으므로 주어진 주소로 주어진 비트를 지웁니다. `btr` 명령어는 `bts`와 같은 역할을 합니다. 이 명령어는 또한 첫 번째 피연산자에 지정된 주어진 비트를 선택하고, 그 값을 `CF` 플래그 레지스터에 저장하고 이 비트를 두 번째 피연산자로 지정된 주어진 비트 배열에서 지웁니다.

`__clear_bit` 의 원자 변형은 `clear_bit`입니다. :

```C
static __always_inline void
clear_bit(long nr, volatile unsigned long *addr)
{
	if (IS_IMMEDIATE(nr)) {
		asm volatile(LOCK_PREFIX "andb %1,%0"
			: CONST_MASK_ADDR(nr, addr)
			: "iq" ((u8)~CONST_MASK(nr)));
	} else {
		asm volatile(LOCK_PREFIX "btr %1,%0"
			: BITOP_ADDR(addr)
			: "Ir" (nr));
	}
}
```

보시다시피 `set_bit`와 매우 유사하며 두 가지 차이점만 있습니다. 첫 번째 차이점은 `set_bit`가 `bts` 명령어를 사용하여 비트를 설정할 때 `btr`명령어를 사용하여 비트를 지우는 것입니다. 두 번째 차이점은 `set_bit`가 `or` 명령어를 사용할 때 negated 마스크와 `and` 명령어를 사용하여 주어진 바이트에서 비트를 지우는 것입니다.

이제 끝입니다. 이제 비트 배열에서 비트를 설정하고 지울 수 있으며 비트 마스크에서 다른 작업을 수행 할 수 있습니다.

비트 어레이에서 가장 널리 사용되는 작업은 Linux 커널에서 비트 어레이의 비트가 설정되고 지워집니다. 그러나이 작업 외에도 비트 배열에서 추가 작업을 수행하는 것이 유용합니다. Linux 커널에서 널리 사용되는 또 다른 작업은 주어진 비트 세트가 비트 배열인지 아는 것입니다. 우리는 `test_bit` 매크로의 도움으로 이것을 달성 할 수 있습니다. 이 매크로는 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h) 헤더 파일에 정의되어 있습니다. 비트 번호에 따라 `constant_test_bit` 또는 `variable_test_bit`의 호출로 확장됩니다. :

```C
#define test_bit(nr, addr)			\
	(__builtin_constant_p((nr))                 \
	 ? constant_test_bit((nr), (addr))	        \
	 : variable_test_bit((nr), (addr)))
```

따라서 컴파일 타임 상수에서 `nr`이 알려진 경우, `test_bit`는 `constant_test_bit` 함수 또는 `variable_test_bit`의 호출로 확장됩니다. 이제 이러한 함수의 구현을 살펴 보겠습니다. `variable_test_bit`부터 시작하겠습니다 :

```C
static inline int variable_test_bit(long nr, volatile const unsigned long *addr)
{
	int oldbit;

	asm volatile("bt %2,%1\n\t"
		     "sbb %0,%0"
		     : "=r" (oldbit)
		     : "m" (* (unsigned long * )addr), "Ir" (nr));

	return oldbit;
}
```

`variable_test_bit` 함수는 `set_bit` 와 다른 함수가 가지는 것과 비슷한 매개변수 세트를 취합니다. 또한 [bt](http://x86.renejeschke.de/html/file_module_x86_id_22.html) 와 [sbb](http://x86.renejeschke.de/html/file_module_x86_id_286.html) 명령을 실행하는 인라인 어셈블리 코드를 볼 수 있습니다. `bt` 또는 `bit test` 명령은 두 번째 피연산자로 지정된 비트 배열에서 첫 번째 피연산자로 지정된 주어진 비트를 선택하고 그 값을 플래그 레지스터의 [CF](https://en.wikipedia.org) 비트에 저장합니다. 두 번째 `sbb` 명령어는 두 번째 피연산자에서 첫 번째 피연산자를 빼고 `CF`의 값을 뺍니다. 따라서 여기에서 주어진 비트 배열의 값을 플래그 레지스터의 `CF` 비트에 기록하고 `sbb` 명령을 실행하여 `00000000-CF`를 계산하고 그 결과를 `oldbit`에 기록합니다.

`constant_test_bit` 함수는 `set_bit`에서 본 것과 동일합니다 :

```C
static __always_inline int constant_test_bit(long nr, const volatile unsigned long *addr)
{
	return ((1UL << (nr & (BITS_PER_LONG-1))) &
		(addr[nr >> _BITOPS_LONG_SHIFT])) != 0;
}
```

이는 높은 비트가 `1`이고 그 외의 비트가 `0`인 바이트를 생성하고 ( `CONST_MASK`에서 보았듯이 ) 주어진 비트 수를 포함하는 바이트에 비트 단위 [and](https://en.wikipedia.org/wiki/Bitwise_operation#AND)를 적용합니다.

다음으로 널리 사용되는 비트 배열 관련 작업은 비트 배열에서 비트를 변경하는 것입니다. Linux 커널은 이를 위해 두 가지 헬퍼를 제공합니다. :

* `__change_bit`;
* `change_bit`.

이미 짐작할 수 있듯이 이 두 변형은 `set_bit` 와 `__set_bit`와 같이 원자적이고 비원자적입니다. 시작하기 위해, `__change_bit` 함수의 구현을 먼저 봅시다.:

```C
static inline void __change_bit(long nr, volatile unsigned long *addr)
{
    asm volatile("btc %1,%0" : ADDR : "Ir" (nr));
}
```

꽤 쉽지않나요?  `__change_bit`의 구현은 `__set_bit`와 동일하지만 `bts` 명령 대신 [btc](http://x86.renejeschke.de/html/file_module_x86_id_23.html)를 사용하고 있습니다. 이 명령어는 주어진 비트 배열에서 주어진 비트를 선택하고, 그 값을 `CF`에 저장하고 보수 연산을 적용하여 그 값을 변경합니다. 따라서 값이 `1`인 비트는 `0`이 되고 그 반대의 경우도 마찬가지로 진행됩니다.


```python
>>> int(not 1)
0
>>> int(not 0)
1
```

`__change_bit`의 원자 버전은 `change_bit` 함수입니다. :

```C
static inline void change_bit(long nr, volatile unsigned long *addr)
{
	if (IS_IMMEDIATE(nr)) {
		asm volatile(LOCK_PREFIX "xorb %1,%0"
			: CONST_MASK_ADDR(nr, addr)
			: "iq" ((u8)CONST_MASK(nr)));
	} else {
		asm volatile(LOCK_PREFIX "btc %1,%0"
			: BITOP_ADDR(addr)
			: "Ir" (nr));
	}
}
```

`set_bit` 함수와 비슷하지만 두 가지 차이점이 있습니다. 첫 번째 차이점은 `or` 대신 `xor` 연산을 사용한다는 것이고, 두 번째 차이점은 `bts` 대신 `btc`을 사용한다는 것 입니다.

현재로서는 비트 배열을 사용하는 가장 중요한 아키텍처 별 작업을 알고 있습니다. 일반적인 비트 맵 API를 살펴볼 시간입니다.

일반적인 비트 연산
================================================================================

[arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h) 헤더 파일의 아키텍처 별 API 외에 Linux 커널은 비트 배열 조작을 위한 공통 API를 제공합니다. 이 파트의 시작 부분에서 알 수 있듯이 [include/linux/bitmap.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/bitmap.h) 헤더 파일과 * [lib/bitmap.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/bitmap.c) 소스 코드 파일에서 찾을 수 있습니다. 그러나 이 소스 코드 파일들 전에 유용한 매크로 세트를 제공하는 [include/linux/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/bitops.h) 헤더 파일을 살펴 보자. 그들 중 일부를 봅시다.

우선 다음 네 가지 매크로를 살펴 보겠습니다. :

* `for_each_set_bit`
* `for_each_set_bit_from`
* `for_each_clear_bit`
* `for_each_clear_bit_from`

이러한 모든 매크로는 비트 배열의 특정 비트 집합에 대해 반복자를 제공합니다. 첫 번째 매크로는 설정된 비트를 반복하고, 두 번째 매크로도 동일하지만 특정 비트에서 시작합니다. 마지막 두 매크로는 동일하지만 명확한 비트를 반복합니다. `for_each_set_bit` 매크로의 구현을 살펴 봅시다. :

```C
#define for_each_set_bit(bit, addr, size) \
	for ((bit) = find_first_bit((addr), (size));		\
	     (bit) < (size);					\
	     (bit) = find_next_bit((addr), (size), (bit) + 1))
```

보시다시피, 세 개의 매개변수를 취하여 첫 번째 비트 세트에서 루프로 확장되는데, 이는 `find_first_bit` 함수의 결과로 반환되고 주어진 크기보다 작으면 마지막 비트 수로 확장됩니다.

이 4가지 매크로 외에도 [arch/x86/include/asm/bitops.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/bitops.h)는 `64 비트` 또는 `32 비트`값 등의 회전을 위한 API를 제공합니다.

비트 배열로 조작하기위한 API를 제공하는 다음 [헤더](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/bitmap.h) 파일입니다. 예를 들어 두 가지 함수를 제공합니다. :


* `bitmap_zero`;
* `bitmap_fill`.

비트 배열을 지우고 `1`로 채웁니다. `bitmap_zero`함수의 구현을 살펴 봅시다. :

```C
static inline void bitmap_zero(unsigned long *dst, unsigned int nbits)
{
	if (small_const_nbits(nbits))
		*dst = 0UL;
	else {
		unsigned int len = BITS_TO_LONGS(nbits) * sizeof(unsigned long);
		memset(dst, 0, len);
	}
}
```

우선 우리는 `nbits`에 대한 확인을 볼 수 있습니다. `small_const_nbits`는 동일한 헤더 [파일](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/bitmap.h)에 정의 된 매크로이며 다음과 같습니다. :

```C
#define small_const_nbits(nbits) \
	(__builtin_constant_p(nbits) && (nbits) <= BITS_PER_LONG)
```

보시다시피 `nbits`는 컴파일 타임에 상수로 알려져 있으며 `nbits` 값은 `BITS_PER_LONG` 또는 `64`을 넘기지 않습니다. 만약 비트 숫자가 `long`값에서 비트의 크기를 오버플로하지 않으면 우리는 단순히 0으로 설정할 수 있습니다. 그렇지 않은 경우에는 비트 배열을 채우고 [memset](http://man7.org/linux/man-pages/man3/memset.3.html)으로 채우는 데 필요한 `long`값의 수를 계산해야합니다.

`bitmap_fill` 함수의 구현은 주어진 비트 배열을 '0xff'값 또는 '0b11111111'로 채우는 것을 제외하고는 'biramp_zero'함수의 구현과 유사합니다.

```C
static inline void bitmap_fill(unsigned long *dst, unsigned int nbits)
{
	unsigned int nlongs = BITS_TO_LONGS(nbits);
	if (!small_const_nbits(nbits)) {
		unsigned int len = (nlongs - 1) * sizeof(unsigned long);
		memset(dst, 0xff,  len);
	}
	dst[nlongs - 1] = BITMAP_LAST_WORD_MASK(nbits);
}
```

`bitmap_fill`와 `bitmap_zero`함수 외에도 [include/linux/bitmap.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/bitmap.h) 헤더 파일은 `bitmap_zero`와 비슷한 `bitmap_copy`를 제공하지만 [memset](http://man7.org/linux/man-pages/man3/memset.3.html) 대신 [memcpy](http://man7.org/linux/man-pages/man3/memcpy.3.html)를 사용합니다. 또한 `bitmap_and`, `bitmap_or`, `bitamp_xor` 등과 같은 비트 배열에 대한 비트 단위 연산을 제공합니다. 이 부분을 모두 이해하면 이러한 함수의 구현을 이해하기 쉽기 때문에 이러한 함수의 구현을 고려하지 않을 것입니다. 어쨌든 이 함수들이 어떻게 구현되었는지에 관심이 있다면 [include/linux/bitmap.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/bitmap.h) 헤더 파일을 열고 공부를 시작할 수 있습니다.

이제 끝입니다.

Links
================================================================================

* [bitmap](https://en.wikipedia.org/wiki/Bit_array)
* [linked data structures](https://en.wikipedia.org/wiki/Linked_data_structure)
* [tree data structures](https://en.wikipedia.org/wiki/Tree_%28data_structure%29)
* [hot-plug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
* [cpumasks](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [IRQs](https://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [API](https://en.wikipedia.org/wiki/Application_programming_interface)
* [atomic operations](https://en.wikipedia.org/wiki/Linearizability)
* [xchg instruction](http://x86.renejeschke.de/html/file_module_x86_id_328.html)
* [cmpxchg instruction](http://x86.renejeschke.de/html/file_module_x86_id_41.html)
* [lock instruction](http://x86.renejeschke.de/html/file_module_x86_id_159.html)
* [bts instruction](http://x86.renejeschke.de/html/file_module_x86_id_25.html)
* [btr instruction](http://x86.renejeschke.de/html/file_module_x86_id_24.html)
* [bt instruction](http://x86.renejeschke.de/html/file_module_x86_id_22.html)
* [sbb instruction](http://x86.renejeschke.de/html/file_module_x86_id_286.html)
* [btc instruction](http://x86.renejeschke.de/html/file_module_x86_id_23.html)
* [man memcpy](http://man7.org/linux/man-pages/man3/memcpy.3.html)
* [man memset](http://man7.org/linux/man-pages/man3/memset.3.html)
* [CF](https://en.wikipedia.org/wiki/FLAGS_register)
* [inline assembler](https://en.wikipedia.org/wiki/Inline_assembler)
* [gcc](https://en.wikipedia.org/wiki/GNU_Compiler_Collection)
