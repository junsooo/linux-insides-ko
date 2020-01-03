CPU 마스크
================================================================================

개요
--------------------------------------------------------------------------------

`Cpumasks`는 시스템에 CPU에 관한 정보를 저장하기 위해 리눅스 커널이 제공하는 특별한 방법입니다. `Cpumasks` 조작을 위한 API가 포함 된 관련 소스 코드 및 헤더 파일들은 다음과 같습니다 :

* [include/linux/cpumask.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cpumask.h)
* [lib/cpumask.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/cpumask.c)
* [kernel/cpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cpu.c)

[include/linux/cpumask.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cpumask.h)의 주석에서 언급 한 바와 같이: Cpumasks는 시스템에 있는 CPU집합을 표시하기에 적합한 비트맵을 제공하며 CPU 번호 당 1 비트 위치입니다. 우리는 이미 [Kernel entry point](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html) 부분에서`boot_cpu_init` 함수에서 cpumask에 대해 약간 살펴 보았었습니다. 이 함수는 첫 번째 부팅 CPU를 온라인, 활성 또는 기타 등등으로 만듭니다.

```C
set_cpu_online(cpu, true);
set_cpu_active(cpu, true);
set_cpu_present(cpu, true);
set_cpu_possible(cpu, true);
```

이러한 함수들의 구현을 고려하기 전에 이러한 마스크를 모두 살펴봅시다.

`cpu_possible`은 시스템 부팅 기간 동안 언제라도 꽂을 수있는 CPU ID 세트입니다. 즉, '가능한 CPU' 마스크에는 시스템에서 사용가능한 최대 CPU 수가 포함됩니다. 그 값은 `CONFIG_NR_CPUS` 커널 설정 옵션을 통해 정적으로 설정되는 `NR_CPUS` 값과 같습니다.

`cpu_present` 마스크는 현재 연결되어있는 CPU를 나타냅니다.

`cpu_online`은 `cpu_present`의 부분 집합을 나타내며 스케줄링에 사용 가능한 CPU를 나타냅니다. 즉, 이 마스크에서 각 비트가 커널에게 알려주는 것은 프로세서가 리눅스 커널에 의해 사용될 수 있음을 나타냅니다.

마지막 마스크는 `cpu_active`입니다. 이 마스크의 각 비트는 리눅스 커널에게 작업이 특정 프로세서로 이동 될 수 있음을 알려줍니다.

이러한 모든 마스크는 `CONFIG_HOTPLUG_CPU` 구성 옵션과 이 옵션이 비활성화 된 경우 `posable == present` 및 `active == online`에 따릅니다. 이러한 함수들의 구현은 모두 매우 유사합니다. 모든 함수는 두 번째 매개 변수를 확인합니다. 만약 `true`이면 `cpumask_set_cpu`를 호출하고 그렇지 않으면 `cpumask_clear_cpu`를 호출합니다.

`cpumask`생성에는 두 가지 방법이 있습니다. 첫번째 방법은 `cpumask_t`를 사용하는 것입니다. 다음과 같이 정의됩니다.

```C
typedef struct cpumask { DECLARE_BITMAP(bits, NR_CPUS); } cpumask_t;
```

이 구조체는 하나의 비트 마스크 `bits` 필드를 포함하는 `cpumask` 구조를 감쌉니다. `DECLARE_BITMAP` 매크로는 두 개의 매개 변수를 받습니다 :

* bitmap name;
* number of bits.

그리고 주어진 이름으로 `unsigned long` 배열을 만듭니다. 이것의 구현은 매우 쉽습니다.

```C
#define DECLARE_BITMAP(name,bits) \
        unsigned long name[BITS_TO_LONGS(bits)]
```

여기서 `BITS_TO_LONGS`는:

```C
#define BITS_TO_LONGS(nr)       DIV_ROUND_UP(nr, BITS_PER_BYTE * sizeof(long))
#define DIV_ROUND_UP(n,d) (((n) + (d) - 1) / (d))
```

우리가 `x86_64` 아키텍처에 초점을 맞추고 있기 때문에, `unsigned long`은 8 바이트 크기이며 배열은 단 하나의 요소만 포함합니다 :

```
(((8) + (8) - 1) / (8)) = 1
```

`NR_CPUS` 매크로는 시스템의 CPU 수를 나타내며 [include/linux/threads.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include / linux / threads.h)에 정의 된 `CONFIG_NR_CPUS` 매크로에 따르고 다음과 같이 생겼습니다.

```C
#ifndef CONFIG_NR_CPUS
        #define CONFIG_NR_CPUS  1
#endif

#define NR_CPUS         CONFIG_NR_CPUS
```

cpumask를 정의하는 두 번째 방법은 `DECLARE_BITMAP` 매크로를 직접 사용하고 주어진 비트 맵을 `struct cpumask *`로 변환하는 `to_cpumask` 매크로를 사용하는 것입니다.

```C
#define to_cpumask(bitmap)                                              \
        ((struct cpumask *)(1 ? (bitmap)                                \
                            : (void *)sizeof(__check_is_bitmap(bitmap))))
```

보면 아시겠지만 여기서 삼항 연산자는 매번 `true`입니다. `__check_is_bitmap` 인라인 함수는 다음과 같이 정의됩니다 :

```C
static inline int __check_is_bitmap(const unsigned long *bitmap)
{
        return 1;
}
```

그리고 매번 `1`을 반환합니다. 이게 여기서 필요한 이유는 딱 하나입니다: 컴파일 할 때 주어진 `bitmap`이 비트 맵인지 확인하거나 다른 말로는 주어진 `bitmap`유형이 `unsigned long *`타입인지 확인합니다. 그래서 우리는`cpu_possible_bits`를 `to_cpumask` 매크로로 전달하여 `unsigned long`의 배열을 `struct cpumask *`로 변환합니다.

cpumask API
--------------------------------------------------------------------------------

여러 방법을 사용하여 cpumask를 정의 할 수 있으므로 리눅스 커널은 cpumask 조작을위한 API를 제공합니다. 위에서 제시한 함수 중 하나를 고려해 봅시다. `set_cpu_online`를 예로 들어봅시다. 이 함수는 두 가지 매개 변수를 사용합니다.

* Number of CPU;
* CPU status;

이 함수의 구현은 다음과 같습니다.

```C
void set_cpu_online(unsigned int cpu, bool online)
{
	if (online) {
		cpumask_set_cpu(cpu, to_cpumask(cpu_online_bits));
		cpumask_set_cpu(cpu, to_cpumask(cpu_active_bits));
	} else {
		cpumask_clear_cpu(cpu, to_cpumask(cpu_online_bits));
	}
}
```

가장 먼저 두 번째 `state` 매개 변수를 확인하고 `cpumask_set_cpu` 또는 `cpumask_clear_cpu`를 호출합니다. 여기서 우리는 `cpumask_set_cpu`에서 두 번째 매개 변수를 `struct cpumask *`로 캐스팅하는 것을 볼 수 있습니다. 우리의 경우는 `cpu_online_bits`이고 이 비트맵은 다음과 같이 정의됩니다:

```C
static DECLARE_BITMAP(cpu_online_bits, CONFIG_NR_CPUS) __read_mostly;
```

`cpumask_set_cpu` 함수는 `set_bit` 함수를 한 번만 호출합니다.

```C
static inline void cpumask_set_cpu(unsigned int cpu, struct cpumask *dstp)
{
        set_bit(cpumask_check(cpu), cpumask_bits(dstp));
}
```

`set_bit` 함수는 두 개의 매개 변수를 취하며 메모리 (두 번째 매개 변수 또는 `cpu_online_bits` 비트 맵)에 주어진 비트 (첫 번째 매개 변수)를 설정합니다. 여기에서 `set_bit`가 호출되기 전에 두 개의 매개 변수가 아래로 넘겨지는 것을 볼 수 있습니다:

* cpumask_check;
* cpumask_bits.

이 두 매크로를 살펴봅시다. 먼저 우리의 경우 `cpumask_check`가 아무 것도 수행하지 않으면 주어진 매개 변수를 반환합니다. 두 번째 `cpumask_bits`는 주어진 `struct cpumask *`구조에서 `bits` 필드를 반환합니다 :

```C
#define cpumask_bits(maskp) ((maskp)->bits)
```

이제 `set_bit`의 구현을 봅시다 :

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

이 기능은 무섭게 보이지만 그렇게 어렵지는 않습니다. 우선 `nr` 또는 비트 수를 GCC 내부`__builtin_constant_p` 함수를 호출하는 `IS_IMMEDIATE` 매크로에 전달합니다.

```C
#define IS_IMMEDIATE(nr)    (__builtin_constant_p(nr))
```

`__builtin_constant_p`는 컴파일 타임에 주어진 매개 변수가 상수인지 확인합니다. 우리의 `cpu`는 컴파일 타임 상수가 아니기 때문에 `else` 절이 실행될 것입니다 :

```C
asm volatile(LOCK_PREFIX "bts %1,%0" : BITOP_ADDR(addr) : "Ir" (nr) : "memory");
```

그것이 어떻게 작동하는지 이해해봅시다.

`LOCK_PREFIX`는 x86 `lock` 명령어입니다. 이 명령은 CPU가 명령이 실행되는 동안 시스템 버스를 점유하도록 지시합니다. 이를 통해 CPU는 메모리 액세스를 동기화하여 여러 프로세서가 (또는 장치 - 예를 들어 DMA 컨트롤러)를 하나의 메모리 셀에 동시에 액세스 할 수 없게 막습니다.

`BITOP_ADDR`은 주어진 매개 변수를`(* (volatile long *)`로 캐스트하고 `+m` 제약 조건을 추가합니다. `+`는이 피연산자가 명령어에 의해 읽히고 쓰여짐을 의미합니다. `m`은 이것이 메모리 피연산자임을 의미합니다. `BITOP_ADDR`은 다음과 같이 정의됩니다.

```C
#define BITOP_ADDR(x) "+m" (*(volatile long *) (x))
```

다음은 `memory`클로버입니다. 이는 컴파일러에게 어셈블리 코드가 입력 및 출력 피연산자에 나열된 항목 이외의 항목에 대한 메모리 읽기 또는 쓰기를 수행함을 알립니다 (예 : 입력 매개 변수 중 하나가 가리키는 메모리에 액세스).

`Ir` - 즉시 레지스터 피연산자.(immediate register operand).

`bts` 명령어는 주어진 비트를 비트 문자열로 설정하고 주어진 비트의 값을 `CF` 플래그에 저장합니다. 따라서 CPU 번호(우리의 경우에는 0임)를 전달하고 `set_bit`가 실행 된 후 `cpu_online_bits` cpumask에서 0 비트를 설정합니다. 이는 현재 첫 번째 CPU가 온라인 상태임을 의미합니다.

`set_cpu_*`API 외에도 cpumask는 cpumask 조작을 위한 다른 API를 제공합니다. 간단히 다뤄봅시다.

추가적인 cpumask API
--------------------------------------------------------------------------------

cpumask는 다양한 상태의 CPU 갯수를 얻기위한 매크로 세트를 제공합니다. 예를 들면:

```C
#define num_online_cpus()	cpumask_weight(cpu_online_mask)
```

이 매크로는 `online` CPU의 갯수를 반환합니다. 이 매크로는 `cpu_online_mask` 비트맵과 함께 `cpumask_weight` 함수를 호출합니다 (읽어보세요). `cpumask_weight` 함수는 두 개의 매개 변수로`bitmap_weight` 함수를 한 번 호출합니다:

* cpumask bitmap;
* `nr_cpumask_bits` - 우리의 경우는 `NR_CPUS`.

```C
static inline unsigned int cpumask_weight(const struct cpumask *srcp)
{
	return bitmap_weight(cpumask_bits(srcp), nr_cpumask_bits);
}
```

그리고 주어진 비트맵의 비트 수를 계산합니다. `num_online_cpus` 외에도 cpumask는 모든 CPU 상태에 대한 매크로를 제공합니다.

* num_possible_cpus;
* num_active_cpus;
* cpu_online;
* cpu_possible.

그 외에도 다수

또한, 리눅스 커널은`cpumask` 조작을 위해 다음과 같은 API를 제공합니다.

* `for_each_cpu` - 마스크의 모든 CPU를 순회(iterate)합니다;
* `for_each_cpu_not` - 보수를 취한 마스크에서 모든 CPU를 순회합니다;
* `cpumask_clear_cpu` - cpumask에서 cpu를 지웁니다;
* `cpumask_test_cpu` - 마스크의 CPU를 테스트합니다;
* `cpumask_setall` - 모든 CPU를 마스크에 설정합니다;
* `cpumask_size` - `struct cpumask`에 할당할 크기를 바이트 단위로 반환합니다.

그 외에도 아주아주 많습니다...

링크
--------------------------------------------------------------------------------

* [cpumask documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
