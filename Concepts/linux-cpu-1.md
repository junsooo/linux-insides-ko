CPU별 변수
================================================================================

CPU별 변수들은 커널 기능의 하나입니다. 이름을 읽으면 이 기능의 의미를 이해할 수 있을 것입니다. 우리는 변수를 만들 수 있고 각 프로세서 코어는 자체적으로 이 변수의 복사본을 가질 것입니다. 이 부분에서는 이 기능을 자세히 살펴보고 구현 방법 및 동작 방법을 이해하기 위해 노력할 것입니다.

커널은 CPU별 변수를 만드는 API를 제공합니다 - `DEFINE_PER_CPU` 메크로입니다:

```C
#define DEFINE_PER_CPU(type, name) \
        DEFINE_PER_CPU_SECTION(type, name, "")
```

이 메크로는 CPU별 변수와 함께 동작하는 다른 많은 매크로들과 마찬가지로 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/percpu-defs.h) 파일에 정의되어 있습니다. 

`DEFINE_PER_CPU` 정의를 살펴보십시오. `type`과 `name`, 2개의 파라미터를 가짐을 볼 수 있습니다: CPU별 변수를 생성하기 위해 그 파라미터들을 사용할 수 있으며, 예를 들면 다음과 같습니다:

```C
DEFINE_PER_CPU(int, per_cpu_n)
```

변수의 타입과 이름을 전달합니다. `DEFINE_PER_CPU`은 `DEFINE_PER_CPU_SECTION` 매크로를 호출하고 동일한 2개의 파라미터와 빈 문자열을 전달합니다. `DEFINE_PER_CPU_SECTION` 정의를 살펴보십시오:

```C
#define DEFINE_PER_CPU_SECTION(type, name, sec)    \
         __PCPU_ATTRS(sec) PER_CPU_DEF_ATTRIBUTES  \
         __typeof__(type) name
```

```C
#define __PCPU_ATTRS(sec)                                                \
         __percpu __attribute__((section(PER_CPU_BASE_SECTION sec)))     \
         PER_CPU_ATTRIBUTES
```

여기서 `section`은:

```C
#define PER_CPU_BASE_SECTION ".data..percpu"
```

모든 매크로가 확장된 후에 우리는 전역 CPU별 변수를 얻을 것입니다.

```C
__attribute__((section(".data..percpu"))) int per_cpu_n
```

이는 `.data..percpu` 섹션에 `per_cpu_n` 변수가 있음을 의미합니다. 우리는 `vmlinux`에서 이 섹션을 찾을 수 있습니다:

```
.data..percpu 00013a58  0000000000000000  0000000001a5c000  00e00000  2**12
              CONTENTS, ALLOC, LOAD, DATA
```

OK, 이제 우리는 `DEFINE_PER_CPU` 매크로를 사용할 때, `.data..percpu` 섹션에 있는 CPU별 변수가 생성될 것을 알고 있습니다. 커널이 초기화할 때, `.data..percpu` 섹션을 여러번 로드하는 `setup_per_cpu_areas` 함수를 호출합니다. 이 함수는 CPU당 하나의 섹션입니다.

CPU별 영역 초기화 과정을 살펴보겠습니다. 그것은 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)소스에서, [arch/x86/kernel/setup_percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/setup_percpu.c)에 정의된 `setup_per_cpu_areas` 함수 호출로 시작됩니다.

```C
pr_info("NR_CPUS:%d nr_cpumask_bits:%d nr_cpu_ids:%d nr_node_ids:%d\n",
        NR_CPUS, nr_cpumask_bits, nr_cpu_ids, nr_node_ids);
```

`setup_per_cpu_areas`는 커널 구성 중에 설정한 최대 CPU 수에 대한 출력 정보에서 시작됩니다. 설정 항목은 `CONFIG_NR_CPUS` 구성 옵션, 실제 CPU 수, `nr_cpumask_bits`(`NR_CPUS` 비트와 동일. `NR_CPUS` 비트는 새로운 `cpumask` 연산자와 `NUMA` 노드 수를 위한 것) 등입니다.

우리는 dmesg 안에서 이러한 출력을 볼 수 있습니다:

```
$ dmesg | grep percpu
[    0.000000] setup_percpu: NR_CPUS:8 nr_cpumask_bits:8 nr_cpu_ids:8 nr_node_ids:1
```

다음 단계에서 우리는 `percpu` 첫번째 청크 할당자(chunk allocator)를 검사합니다. 모든 percpu 영역은 청크 안에 할당됩니다. 첫번째 청크는 static한 percpu 변수를 위해 사용됩니다. 리눅스 커널은 첫번째 청크 할당자의 타입을 제공하는 `percpu_alloc` 명령줄 파라미터를 가집니다. 우리는 커널 문서에서 이것에 대해 읽을 수 있습니다:

```
percpu_alloc=	Select which percpu first chunk allocator to use.
		Currently supported values are "embed" and "page".
		Archs may support subset or none of the	selections.
		See comments in mm/percpu.c for details on each
		allocator.  This parameter is primarily	for debugging
		and performance comparison.
```

[mm/percpu.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/percpu.c)소스는 이 명령줄 옵션의 핸들러를 포함합니다:

```C
early_param("percpu_alloc", percpu_alloc_setup);
```

`pcpu_chosen_fc` 변수를 설정하는 `percpu_alloc_setup` 함수는 `percpu_alloc` 파라미터 값에 의존합니다. 첫번째 청크 할당자는 기본적으로 `auto` 입니다.

```C
enum pcpu_fc pcpu_chosen_fc __initdata = PCPU_FC_AUTO;
```

만약 `percpu_alloc` 파라미터가 커널 명령줄에 주어지지 않는다면, `embed` 할당자가 [memblock](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html)와 함께 bootmem에 첫번째 percpu 청크를 끼워넣는 데에 사용될 것입니다. 마지막 할당자는, 첫번째 청크를 `PAGE_SIZE` 페이지에 매핑하는 첫번째 청크 `page` 할당자입니다.

제가 위에 쓴데로, 우선 우리는 `setup_per_cpu_areas` 안에 있는 첫번째 청크 할당자 타입을 확인합니다. 첫번째 청크 할당자가 페이지가 아닌지 확인합니다:

```C
if (pcpu_chosen_fc != PCPU_FC_PAGE) {
    ...
    ...
    ...
}
```

만약 `PCPU_FC_PAGE`가 아니라면, 우리는 `embed` 할당자를 사용하고 `pcpu_embed_first_chunk` 함수와 함께 첫번째 청크를 위한 공간을 할당할 것입니다.

```C
rc = pcpu_embed_first_chunk(PERCPU_FIRST_CHUNK_RESERVE,
					    dyn_size, atom_size,
					    pcpu_cpu_distance,
					    pcpu_fc_alloc, pcpu_fc_free);
```

위에서 보았듯, `pcpu_embed_first_chunk` 함수는 첫번째 CPU별 청크를 bootmem에 내장시키고 `pcup_embed_first_chunk`에 몇개의 파라미터를 전달합니다. 파라미터는 다음과 같습니다:

* `PERCPU_FIRST_CHUNK_RESERVE` - static `percpu` 변수를 위한 예약공간의 크기(사이즈);
* `dyn_size` - 동적 할당을 위한 최소의 여유 사이즈(바이트);
* `atom_size` - 모든 할당은 이것의 배수이며, 이 파라미터와 결합된다;
* `pcpu_cpu_distance` - cpu 사이에서 거리를 결정하는 콜백;
* `pcpu_fc_alloc` - `percpu` 페이지를 할당하는 함수;
* `pcpu_fc_free` - `percpu` 페이지를 해제하는 함수.

우리는 모든 파라미터를 `pcpu_embed_first_chunk` 호출 전에 계산합니다:

```C
const size_t dyn_size = PERCPU_MODULE_RESERVE + PERCPU_DYNAMIC_RESERVE - PERCPU_FIRST_CHUNK_RESERVE;
size_t atom_size;
#ifdef CONFIG_X86_64
		atom_size = PMD_SIZE;
#else
		atom_size = PAGE_SIZE;
#endif
```

만약 첫번째 청크 할당자가 `PCPU_FC_PAGE`라면, 우리는 `pcpu_embed_first_chunk` 대신에 `pcpu_page_first_chunk`를 사용할 것입니다. 그 `percpu` 영역이 할당된 이후, 우리는 `percpu` 오프셋과 그 세그먼트를 설정합니다.
이것은 `setup_percpu_segment` 함수를 가진 모든 CPU에 대한 것 입니다. 그리고 모든 데이터를 배열에서 `percpu` 변수(`x86_cpu_to_apicid`, `irq_stack_ptr` 등)로 옮깁니다.
커널이 초기화 과정을 끝낸 후, 우리는 N `.data..percpu` 섹션을 로드했을 것이며, N은 CPU의 수이고 부트스트랩 프로세서가 사용하는 섹션은 `DEFINE_PER_CPU` 매크로와 함께 생성된 초기화되지 않은 변수를 포함할 것입니다.

커널은 cpu별 변수를 조작할 수 있는 API를 제공합니다:

* get_cpu_var(var)
* put_cpu_var(var)


`get_cpu_var` 구현을 살펴봅시다:

```C
#define get_cpu_var(var)     \
(*({                         \
         preempt_disable();  \
         this_cpu_ptr(&var); \
}))
```

리눅스 커널은 선점가능하며 cpu별 변수에 접근하려면 커널이 실행중인 프로세서를 알 필요가 있습니다. 그래서, 현재의 코드는 cpu별 변수를 접근하는 동안에 선점되어 다른 CPU로 옮겨지면 안됩니다. 그래서, 우선 `preempt_disable`를 호출한 다음 `this_cpu_ptr` 매크로를 호출하는 것을 볼 수 있습니다:

```C
#define this_cpu_ptr(ptr) raw_cpu_ptr(ptr)
```

and

```C
#define raw_cpu_ptr(ptr)        per_cpu_ptr(ptr, 0)
```

여기서 `per_cpu_ptr`는 주어진 cpu(2번째 변수)에 대한 CPU별 변수의 포인터를 리턴합니다. 우리가 cpu별 변수를 만들고 수정한 후에, `preempt_enable` 함수를 호출하여 선점하도록 하는 `put_cpu_var` 매크로를 호출해야 합니다. 그래서 CPU별 변수의 일반적인 사용법은 다음과 같습니다:


```C
get_cpu_var(var);
...
//Do something with the 'var'
...
put_cpu_var(var);
```

`per_cpu_ptr` 매크로를 보십시오:

```C
#define per_cpu_ptr(ptr, cpu)                             \
({                                                        \
        __verify_pcpu_ptr(ptr);                           \
         SHIFT_PERCPU_PTR((ptr), per_cpu_offset((cpu)));  \
})
```

위에 쓴 대로, 이 매크로는 주어진 CPU에 대한 CPU별 변수를 리턴합니다. 우선 그것은 `__verify_pcpu_ptr`를 호출합니다:

```C
#define __verify_pcpu_ptr(ptr)
do {
	const void __percpu *__vpp_verify = (typeof((ptr) + 0))NULL;
	(void)__vpp_verify;
} while (0)
```

`const void __percpu *`의 주어진 `ptr` 타입을 만듭니다.

이후 2개의 파라미터를 가지는 `SHIFT_PERCPU_PTR` 매크로의 호출을 볼 수 있습니다. 첫번째 파라미터로 우리의 ptr를 전달하고 두번째 파라미터로 cpu 번호를 `per_cpu_offset` 매크로에 전달합니다:


```C
#define per_cpu_offset(x) (__per_cpu_offset[x])
```

이것은 `__per_cpu_offset` 배열로부터 `x`  요소를 얻는 것으로 확장됩니다:


```C
extern unsigned long __per_cpu_offset[NR_CPUS];
```

`NR_CPUS`는 CPU들의 수입니다. `__per_cpu_offset` 배열은 CPU 변수 복사본 사이의 거리로 채워집니다. 예를 들면 모든 CPU별 데이터는 `X` 바이트이기 때문에 `__per_cpu_offset [Y]`에 접근하면`X * Y`에 액세스됩니다. `SHIFT_PERCPU_PTR` 구현을 살펴보십시오:

```C
#define SHIFT_PERCPU_PTR(__p, __offset)                                 \
         RELOC_HIDE((typeof(*(__p)) __kernel __force *)(__p), (__offset))
```

`RELOC_HIDE`는 단지 `(typeof(ptr)) (__ptr + (off))` 오프셋을 리턴하고, 그것은 변수에 대한 포인터를 리턴할 것입니다.

이것이 전부입니다! 물론 전체 API는 아니지만, 일반적인 개요입니다. 시작하기는 어려울 지 모르지만, CPU별 변수를 이해하기 위해서 당신은 주로 [include/linux/percpu-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/percpu-defs.h) 마법을 이해할 필요가 있습니다.

CPU별 변수의 포인터를 얻는 알고리즘을 다시 살펴보겠습니다:

* 초기화 과정에서 커널은 여러 개의 `.data..percpu` 섹션(CPU당 하나)을 생성합니다;
* `DEFINE_PER_CPU` 매크로와 함께 생성된 모든 변수들은 첫번째 섹션이나 CPU0을 위해 재할당될 것입니다;
* `__per_cpu_offset` 배열은 `.data..percpu` 섹션들 사이의 거리(`BOOT_PERCPU_OFFSET`)로 채워집니다;
* `per_cpu_ptr`가 호출될 때, 예를 들어 세번째 CPU를 위한 특정 CPU별 변수에 대한 포인터를 얻는 경우,  `__per_cpu_offset` 배열이 액세스 될 것입니다. offset 배열은 모든 인덱스가 필요로 하는 CPU를 가리킵니다.

이것이 전부입니다.
