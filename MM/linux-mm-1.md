리눅스 커널 메모리 관리 파트 1.
================================================================================

소개
--------------------------------------------------------------------------------

메모리 관리는 운영체제에서 어려운 부분 중에 하나입니다 (저는 제일 어려운 부분이라 생각합니다). [커널 엔트리 포인트로 가기 전에 마지막 준비 작업](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-3.html) 에서는 `start_kernel` 함수 직전까지 설명하였습니다. 이 함수는 커널이 `init` 프로세스가 동작하기 전에, 커널의 모든 기능(아키텍쳐 의존적인 기능까지 포함)을 초기화합니다. 커널은 early page table, fixmap page table 그리고 identity page table을 만듭니다. 아직 복잡한 메모리 관리는 들어가지 않았습니다. `start_kernel` 함수를 본격적으로 보기 시작하면 메모리 관리를 위한 복잡한 자료구조와 테크닉들을 보게 될 것입니다. 커널의 초기화 과정을 좀 더 잘 이해하기 위해서는 이러한 테크닉에 대해서 명확히 이해할 필요가 있습니다. 이 챕터는 리눅스 `memblock`을 시작으로, 커널의 메모리 관리에 있어서 다양한 프레임워크나 API들에 대해서 알아볼 것입니다.

Memblock
--------------------------------------------------------------------------------

Memblock은 초기 부팅 단계에서 커널의 메모리 할당자가 초기화가 되지 않아 제대로 동작하기 전에, 메모리를 관리하는 방법 중 하나입니다. 원래는 `Logical Memory Block`이라는 이름으로 사용되었으나, Yinghai Lu 라는 사람이 만든 [패치](https://lkml.org/lkml/2010/7/13/68) 이후에 `memblock`으로 변경되었습니다. `x86_64` 아케텍쳐를 사용하는 커널이 `memblock`를 사용하기에 [커널 엔트리 포인트로 가기 전에 마지막 준비 작업](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-3.html)에서 봤던 적이 있습니다. 이번에는 어떻게 구현되어있는지 알아보겠습니다.

먼저 `memblock`를 자료구조 입장에서 시작해보겠습니다. 논리 메모리 블락 관련 자료구조는 [include/linux/memblock.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/memblock.h)헤더 파일에 정의되어 있습니다.

첫 번째 자료구조는 `memblock`이라는 이름 그대로 구조체가 정의되어 있습니다:

```C
struct memblock {
         bool bottom_up;
         phys_addr_t current_limit;
         struct memblock_type memory;   --> array of memblock_region
         struct memblock_type reserved; --> array of memblock_region
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
         struct memblock_type physmem;
#endif
};
```

이 구조체에는 5개 변수가 선언되어 있습니다. 첫 번째 `bottom_up`는 메모리를 아래에서 위로 할당 해 줄 수 있는지 여부를 결정하는 변수이며 맞다면, `true`이다. 다음 변수는 `current_limit`는 메모리 블락의 제한된 크기를 나타냅니다. 다음 세 개의 변수들은, 메모리 블락 타입(reserved, memory 그리고 physical memory)을 나타냅니다. (타입 중에 physical memory는 `CONFIG_HAVE_MEMBLOCK_PHYS_MAP` 설정이 활성화 되어있어야 가능합니다.)
지금부터 `memblock_type` 자료구조를 봅시다.

```C
struct memblock_type {
	unsigned long cnt;
	unsigned long max;
	phys_addr_t total_size;
	struct memblock_region *regions;
};
```

이 구조체는 메모리의 타입에 대해서 나타냅니다. 각 변수들은 한 메모리 블락내에 있는 메모리 영역의 갯수, 모든 메모리 영역들의 총 사이즈 그리고 할당 된 메모리 영역의 배열 크기를 나타냅니다. 마지막으로는 `memblock_region` 구조체의 배열에 대한 포인터를 나타내는 변수가 있습니다. `memblock_region` 구조체는 메모리 영역을 표현하는 구조체로 아래와 같이 정의되어 있습니다.

```C
struct memblock_region {
        phys_addr_t base;
        phys_addr_t size;
        unsigned long flags;
#ifdef CONFIG_HAVE_MEMBLOCK_NODE_MAP
        int nid;
#endif
};
```

`memblock_region`은 메모리 영역의 시작주소와 크기를 나타내고 있으며, 아래의 값들을 갖는 플래그를 변수로 갖고 있습니다.

```C
enum {
    MEMBLOCK_NONE	= 0x0,	/* No special request */
    MEMBLOCK_HOTPLUG	= 0x1,	/* hotpluggable region */
    MEMBLOCK_MIRROR	= 0x2,	/* mirrored region */
    MEMBLOCK_NOMAP	= 0x4,	/* don't add to kernel direct mapping */
};
```

또한 만약 `CONFIG_HAVE_MEMBLOCK_NODE_MAP` 설정이 활성화 되어 있다면, `memblock_region`은 정수 타입의 변수인 [numa](http://en.wikipedia.org/wiki/Non-uniform_memory_access)를 포함하게 됩니다.  

위에서 간략하게 본 구조체들의 관계를 의미론 적으로 표현하면 아래와 같이 나타낼 수 있습니다.

```
+---------------------------+   +---------------------------+
|         memblock          |   |                           |
|  _______________________  |   |                           |
| |        memory         | |   |       Array of the        |
| |      memblock_type    |-|-->|      memblock_region      |
| |_______________________| |   |                           |
|                           |   +---------------------------+
|  _______________________  |   +---------------------------+
| |       reserved        | |   |                           |
| |      memblock_type    |-|-->|       Array of the        |
| |_______________________| |   |      memblock_region      |
|                           |   |                           |
+---------------------------+   +---------------------------+
```

`Memblock`에서는 `memblock` 구조체, `memblock_type` 구조체 그리고 `memblock_region` 구조체가 제일 주된 구조체들입니다. 이제 부터는 메모리 블락의 초기화 과정에 대해서 알아봅시다.

Memblock 초기화
--------------------------------------------------------------------------------

`memblock`에 관련된 API들에 대해서 선언은 [include/linux/memblock.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/memblock.h) 헤더파일에서, 정의는 . [mm/memblock.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/memblock.c) 파일에서 찾아 볼 수 있습니다. 소스코드의 처음 시작 부분을 보면, `memblock`구조체의 초기화관련 코드를 찾아 볼 수 있습니다:

```C
struct memblock memblock __initdata_memblock = {
	.memory.regions		= memblock_memory_init_regions,
	.memory.cnt		    = 1,
	.memory.max		    = INIT_MEMBLOCK_REGIONS,

	.reserved.regions	= memblock_reserved_init_regions,
	.reserved.cnt		= 1,
	.reserved.max		= INIT_MEMBLOCK_REGIONS,

#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
	.physmem.regions	= memblock_physmem_init_regions,
	.physmem.cnt		= 1,
	.physmem.max		= INIT_PHYSMEM_REGIONS,
#endif
	.bottom_up		    = false,
	.current_limit		= MEMBLOCK_ALLOC_ANYWHERE,
};
```

여기 `memblock`구조체 이름과 같은 객체 `memblock`의 초기화를 볼 수 있습니다. 그 전에 `__initdata_memblock`에 대해서 먼저 보면, 이 매크로는 이렇게 정의되어 있습니다:

```C
#ifdef CONFIG_ARCH_DISCARD_MEMBLOCK
    #define __init_memblock __meminit
    #define __initdata_memblock __meminitdata
#else
    #define __init_memblock
    #define __initdata_memblock
#endif
```

매크로는 `CONFIG_ARCH_DISCARD_MEMBLOCK`에 의존하고 있습니다. 이 설정이 활성화되면, memblock 코드들은 `.init` 섹션에 위치하게 되고, 커널이 부팅이 끝나고 나면 해제됩니다. 

다음으로는 `memblock` 구조체의 변수들인 `memblock_type memory`, `memblock_type reserved` 그리고 `memblock_type physmem`의 초기화 과정을 볼 수 있습니다. 여기서 `memblock_type.regions`의 초기화 과정에만 관심이 있습니다. `memblock_type`의 모든 변수들은 `memblock_region`의 배열로 초기화를 합ㄴ디ㅏ. 

```C
static struct memblock_region memblock_memory_init_regions[INIT_MEMBLOCK_REGIONS] __initdata_memblock;
static struct memblock_region memblock_reserved_init_regions[INIT_MEMBLOCK_REGIONS] __initdata_memblock;
#ifdef CONFIG_HAVE_MEMBLOCK_PHYS_MAP
static struct memblock_region memblock_physmem_init_regions[INIT_PHYSMEM_REGIONS] __initdata_memblock;
#endif
```

모든 배열들은 128개(`INIT_MEMBLOCK_REGIONS`)의 memblock_region을 갖습니다. 

```C
#define INIT_MEMBLOCK_REGIONS   128
```

그리고 또한 모든 배열들은 `__initdata_memblock` 매크로와 함께 정의 되어있습니다.
이 매크로는 위에서 `memblock` 초기화 과정에서 본 적이있는데 기억이 안나면 위를 한 번 다시 읽어보세요.

초기화 과정에서 마지막 두 개 변수를 보면, `bottom_up` 할당은 비활성화 되어있고, 현재 Memblock의 한계를 보면: 

```C
#define MEMBLOCK_ALLOC_ANYWHERE (~(phys_addr_t)0)
```

이 값은 `0xffffffffffffffff` 이다.

이로써 `memblock` 구조체 초기화 과정은 마무리하고, Memblock의 API에 대해서 살펴보도록 하겠습니다.

Memblock API
--------------------------------------------------------------------------------

`memblock` 구조체의 초기화를 살펴보았고, 지금부터는 Memblock의 API가 어떻게 구현되어있는지에 대해서 살펴보겠습니다. 위에서 언급한것 처럼, `memblock`의 구현은 [mm/memblock.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/memblock.c) 에 되어있습니다. `memblock`이 어떻게 동작하는지 그리고 어떻게 구현되어있는지를 이해하기 위해서는 어떻게 사용되는지를 살펴보겠습니다. 커널에는 memblock이 [몇 차례](http://lxr.free-electrons.com/ident?i=memblock) 사용되고 있습니다. 예를 들어, [arch/x86/kernel/e820.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/e820.c#L1061)에 있는 `memblock_x86_fill` 함수가 있겠습니다. 이 함수는 [e820](http://en.wikipedia.org/wiki/E820)에서 제공하는 메모리 맵을 따르고 있고, 커널은 reserved에 해당하는 memory region들을 `memblock`에 `memblock_add` 함수를 이용하여 추가하고 있습니다. 우리가 `memblock_add`함수를 처음으로 만났으니, 이 함수부터 시작해보겠습니다.

이 함수는 물리 메모리의 시작 주소 그리고 사이즈를 함수인자로 받아 `memblock`에 추가합니다. `memblock_add` 함수는 자체적으로는 하는 것이 없고 아래 함수를 호출합니다:

```C
memblock_add_range(&memblock.memory, base, size, MAX_NUMNODES, 0);
```

이 함수의 인자로 메모리 블락 타입을 `memory`로 전달하고 있습니다. 그리고 물리 메모리의 시작 주소와 사이즈, 그리고 노드 갯수의 최대인 1(`CONFIG_NODES_SHIFT`가 비활성화 되어있으면 1, 활성화 되어있다면 `1 << CONFIG_NODES_SHIFT`)을 마지막으로는 플레그를 전달하고 있습니다. `memblock_add_range` 함수는 새로운 메모리 영역을 메모리 블락에 추가해주는 함수입니다. 함수는 인자로 넘어온 사이즈를 체크(사이즈가 0인지 확인 후 0이라면 바로 리턴)하는 것부터 시작합니다. 그리고 `memblock_add_range`함수는 `memblock`구조체에 넘어온 `memblock_type`의 존재 여부를 체크합니다. 만약 존재하지 않는다면(비어있다면), 하나의 노드를 추가하고 바로 리턴해버립니다. (이미 이 부현부분은 [First touch of the linux kernel memory manager framework](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-3.html)에서 확인한 적이 있습니다.) 만약 `memblock_type`이 비어있지 않다면, 새로운 메모리 영역을 `memblock`에 주어진 `memblock_type`으로 추가합니다.

무엇보다도, 우리는 메모리의 마지막 주소를 아래의 방법으로 얻을 수 있습니다:

```C
phys_addr_t end = base + memblock_cap_size(base, &size);
```

`memblock_cap_size` 는 `base + size`가 오버플로우가 발생하지 않도록 `size`를 잘 조절합니다. 구현은 생각보다 쉽습니다:

```C
static inline phys_addr_t memblock_cap_size(phys_addr_t base, phys_addr_t *size)
{
	return *size = min(*size, (phys_addr_t)ULLONG_MAX - base);
}
```

`memblock_cap_size`함수는 주어진 사이즈와 `ULLONG_MAX - base` 중에 더 작은 값을 리턴합니다.

새로운 메모리 영역의 마지막 주소를 계산해 낸 다음에, `memblock_add_range` 함수는 새로운 메모리 영역과 기존에 추가된 영역사이에 겹쳐지는 부분 그리고 합쳐지는 조건을 확인한다. `memblock`에 새로운 메모리 영역을 추가하는  것은 두 단계로 나누어 집니다:

* 겹쳐지는 영역이 없는 메모리에 대해서 새로운 메모리 영역으로 추가한다.
* 인접한 메모리 영역에 대해서는 하나로 합친다.

우리는 이미 추가된 메모리 영역에 대해서 새로운 메모리 영역 사이에 겹쳐지는 부분이 있는 지 살펴볼 겁니다:

```C
	for (i = 0; i < type->cnt; i++) {
		struct memblock_region *rgn = &type->regions[i];
		phys_addr_t rbase = rgn->base;
		phys_addr_t rend = rbase + rgn->size;

		if (rbase >= end)
			break;
		if (rend <= base)
			continue;
        ...
		...
		...
	}
```

`memblock`에 이미 추가된 메모리 영역과 새로 더해 질 메모리 영역 사이에 겹쳐지는 영역이 없다면, 기존의 영역과는 다른 메모리 영역으로 추가해주는 것이 첫 번재 단계입니다. 그리고 새로운 메모리 영역이 메모리 블락에 알맞은지 확인한다. 그리고 `memblock_double_array`함수를 다른 방법으로 호출합니다:

```C
while (type->cnt + nr_new > type->max)
	if (memblock_double_array(type, obase, size) < 0)
		return -ENOMEM;
	insert = true;
	goto repeat;
```

`memblock_double_array` 함수는 영역들의 배열의 크기를 두 배로 만들어줍니다. 그리고 `insert`변수를 `true` 로 설정해주고, `repeat` 라벨로 이동합니다. 두 번째 단계로는 `repeat` 라벨부터 시작해서 같은 반복문을 실행하고, 현재 메모리 영역을 `memblock_insert_region`함수로 메모리 블락에 삽입해줍니다:

```C
	if (base < end) {
		nr_new++;
		if (insert)
			memblock_insert_region(type, i, base, end - base,
					       nid, flags);
	}
```

`insert`를 `true`로 첫 번재 단계에서 변경해주었기 때문에, `memblock_insert_region`함수는 호출될 것입니다. `memblock_insert_region`함수는 위에서 설명했던 비어 있는 `memblock_type`에 새로운 메모리 영역을 추가하는 것과 매우 유사하게 구현되어 있습니다. (위 코드들을 참조)
이 함수는 마지막 메모리 영역을 얻어 냅니다:

```C
struct memblock_region *rgn = &type->regions[idx];
```

그리고 `memmove`함수로 메모리를 복사합니다:

```C
memmove(rgn + 1, rgn, (type->cnt - idx) * sizeof(*rgn));
```

그 후에는 `memblock_region`함수로 새로운 메모리 영역의 시작주소, 크기 등을 체웁니다. 그리고 `memblock_type`의 크기를 증가시킵니다. 수행의 마지막에는 `memblock_add_range`함수는 두 번째 단계에 해당하는 근접한 메모리들을 합치는 `memblock_merge_regions`함수를 호출합니다.

두 번째 단계에서 새로운 메모리 영역은 다른 기존의 영역과 겹쳐질 수 있습니다. 예를 들어 `memblock`에 `region1`영역이 존재하고 있었다고 한다면:

```
0                    0x1000
+-----------------------+
|                       |
|                       |
|        region1        |
|                       |
|                       |
+-----------------------+
```
`memblock`에 아래와 같은 `region2`를 추가하려한다고 해보겠습니다.

```
0x100                 0x2000
+-----------------------+
|                       |
|                       |
|        region2        |
|                       |
|                       |
+-----------------------+
```
이런 경우, 새로운 메모리 영역의 시작주소를 인접하는 영역의 끝 주소로 설정합니다:

```C
base = min(rend, end);
```

그래서 우리의 경우 `0x1000`번지가 새로운 메모리 영역의 시작주소가 됩니다. 그리고 두 번째 단계에서 우리가 했던 것처럼 추가해줍니다:

```
if (base < end) {
	nr_new++;
	if (insert)
		memblock_insert_region(type, i, base, end - base, nid, flags);
}
```

이 경우, 우리는 겹치지 않는 나머지 영역(overlapping portion)만을 분리하여 삽입합니다. (아랫부분의 메모리는 이미 기존 영역과 겹쳐져 있으므로, 겹치지 않는 윗부분의 영역만 새로 삽입하게 됩니다.) 그 후 남은 부분들을 추가하고, `memblock_merge_regions` 함수를 호출하여 두 메모리 영역을 하나로 합칩니다.

앞서 언급했듯이, `memblock_merge_regions` 함수는 인접한 동일한 속성의 메모리 영역들을 병합하는 역할을 합니다. 이 함수는 인자로 전달된 `memblock_type` 내의 모든 메모리 영역을 하나씩 확인하면서 서로 인접한 영역들을 병합해 나갑니다. 이때 `type->regions[i]`(현재 영역)와 `type->regions[i + 1]`(다음 영역)을 비교하며, 이웃한 두 메모리 영역의 플래그가 일치하는지, 그리고 동일한 노드에 속해 있는지(즉, 첫 번째 메모리 영역의 끝 주소와 두 번째 메모리 영역의 시작 주소가 딱 맞닿아 있는지) 확인합니다.

```C
while (i < type->cnt - 1) {
	struct memblock_region *this = &type->regions[i];
	struct memblock_region *next = &type->regions[i + 1];
	if (this->base + this->size != next->base ||
	    memblock_get_region_node(this) !=
	    memblock_get_region_node(next) ||
	    this->flags != next->flags) {
		BUG_ON(this->base + this->size > next->base);
		i++;
		continue;
	}
```

이 조건 중 하나라도 `true`가 아니면, 첫 번째 메모리 영역의 사이즈를 두 번째 메모리 영역의 사이즈로 업데이트합니다:

```C
this->size += next->size;
```

첫 번재 메모리 영역의 사이즈를 다음 번 메모리 영역의 사이즈로 업데이트함에 따라서, 우리는 (`next`) 메모리 영역 뒤의 모든 메모리 영역을`memmove` 함수로 역순으로 하나의 인덱스로 이동시킵니다 :

```C
memmove(next, next + 1, (type->cnt - (i + 2)) * sizeof(*next));
```

`memmove` 함수는`next` 영역 다음에 위치하는 모든 영역을 `next` 영역의 시작 주소로 이동시킵니다. 마지막으로`memblock_type`에 속한 메모리 영역의 수를 줄입니다.:

```C
type->cnt--;
```

이 후에는 두 메모리 영역을 하나로 합칩니다.

```
0                                             0x2000
+------------------------------------------------+
|                                                |
|                                                |
|                   region1                      |
|                                                |
|                                                |
+------------------------------------------------+
```

특정 타입의 메모리 블록 내 영역 개수(`cnt`)를 줄였으므로, 현재 영역(`this`)의 크기를 늘리고 `next` 영역 다음에 위치한 모든 영역을 원래 next가 있던 위치로 이동시켰습니다. 

그게 다입니다. 이것이 `memblock_add_range` 함수가 작동하는 전체 원리입니다.

memblock_reserve 에 대해선, 이 함수는 memblock_add와 동일하지만 한 가지 차이점이 있습니다. memblock_type.emory 대신 memblock_type.reserved를 저장합니다.

당연히 이것이 API의 전부는 아닙니다. Memblock은 메모리 및 예약된 메모리 영역을 추가하는 것뿐만 아니라 다음과 같은 다양한 API를 제공합니다:

- `memblock_remove` - memblock에서 메모리 영역을 제거합니다.
- `memblock_find_in_range` - 주어진 범위 내에서 빈 공간을 찾습니다.
- `memblock_free` - memblock에서 메모리 영역을 해제합니다.
- `for_each_mem_range` - memblock 영역들을 순회(반복문)합니다.

그리고 이 외에도 더 많은 API가 있습니다....

Getting info about memory regions
--------------------------------------------------------------------------------

Memblock은 Memblock에 할당된 메모리 영역에 대한 정보를 얻기 위한 API를 제공합니다. 이 API는 두 부분으로 나뉩니다:

`get_allocated_memblock_memory_regions_info` - 메모리 영역에 대한 정보 가져옵니다;

`get_allocated_memblock_reserved_regions_info` - 예약된 지역에 대한 정보를 가져옵니다.

이러한 함수의 구현은 간단합니다. 예를 들어 `get_allocated_memblock_reserved_regions_info`를 살펴보겠습니다:

```C
phys_addr_t __init_memblock get_allocated_memblock_reserved_regions_info(
					phys_addr_t *addr)
{
	if (memblock.reserved.regions == memblock_reserved_init_regions)
		return 0;

	*addr = __pa(memblock.reserved.regions);

	return PAGE_ALIGN(sizeof(struct memblock_region) *
			  memblock.reserved.max);
}
```

우선, 이 함수는 `memblock`에 예약된 메모리 영역이 포함되어 있는지 확인합니다. `memblock`에 예약된 메모리 영역이 없다면 0을 반환합니다. 

그렇지 않다면, 예약된 메모리 영역 배열의 물리 주소(Physical Address)를 인자로 주어진 주소(`addr`)에 기록하고, 할당된 배열의 정렬된 크기(aligned size)를 반환합니다. 정렬에 `PAGE_ALIGN` 매크로가 사용된다는 점에 유의하세요. 이 매크로는 실제로는 페이지 크기에 의존합니다:

```C
#define PAGE_ALIGN(addr) ALIGN(addr, PAGE_SIZE)
```

`get_allocated_memblock_memory_regions_info` 함수의 구현도 동일합니다. `memblock_type.reserved` 대신 `memblock_type.memory`라는 한가지 차이점만 있습니다.

Memblock debugging
--------------------------------------------------------------------------------

memblock의 구현 코드를 보면 `memblock+dbg` 함수가 자주 호출되는 것을 볼 수 있습니다. 커널 커맨드 라인(부팅 옵션)에 `memblock=debug` 옵션을 전달하면 이 함수가 활성화되어 호출됩니다. 실제로 'memblock_dbg'는 'printk'로 확장되는 매크로일 뿐입니다:

```C
#define memblock_dbg(fmt, ...) \
         if (memblock_debug) printk(KERN_INFO pr_fmt(fmt), ##__VA_ARGS__)
```

예를 들어, `memblock_reserve` 함수 내부에서 이 매크로가 어떻게 호출되는지 볼 수 있습니다.

```C
memblock_dbg("memblock_reserve: [%#016llx-%#016llx] flags %#02lx %pF\n",
		     (unsigned long long)base,
		     (unsigned long long)base + size - 1,
		     flags, (void *)_RET_IP_);
```

그러면 부팅 로그(`dmesg`)에서 다음과 같은 형태의 디버깅 메시지를 확인할 수 있게 됩니다:

<! -![Memblock](http://oi57.tinypic.com/1zoj589.jpg) ->

`memblock_reserve: [0x0000000000100000-0x0000000000ffffff] flags 0x00 stext+0x0/0x0`

또한 Memblock은 가상 파일 시스템인 debugfs를 통한 조회도 지원합니다. 만약 X86 이외의 아키텍처에서 커널을 실행 중이라면, 아래 경로에 접근하여 Memblock의 내용이 덤프(Dump)된 정보를 확인할 수 있습니다:

* /sys/kernel/debug/memblock/memory
* /sys/kernel/debug/memblock/reserved
* /sys/kernel/debug/memblock/physmem

to get a dump of the `memblock` contents.

Conclusion
--------------------------------------------------------------------------------

이것으로 리눅스 커널 메모리 관리에 관한 첫 번째 파트를 마칩니다. 

**질문/제안(내용 관련)**: 질문이나 제안할 것이 있으면 언제든 트위터 [@0xAX](https://twitter.com/0xAX)를 핑(ping)하거나 [이슈](https://github.com/0xAX/linux-insides/issues/new)를 추가하세요. 아니면 원저자에게 간단히 [이메일](mailto:anotherworldofworld@gmail.com)을 주세요.

**질문/제안(번역 repo 관련)**: linux-insides-ko 레포지토리 관련해서 제안할 것이나 궁금한 것 등등이 있으시면 언제든 @junsooo에게 간단히 [이메일](mailto:freshmilk264@gmail.com)이나 [이슈](https://github.com/junsooo/linux-insides-ko/issues/new)를 추가해주세요.

Links
--------------------------------------------------------------------------------

* [e820](http://en.wikipedia.org/wiki/E820)
* [numa](http://en.wikipedia.org/wiki/Non-uniform_memory_access)
* [debugfs](http://en.wikipedia.org/wiki/Debugfs)
* [First touch of the linux kernel memory manager framework](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-3.html)
