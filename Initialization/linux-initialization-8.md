커널 초기화. Part 8.
================================================================================

스케줄러 초기화
================================================================================

이것은 Linux 커널 초기화 프로세스 장의 여덟 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Initialization/index.html)이며 [이전 파트](https://github.com/0xAX/linux-insides/blob/master/Initialization/linux-initialization-7.md)에선 `setup_nr_cpu_ids` 함수에서 멈췄었습니다. 

이번 파트의 주요 요점은 [스케줄러](http://en.wikipedia.org/wiki/Scheduling_%28computing%29) 초기화입니다. 그러나 스케줄러의 초기화 프로세스를 배우기 전에 몇 가지가 필요합니다. [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)에서 다음 단계는`setup_per_cpu_areas` 함수입니다. 이 함수는 `percpu` 변수를 위한 메모리 구역을 설정합니다. 자세한 내용은 특별히 [Per-CPU variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)에 대한 파트에서 읽을 수 있습니다. `percpu` 영역이 가동되어 실행되기 시작하면 다음 단계는`smp_prepare_boot_cpu` 함수입니다.

이 함수는 [symmetric multiprocessing](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)을 위한 몇가지 준비를합니다. 이 함수는 각 아키텍처에 따라 맞춰져있으므로, [arch/x86/include/asm/smp.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/include/asm/smp.h#L78) Linux 커널 헤더 파일에 있습니다. 이 함수의 정의를 살펴봅시다 :

```C
static inline void smp_prepare_boot_cpu(void)
{
         smp_ops.smp_prepare_boot_cpu();
}
```

여기서는`smp_ops` 구조체의`smp_prepare_boot_cpu` 콜백을 호출하는 것을 볼 수 있습니다. [arch/x86/kernel/smp.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/smp.c) 소스 코드 파일에서 이 구조체의 인스턴스의 정의를 보면 `smp_prepare_boot_cpu`가 `native_smp_prepare_boot_cpu` 함수의 호출로 확장됨을 알 수 있습니다.

```C
struct smp_ops smp_ops = {
    ...
    ...
    ...
    smp_prepare_boot_cpu = native_smp_prepare_boot_cpu,
    ...
    ...
    ...
}
EXPORT_SYMBOL_GPL(smp_ops);
```

`native_smp_prepare_boot_cpu` 함수의 생김새는 다음과 같습니다:

```C
void __init native_smp_prepare_boot_cpu(void)
{
        int me = smp_processor_id();
        switch_to_new_gdt(me);
        cpumask_set_cpu(me, cpu_callout_mask);
        per_cpu(cpu_state, me) = CPU_ONLINE;
}
```

그리고 이 함수는 다음과 같은 것들을 실행합니다 : 우선`smp_processor_id` 함수를 사용하여 현재 CPU의 `id` (부트스트랩 프로세서이고 이 순간에 `id`는 0) 를 얻습니다. 이미 [Kernel entry point](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-4.html)파트에서 보았기 때문에 `smp_processor_id`가 어떻게 작동하는지는 설명하지 않겠습니다. 프로세서`id` 번호를 얻은 후에는 `switch_to_new_gdt` 함수를 사용하여 주어진 CPU에 대해 [Global Descriptor Table](http://en.wikipedia.org/wiki/Global_Descriptor_Table)을 다시 로드합니다 :

```C
void switch_to_new_gdt(int cpu)
{
        struct desc_ptr gdt_descr;

        gdt_descr.address = (long)get_cpu_gdt_table(cpu);
        gdt_descr.size = GDT_SIZE - 1;
        load_gdt(&gdt_descr);
        load_percpu_segment(cpu);
}
```

`gdt_descr` 변수는 여기서 `GDT` 디스크립터에 대한 포인터를 나타냅니다 (이미 [Early interrupt and exception handling](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-2.html) 부분에서`desc_ptr` 구조체의 정의를 보았습니다). 우리는 주어진`id`를 가진 `CPU`에 대한 `GDT` 디스크립터의 주소와 크기를 얻습니다. `GDT_SIZE`는`256`이거나:

```C
#define GDT_SIZE (GDT_ENTRIES * 8)
```

이며, `get_cpu_gdt_table`을 통해 얻을 수있는 서술자의 주소는 :

```C
static inline struct desc_struct *get_cpu_gdt_table(unsigned int cpu)
{
        return per_cpu(gdt_page, cpu).gdt;
}
```

`get_cpu_gdt_table`은 주어진 CPU 번호에 대한 `gdt_page` percpu 변수의 값을 얻기 위해 `per_cpu` 매크로를 사용합니다. (이 경우에는 `id` -0 인 부트스트랩 프로세서)

다음과 같은 의문이 생길 수도 있습니다: 그래서 만약 우리가`gdt_page` percpu 변수에 접근 할 수 있다면, 어디에 정의되어 있나요? 사실 우리는 이미 이 책에서 그것을 보았습니다. 이 장의 첫 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-1.html)를 읽었다면 [arch/x86/kernel/head_64.S](https://github.com/0xAX/linux/blob/0a07b238e5f488b459b6113a62e06b6aab017f71/arch/x86/kernel/head_64.S)에서 `gdt_page`의 정의를 보았던 것을 기억할 것입니다 :

```assembly
early_gdt_descr:
	.word	GDT_ENTRIES*8-1
early_gdt_descr_base:
	.quad	INIT_PER_CPU_VAR(gdt_page)
```

또한 [링커](https://github.com/0xAX/linux/blob/0a07b238e5f488b459b6113a62e06b6aab017f71/arch/x86/kernel/vmlinux.lds.S) 파일을 살펴보면`__per_cpu_load `기호다음에 위치한 것을 볼 수 있습니다 :

```C
#define INIT_PER_CPU(x) init_per_cpu__##x = x + __per_cpu_load
INIT_PER_CPU(gdt_page);
```

[arch/x86/kernel/cpu/common.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/kernel/cpu/common.c#L94)에서`gdt_page`를 채웠습니다 :

```C
DEFINE_PER_CPU_PAGE_ALIGNED(struct gdt_page, gdt_page) = { .gdt = {
#ifdef CONFIG_X86_64
	[GDT_ENTRY_KERNEL32_CS]		= GDT_ENTRY_INIT(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_CS]		= GDT_ENTRY_INIT(0xa09b, 0, 0xfffff),
	[GDT_ENTRY_KERNEL_DS]		= GDT_ENTRY_INIT(0xc093, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER32_CS]	= GDT_ENTRY_INIT(0xc0fb, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_DS]	= GDT_ENTRY_INIT(0xc0f3, 0, 0xfffff),
	[GDT_ENTRY_DEFAULT_USER_CS]	= GDT_ENTRY_INIT(0xa0fb, 0, 0xfffff),
    ...
    ...
    ...
```

`percpu` 변수에 대한 자세한 내용은 [Per-CPU variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html) 파트를 참고해주세요. `GDT` 디스크립터의 주소와 크기를 얻었으므로 `load_gdt`로 `GDT`를 다시 로드합니다. `load_gdt`는 단지 `lgdt` 명령을 실행하고 다음 함수를 사용하여 `percpu_segment`를 로드하는 함수입니다:

```C
void load_percpu_segment(int cpu) {
    loadsegment(gs, 0);
    wrmsrl(MSR_GS_BASE, (unsigned long)per_cpu(irq_stack_union.gs_base, cpu));
    load_stack_canary_segment();
}
```

`percpu` 영역의 베이스 주소는 `gs`레지스터 (또는 `x86`의 경우 `fs`레지스터)를 포함해야하므로 우리는 `loadsegment` 매크로를 사용하고 `gs`를 전달합니다. 다음 단계에서 [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29) 스택 및 설정 스택이 [canary](http://en.wikipedia.org/wiki/Buffer_overflow_protection)인 경우 베이스 주소를 작성합니다. (`x86_32`에만 해당) 새로운`GDT`를 로드 한 후에는, 현재 CPU로 `cpu_callout_mask` 비트맵을 채우고 `cpu_state` percpu 변수를 현재 프로세서에 대한 값-`CPU_ONLINE`-으로 설정해 cpu 상태를 온라인으로 설정합니다 :

```C
cpumask_set_cpu(me, cpu_callout_mask);
per_cpu(cpu_state, me) = CPU_ONLINE;
```

그래서 `cpu_callout_mask`비트 맵은 뭘까요... 부트스트랩 프로세서 (x86에서 첫 번째로 부팅되는 프로세서)를 초기화함에 따라 멀티프로세서 시스템의 다른 프로세서들은 '보조 프로세서'(secondary processors)라고합니다. 리눅스 커널은 다음 두 비트 마스크를 사용합니다.

* `cpu_callout_mask`
* `cpu_callin_mask`

부트 스트랩 프로세서가 초기화되면 커널은 `cpu_callout_mask`를 업데이트하여 다음으로 초기화 할 수 있는 보조 프로세서를 나타냅니다. All other or secondary processors can do some initialization stuff before and check the `cpu_callout_mask` on the boostrap processor bit. 다른 모든 프로세서나 몇개의 보조 프로세서들도 초기화 작업을 수행한 후 `cpu_callout_mask`의 부트스트랩 프로세서 비트를 체크할 수 있습니다. 부트스트랩 프로세서가 이 보조 프로세서로 `cpu_callout_mask`를 채운 후에만 나머지 초기화가 계속됩니다. 특정 프로세서가 초기화 과정을 마치면 해당 프로세서는`cpu_callin_mask`의 비트를 설정합니다. 부트스트랩 프로세서가 현재 보조 프로세서에 해당하는 비트를 `cpu_callin_mask`에서 찾으면 이 프로세서는 나머지 보조 프로세서 중 하나의 초기화를 위해 동일한 절차를 반복합니다. 간단히 설명하여 여태까지 말한대로 작동하지만 좀더 자세한 내용은 `SMP` 챕터에서  살펴보도록 하겠습니다.
        
이것이 끝입니다. 우리는 `SMP` 부팅 준비를 모두 마쳤습니다.

존리스트(zonelist) 구축
-----------------------------------------------------------------------

다음 단계에서는`build_all_zonelists` 함수의 호출을 볼 수 있습니다. 이 함수는 할당이 선호되는대로 구역의 순서를 설정합니다. 구역이란 무엇이며 순서는 무엇인지는 곧 이해하게 될 것입니다. 우선은 리눅스 커널이 물리적 메모리를 어떻게 여기는지 봅시다. 물리적 메모리는 '노드'(`nodes`)라고 불리는 덩어리로 나뉩니다. `NUMA`에 대한 하드웨어 지원이없는 경우 하나의 노드만 보일 것입니다.

```
$ cat /sys/devices/system/node/node0/numastat 
numa_hit 72452442
numa_miss 0
numa_foreign 0
interleave_hit 12925
local_node 72452442
other_node 0
```

모든 `node`는 리눅스 커널에서 `struct pglist_data`로 표시됩니다. 각 노드는 '구역'(`zones`)이라고 불리는 여러 개의 특수한 블록으로 나뉩니다. 모든 구역은 리눅스 커널에서`zone struct`로 표시되며 다음 중 하나의 유형입니다.

* `ZONE_DMA` - 0-16M;
* `ZONE_DMA32` - 4G 미만의 영역(area)만  DMA를 수행 할 수있는 32 비트 장치에 사용됨;
* `ZONE_NORMAL` - `x86_64`에서 4GB부터의 모든 RAM;
* `ZONE_HIGHMEM` - `x86_64`에서 존재하지 않음;
* `ZONE_MOVABLE` - 움직일 수 있는(moveable) 페이지를 포함하는 zone.

이것들은 `zone_type` 열거형으로 표시됩니다. 다음을 통해 zone에 대한 정보를 얻을 수 있습니다.

```
$ cat /proc/zoneinfo
Node 0, zone      DMA
  pages free     3975
        min      3
        low      3
        ...
        ...
Node 0, zone    DMA32
  pages free     694163
        min      875
        low      1093
        ...
        ...
Node 0, zone   Normal
  pages free     2529995
        min      3146
        low      3932
        ...
        ...
```

위에 적힌 것과 같이 모든 노드는 메모리의 `pglist_data` 또는 `pg_data_t` 구조체로 기술됩니다. 이 구조체는 [include / linux / mmzone.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/mmzone.h)에 정의되어 있습니다. [mm/page_alloc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/page_alloc.c)의 `build_all_zonelists` 함수는 정렬된 (각 zone의 `DMA` ,`DMA32`,`NORMAL`,`HIGH_MEMORY`,`MOVABLE`에 대한)`zonelist`를 구성합니다. 이것은 선택된 `zone` 또는 `node`가 할당 요청을 충족시킬 수 없을 때 방문 할 zone/노드를 지정합니다. 이게 전부입니다. `NUMA` 및 멀티프로세서 시스템에 대한 자세한 내용은 특별 파트에 있습니다.

스케줄러 초기화 전의 나머지 것들
--------------------------------------------------------------------------------

리눅스 커널 스케줄러 초기화 프로세스를 시작하기 전에 몇 가지 작업이 필요합니다. 첫 번째는 [mm/page_alloc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/mm/page_alloc.c)의 `page_alloc_init` 함수입니다  이 함수는 꽤 쉽습니다.

```C
void __init page_alloc_init(void)
{
    int ret;
 
    ret = cpuhp_setup_state_nocalls(CPUHP_PAGE_ALLOC_DEAD,
                                    "mm/page_alloc:dead", NULL,
                                    page_alloc_cpu_dead);
    WARN_ON(ret < 0);
}
```

이 함수는 `CPUHP_PAGE_ALLOC_DEAD` cpu [hotplug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt) 상태에 대한`startup` 및 `teardown` 콜백 (두 번째 및 세 번째 매개 변수)을 설정합니다. 물론 이 함수의 구현은 `CONFIG_HOTPLUG_CPU` 커널 설정 옵션에 따라 달라지며 이 옵션을 설정하면 `hotplug` 상태에 따라 시스템의 모든 CPU에 대해 이러한 콜백이 설정됩니다. [hotplug](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt) 메커니즘은 너무 큰 주제이므로이 책에서는 설명하지 않겠습니다.

이 함수 후에 초기화 출력에서 커널 명령 행을 볼 수 있습니다.

![kernel command line](http://oi58.tinypic.com/2m7vz10.jpg)

그리고 리눅스 커널 커맨드 라인을 처리하는 `parse_early_param` 및 `parse_args`와 같은 몇 가지 기능이 있습니다. 우리는 이미 커널 초기화 챕터의 여섯 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-6.html)에서 `parse_early_param` 함수의 호출을 보았을 것입니다. 그런데 왜 다시 호출할까요? 답은 간단합니다:  (우리의 경우`x86_64`) 특정 아키텍처를 위한 (architecture-specific) 코드에서 이 함수를 호출했어도 모든 아키텍처가 이 함수를 호출하는 것은 아니기 때문입니다. 비-초기(non-early) 명령 줄 인수(argument)들을 분석(parse)하고 처리하려면 두 번째 함수 인 `parse_args`를 호출해야합니다.

다음 단계에서 [kernel / jump_label.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/jump_label.c) 내의 `jump_label_init`의 호출을 볼 수 있습니다. [jump label](https://lwn.net/Articles/412072/)을 초기화합니다.

그 후에 우리는 [printk](http://www.makelinux.net/books/lkd2/ch18lev1sec3) 로그 버퍼를 설정하는`setup_log_buf` 함수의 호출을 볼 수 있습니다. 우리는 이미 이 함수를 리눅스 커널 초기화 프로세스 챕터의 일곱 번째 [파트](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-7.html)에서 보았습니다.

PID 해시 초기화
--------------------------------------------------------------------------------

다음은`pidhash_init` 함수입니다. 아시다시피 각 프로세스에는 '프로세스 식별 번호'(`process identification number`) 또는 `PID`라고 불리는 고유 번호가 할당되어 있습니다. 포크 또는 클론으로 생성 된 각 프로세스에는 커널에 의해 새로운 고유  `PID`값이 자동으로 할당됩니다. `PID`의 관리는 `struct pid`와`struct upid`라는 두 가지의 특별한 자료구조를 중심으로 이루집니다. 첫 번째 자료구조는 커널의 `PID`에 대한 정보를 나타냅니다. 두 번째 자료구조는 특정 네임 스페이스에서 볼 수있는 정보를 나타냅니다. 특수 해시 테이블에 저장된 모든`PID` 인스턴스는:

```C
static struct hlist_head *pid_hash;
```

이 해시 테이블은 `PID` 숫자값에 해당하는 pid 인스턴스를 찾는 데 사용됩니다. 따라서 `pidhash_init`는이 해시 테이블을 초기화합니다. `pidhash_init` 함수의 시작에서 우리는`alloc_large_system_hash`의 호출을 볼 수 있습니다 :

```C
pid_hash = alloc_large_system_hash("PID", sizeof(*pid_hash), 0, 18,
                                   HASH_EARLY | HASH_SMALL,
                                   &pidhash_shift, NULL,
                                   0, 4096);
```

`pid_hash`의 요소 수는`RAM` 구성에 따라 다르지만`2 ^ 4`와`2 ^ 12` 사이입니다. `pidhash_init`는 크기를 계산하고 필요한 저장공간(이 경우`hlist`임 - [doubly linked list](https://0xax.gitbooks.io/linux-insides/content/DataStructures/linux-datastructures-1.html)와 동일함)을 할당하지만 대신 [struct hlist_head](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/types.h)에 포인터를 하나 포함합니다. `alloc_large_system_hash` 함수는 만약 우리가 `HASH_EARLY` 플래그를 전달한다면  `meakblock_virt_alloc_nopanic`와 함께 큰 시스템 해시 테이블(large system hash table)을 할당하고, 플래그를 전달하지 않으면 `__vmalloc`과 함께 할당합니다.

그 결과는`dmesg` 출력에서 확인할 수 있습니다 :

```
$ dmesg | grep hash
[    0.000000] PID hash table entries: 4096 (order: 3, 32768 bytes)
...
...
...
```

이게 다입니다. 스케줄러 초기화 전 나머지 기능은 다음과 같습니다. `vfs_caches_init_early`는 가상 파일 시스템([virtual file system](http://en.wikipedia.org/wiki/Virtual_file_system))의 초기 초기화를 수행하고 (자세한 내용은 가상 파일 시스템을 설명하는 챕터에 있습니다), `sort_main_extable`은 커널의 내장 예외 테이블의 `__start ___ ex_table`과`__stop ___ ex_table` 사이의 엔트리들을 정렬하며, `trap_init`는 트랩 핸들러를 초기화합니다 (뒤쪽 두 함수에 대한 자세한 내용은 인터럽트에 대한 챕터에서 설명합니다).

스케줄러 초기화 전의 마지막 단계는 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)의 `mm_init` 함수를 사용하여 메모리 관리자를 초기화하는 것입니다. 보시다시피 `mm_init` 함수는 리눅스 커널 메모리 관리자의 다른 부분을 초기화합니다 :

```C
page_ext_init_flatmem();
mem_init();
kmem_cache_init();
percpu_init_late();
pgtable_init();
vmalloc_init();
```

첫 번째 함수는 `page_ext_init_flatmem`이며 `CONFIG_SPARSEMEM` 커널 구성 옵션에 따라 매 페이지 처리 마다 확장 데이터를 초기화합니다. `mem_init`는 모든 `bootmem`을 해제하고, `kmem_cache_init`는 커널 캐시를 초기화합니다. `percpu_init_late`는 `percpu` 청크를 [slub](http://en.wikipedia.org/wiki/SLUB%28software%29)으로 바꾸고, `pgtable_init`은 `page-> ptl` 커널 캐시를 초기화하고, `vmalloc_init`는 `vmalloc`을 초기화합니다. **참고:** 지금은 이러한 함수들과 개념에 대한 자세한 내용은 다루지 않을 것이지만 이후 [Linux 커널 메모리 관리자](https://0xax.gitbooks.io/linux-insides/content/MM/index.html) 챕터에서 다루게 될 예정입니다.

이것으로 마무리가 되었으며, 이제 우리는 스케줄러(`scheduler`)를 보러 갈 수 있습니다.

스케줄러 초기화
--------------------------------------------------------------------------------

이제 우리는 이번 파트의 주 목적, 즉 작업 스케줄러 초기화에 도달했습니다.이미 여러 번 했어도 다시 말씀드리는 것이지만, 여기서는 스케줄러에 대한 전부를 설명하지 않습니다. 이에 대해서는 별도의 챕터에서 다룰 것입니다. 여기에선 가장 먼저 초기화되는 첫 번째 스케줄러 초기화 메커니즘에 대하여 설명할 것입니다. 그럼 시작해봅시다.

현재 우리는 [kernel/sched/core.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sched/core.c) 커널 소스 코드 파일의`sched_init` 함수에 있습니다. 그리고 함수 이름에서 알 수 있듯이 이 함수는 스케줄러를 초기화합니다. 이 함수를 파헤치고 스케줄러가 초기화되는 방법을 이해해 봅시다. `sched_init` 함수의 시작 부분에서 우리는 다음 호출을 볼 수 있습니다 :

```C
sched_clock_init();
```

`sched_clock_init`는 꽤나 쉬운 함수이며 보이는 것과 같이 단순히 `sched_clock_init` 변수를 설정할 뿐입니다.

```C
void sched_clock_init(void)
{
	sched_clock_running = 1;
}
```

저것은 나중에 사용될 것입니다. 그 다음 단계에서는 '대기열'(`waitqueues`)배열을 초기화합니다:

```C
for (i = 0; i < WAIT_TABLE_SIZE; i++)
	init_waitqueue_head(bit_wait_table + i);
```

여기서 `bit_wait_table`은 다음과 같이 정의됩니다:

```C
#define WAIT_TABLE_BITS 8
#define WAIT_TABLE_SIZE (1 << WAIT_TABLE_BITS)
static wait_queue_head_t bit_wait_table[WAIT_TABLE_SIZE] __cacheline_aligned;
```

`bit_wait_table`은 지정된(해당하는) 비트의 값에 의존하는 프로세스의 대기/활성화(wait/wake up)를 위해 사용될 대기 큐의 배열입니다. `waitqueues`배열 초기화 후 다음 단계는`root_task_group`에 할당 할 메모리 크기를 계산하는 것입니다. 아래 보이듯 그 크기는 다음 두 가지 커널 구성 옵션에 영향을 받습니다.

```C
#ifdef CONFIG_FAIR_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
#ifdef CONFIG_RT_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
```

* `CONFIG_FAIR_GROUP_SCHED`;
* `CONFIG_RT_GROUP_SCHED`.

이 두 옵션은 서로 다른 두 가지 계획(planning) 모델을 제공합니다. [문서](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)에서 읽을 수 있듯이 현재 스케줄러-`CFS` 혹은 `Completely Fair Scheduler`가 사용하는 개념은 간단합니다. 스케줄러는 각 프로세스가 `1/n` 프로세서 시간(processor time)을 나눠받는 이상적인 멀티 태스킹 프로세서가 시스템에 있는 것처럼 프로세스 스케줄링을 모델링합니다. 여기서`n`은 실행 가능한 프로세스의 수입니다.스케줄러는 특수한 규칙 세트를 사용합니다. 이 규칙은 실행할 새 프로세스를 언제 어떻게 선택할 지 결정하며 이를 '스케쥴링 정책'(`scheduling policy`)이라고 합니다.

'완전히 공정한 스케줄러'(`Completely Fair Scheduler`)는 '정상적인'(`normal`) 또는 다른 말로 '비 실시간'(`non-real-time`) 스케줄링 정책을 지원합니다.

* `SCHED_NORMAL`;
* `SCHED_BATCH`;
* `SCHED_IDLE`.

`SCHED_NORMAL`은 대부분의 일반적인 응용 프로그램에 사용되며 각 프로세스가 소비하는 CPU의 양은 대부분 [nice](http://en.wikipedia.org/wiki/Nice_%28Unix%29) 값에 의해 결정됩니다. `SCHED_BATCH`는 100 % 비 대화식(non-interactive) 작업에 사용되고, `SCHED_IDLE`은 프로세서가 이 작업 외에 실행할 다른 작업이 없는 경우에만 작업을 실행하는 데에 사용됩니다.

'실시간'(`realtime`) 정책은 시간이 중요한 응용 프로그램(`SCHED_FIFO` 및 `SCHED_RR`)에 의해서도 지원됩니다. 리눅스 커널 스케줄러에 대해 읽은 것이 좀 있으신 분들은 이것이 모듈 식임을 알아차렸을 것입니다. 이것은 즉, 다른 유형의 프로세스를 스케쥴링하기 위해 서로 다른 알고리즘들을 지원한다는 뜻입니다. 일반적으로 이 모듈성을 '스케줄러 클래스'(`scheduler classes`)라고 합니다. 이 모듈들은 스케줄링 정책의 세부 사항을 캡슐화하며 스케줄러 코어가 그것들에 대해 아주 많이 알고 있지 않아도 처리할 수 있게 합니다.

이제 코드로 돌아가서 `CONFIG_FAIR_GROUP_SCHED`와 `CONFIG_RT_GROUP_SCHED`의 두 가지 구성 옵션을 살펴 보겠습니다. 스케줄러가 작동하는 최소 단위는 개별 작업(task) 또는 스레드입니다. 그러나 프로세스는 그저 스케줄러가 작동하는 엔티티중 한 종류가 아닙니다. 저 두 옵션 모두 그룹 스케줄링을 지원합니다. 첫 번째 옵션은 '완전히 공정한 스케줄러'(`completely fair scheduler`)정책으로 그룹 스케줄링을 지원하고 두 번째 옵션은 각각 '실시간'(`realtime`)정책으로 지원합니다.

간단히 말해서, 그룹 스케줄링은 일련의 작업을  단일 작업처럼 스케줄링 할 수있는 기능입니다. 예를 들어 두 개의 작업을 묶어 그룹을 만들면, 이 그룹은 커널 관점에서 보면 하나의 일반적인 작업과 같습니다. 그룹이 스케줄링 된 후 스케줄러는 이 그룹 내에서 작업을 선택하고 스케줄링합니다. 따라서 이러한 메커니즘을 통해 계층(hierarchies)을 구축하고 리소스를 관리 할 수 있습니다. 스케줄링의 최소 단위는 프로세스임에도, Linux 커널 스케줄러는 내부적으로 `task_struct` 구조를 사용하지 않습니다. 리눅스 커널 스케줄러가 스케줄링 단위로 사용하는 특별한 `sched_entity` 구조가 있습니다.

따라서 현재 목표는 루트 작업 그룹의 `sched_entity (ies)`에 할당 할 공간을 계산하여 다음과 같이 두 번 수행하는 것입니다.

```C
#ifdef CONFIG_FAIR_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
#ifdef CONFIG_RT_GROUP_SCHED
         alloc_size += 2 * nr_cpu_ids * sizeof(void **);
#endif
```

첫 번째는 '완전히 공정한'(`completely fair`) 스케줄러로 작업 그룹의 스케줄링이 활성화 된 경우이고, 두 번째는 '실시간'(`realtime`) 스케줄러로 작업 그룹의 스케줄링이 활성화된 경우를 위한 것입니다. 그래서 여기서는 크기를 계산하는데, 이 크기는 포인터의 크기에 CPU 갯수를 곱하고 `2`를 곱한 것과 같습니다. 2를 곱하는 것은 우리가 두가지를 저장할 공간을 할당해야 하기 때문입니다.

* 스케줄러 엔티티 구조(scheduler entity structure);
* `runqueue`.

크기를 계산 한 후에는 `kzalloc` 함수로 공간을 할당하고 `sched_entity`와 `runquques`에 포인터를 설정합니다.

```C
ptr = (unsigned long)kzalloc(alloc_size, GFP_NOWAIT);
 
#ifdef CONFIG_FAIR_GROUP_SCHED
        root_task_group.se = (struct sched_entity **)ptr;
        ptr += nr_cpu_ids * sizeof(void **);

        root_task_group.cfs_rq = (struct cfs_rq **)ptr;
        ptr += nr_cpu_ids * sizeof(void **);
#endif
#ifdef CONFIG_RT_GROUP_SCHED
		root_task_group.rt_se = (struct sched_rt_entity **)ptr;
		ptr += nr_cpu_ids * sizeof(void **);

		root_task_group.rt_rq = (struct rt_rq **)ptr;
		ptr += nr_cpu_ids * sizeof(void **);

#endif
```

이미 언급했듯이, 리눅스 그룹 스케줄링 메커니즘을 사용하면 계층 구조를 지정할 수 있습니다. 이러한 계층의 루트는 `root_runqueuetask_group` 작업 그룹 구조체입니다. 이 구조체에는 많은 필드가 포함되어 있지만, 이 시점에서 우리가 관심 있는 것은 `se`,`rt_se`,`cfs_rq` 및`rt_rq`입니다.

처음 두 가지는`sched_entity` 구조체의 인스턴스입니다. 이것은 [include/linux/sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/sched.h) 커널 헤더 파일에 정의되어 있으며 스케줄링의 단위로 스케줄러에 의해 사용됩니다.

```C
struct task_group {
    ...
    ...
    struct sched_entity **se;
    struct cfs_rq **cfs_rq;
    ...
    ...
}
```

`cfs_rq`와`rt_rq`는`run queues`를 나타냅니다. `run queue` 는 리눅스 커널 스케줄러가 `active` 스레드를 저장하기 위해 사용하는 특별한 `per-cpu` 구조체이며 다른 말로 하자면 잠재적으로 스케줄러에 의해 픽업되어 실행될 수 있는 스레드들의 집합입니다. .

 공간이 할당되고 다음 단계는 '실시간' (`realtime`) 및 '데드라인'(`deadline`) 작업을 위한 CPU의 '대역폭'(`bandwidth`)을 초기화하는 것입니다. 

```C
init_rt_bandwidth(&def_rt_bandwidth,
                  global_rt_period(), global_rt_runtime());
init_dl_bandwidth(&def_dl_bandwidth,
                  global_rt_period(), global_rt_runtime());
```

모든 그룹은 일정량의 CPU 시간(CPU time)을 사용할 수 있어야 합니다.  이하 두 구조체: `def_rt_bandwidth`와 `def_dl_bandwidth`의 두 구조체는 `realtime` 및 `deadline` 작업에 대한 대역폭의 기본값을 나타냅니다. 현재로서는 그다지 중요하지 않기 때문에 이러한 구조체의 정의는 고려하지 않을 것이며, 우리가 관심 있는 것은 다음 두 값입니다: 

* `sched_rt_period_us`;
* `sched_rt_runtime_us`.

 첫 번째는 구간(기간)을 나타내고 두 번째는 `sched_rt_period_us` 도중 `realtime` 작업에 할당된 양(quantum)입니다. 다음에서 이러한 매개 변수들의 전역 값을 볼 수 있습니다.

```
$ cat /proc/sys/kernel/sched_rt_period_us 
1000000

$ cat /proc/sys/kernel/sched_rt_runtime_us 
950000
```

그룹과 관련된 값은`<cgroup>/cpu.rt_period_us` 및`<cgroup>/cpu.rt_runtime_us`에서 구성 할 수 있습니다. 파일 시스템이 아직 마운트되지 않았기 때문에 `def_rt_bandwidth` 및`def_dl_bandwidth`는 `global_rt_period` 및 `global_rt_runtime` 함수가 리턴하는 기본값으로 초기화됩니다.

`realtime`과 `deadline` 작업의 대역폭, 이것으로 끝이며 그 다음 단계에서는 [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)의 활성화 여부에 따라 `루트 도메인`의 초기화를 진행합니다:

```C
#ifdef CONFIG_SMP
	init_defrootdomain();
#endif
```

실시간 스케줄러는 스케줄링 결정을 위해 전역 자원(global resources)이 필요합니다. 그러나 안타깝게도 CPU 수가 증가함에 따라 확장성 병목 현상(scalability bottlenecks)이 나타납니다. '루트 도메인'(`root domains`)이라는 개념은 확장성을 개선하고 이러한 병목 현상을 피하기 위해 도입되었습니다. 스케줄러는 모든`run queues`를 우회하는 대신, `root_domain` 구조체에서 `realtime` 작업(task)을 어느 CPU로 푸시/풀 할 것인지에 대한 정보를 얻습니다. 이 구조체는 [kernel/sched/sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sched/sched.h) 커널 헤더 파일에 정의되어 있으며 프로세스를 푸시하거나 풀할 수 있는 CPU만 추적합니다. 

'루트 도메인'(`root domain`) 초기화 후 위에서 했던 것과 동일한 기본값으로 '루트 태스크 그룹'(`root task group`) 의 `realtime` 작업에 대한 '대역폭'(`bandwidth`)을 초기화합니다.
```C
#ifdef CONFIG_RT_GROUP_SCHED
	init_rt_bandwidth(&root_task_group.rt_bandwidth,
			global_rt_period(), global_rt_runtime());
#endif
```

다음 단계에서는, `CONFIG_CGROUP_SCHED` 커널 구성 옵션에 따라 작업 그룹(들)(`task_group(s)`)에 `slab` 캐시를 할당하고 루트 작업 그룹의 `siblings` 및 `children` 리스트를 초기화합니다. 문서에서 읽을 수 있듯이, `CONFIG_CGROUP_SCHED`는 다음과 같습니다.

```
This option allows you to create arbitrary task groups using the "cgroup" pseudo
filesystem and control the cpu bandwidth allocated to each such task group.
(이하 번역)
이 옵션을 사용하면 "cgroup" 의사 파일 시스템을 사용하는 임의의 작업 그룹을 만들 수 있으며, 이러한 각 작업 그룹에 할당된 CPU 대역폭을 제어할 수 있습니다.
```

리스트 초기화를 마치면, `autogroup_init` 함수의 호출을 볼 수 있습니다 :

```C
#ifdef CONFIG_CGROUP_SCHED
         list_add(&root_task_group.list, &task_groups);
         INIT_LIST_HEAD(&root_task_group.children);
         INIT_LIST_HEAD(&root_task_group.siblings);
         autogroup_init(&init_task);
#endif
```

이 함수는 자동 프로세스 그룹 스케줄링을 초기화합니다. `autogroup` 기능은 [setsid](https://linux.die.net/man/2/setsid) 호출을 통해 새 세션을 생성하는 동안 새 작업 그룹을 자동으로 생성하고 채우는 것에 관한 것입니다.

그런 다음, 모든 '가능한'(`possible`) CPU (시스템에서 사용 가능한 `possible` CPU들은 `cpu_possible_mask` 비트 맵에 저장되어 있음)를 돌며 각 `possible` CPU에 대해 `runqueue`를 초기화합니다 :

```C
for_each_possible_cpu(i) {
    struct rq *rq;
    ...
    ...
    ...
```

리눅스 커널의 `rq` 구조체는 [kernel/sched/sched.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/sched/sched.h#L625)에 정의되어 있습니다. 위에서 이미 언급했듯이, `run queue`는 프로세스 스케줄링에서의 기본적인 데이터 구조입니다. 스케줄러는 이걸 사용하여 다음에 실행할 프로세스를 결정합니다. 보시다시피, 이 구조체에는 여러 가지 구성요소(field)가 있지만 여기에서 모두 다루지는 않고, 각각을 직접 사용할 때에 살펴볼 것입니다.

CPU 별(`per-cpu`) 실행 큐를 기본값으로 초기화 한 후에는, 시스템에서 첫 번째 작업의 '로드 가중치'(`load weight`)를 설정해야합니다.

```C
set_load_weight(&init_task);
```

우선 프로세스의 '로드 가중치'(`load weight`)가 무엇인지 이해해봅시다. `sched_entity` 구조체의 정의를 살펴보면 `load` 필드로 시작한다는 것을 알 수 있습니다:

```C
struct sched_entity {
	struct load_weight		load;
    ...
    ...
    ...
}
```

`load` 필드는 스케줄러 엔티티의 실제 로드 가중치와 그 불변 값을 나타내는 두 개의 필드만으로 구성된 `load_weight` 구조체로 표현됩니다.

```C
struct load_weight {
	unsigned long	weight;
	u32				inv_weight;
};
```

시스템의 각 프로세스에는 '우선 순위'가 있다는 것을 이미 알고있을 것입니다. 우선 순위가 높을수록 더 많은 실행 시간을 확보할 수 있습니다. 프로세스의 '로드 가중치'(`load weight`)는 그 프로세스의 우선 순위와 타임 슬라이스(timeslices, 작업의 CPU 시간이 한도를 넘으면 강제로 다른 작업으로 전환하는 방식) 사이의 관계입니다. 각 프로세스에는 우선 순위와 관련하여 아래 세 가지 필드를 가집니다.

```C
struct task_struct {
...
...
...
	int				prio;
	int				static_prio;
	int				normal_prio;
...
...
...
}
```

첫 번째는 프로세스의 정적 우선 순위와 프로세스의 상호 작용성에 따라 프로세스 수명 동안 변경할 수없는 '동적 우선 순위'(`dynamic priority`)입니다. `static_prio`는 우리가 `nice value`로 잘 알고 있는 초기 우선 순위(initail priority)를 포함합니다. 이 값은 사용자가 변경하지 않으면 커널이 따로 변경하거나 하지 않습니다. 마지막은 `normal_priority`이며, `static_prio`의 값을 베이스로 하지만, 프로세스의 스케줄링 정책에도 영향을 받습니다.

그래서 `set_load_weight` 함수의 주요한 목적은 `init` 작업을 위해 `load_weights` 필드를 초기화하는 것입니다:

```C
static void set_load_weight(struct task_struct *p)
{
	int prio = p->static_prio - MAX_RT_PRIO;
	struct load_weight *load = &p->se.load;

	if (idle_policy(p->policy)) {
		load->weight = scale_load(WEIGHT_IDLEPRIO);
		load->inv_weight = WMULT_IDLEPRIO;
		return;
	}

	load->weight = scale_load(sched_prio_to_weight[prio]);
	load->inv_weight = sched_prio_to_wmult[prio];
}
```

보시다시피, 우리는 `init` 작업의 `static_prio`의 초기 값에서 초기 `prio`를 계산하고, 이것을 `sched_prio_to_weight` 및 `sched_prio_to_wmult` 배열의 인덱스로 사용하여 `weight` 및`inv_weight` 값을 설정합니다. 이 두 배열은 우선 순위 값에 따라 `load weight`를 포함합니다. 프로세스가 `idle` 프로세스 인 경우 로드 가중치를 최소로 설정합니다.

이 시점에서 우리는 리눅스 커널 스케줄러 초기화 프로세스의 끝부분에 도달했습니다. 마지막 단계는- : CPU가 실행할 다른 프로세스가 없을 때 실행되게 현재 프로세스(아마 첫 번째 `init` 프로세스일 것입니다)를 `idle`로 만드는 것입니다. CPU load의 다음 계산 및 `fair` 클래스 초기화의 다음 기간 계산은 다음과 같습니다.

```C
__init void init_sched_fair_class(void)
{
#ifdef CONFIG_SMP
	open_softirq(SCHED_SOFTIRQ, run_rebalance_domains);
#endif
}
```

여기서 우리는 `run_rebalance_domains` 처리기(handler)를 호출할 [soft irq](https://0xax.gitbooks.io/linux-insides/content/Interrupts/linux-interrupts-9.html)를 등록합니다. `SCHED_SOFTIRQ`가 트리거 된 후, `run_rebalance`가 현재 CPU에서 실행 큐를 재조정하기 위해 호출됩니다.

`sched_init` 함수의 마지막 두 단계는 스케줄러 통계를 초기화하고 `scheduler_running` 변수를 설정하는 것입니다.

```C
scheduler_running = 1;
```

이것으로 리눅스 커널 스케줄러가 초기화되었습니다. 비록 리눅스 커널에서 프로세스와 프로세스 그룹, runqueue, rcu 등 서로 다른 개념들이 어떻게 작동하는지 알고 이해해야 하기 때문에 여기서는 많은 다른 디테일과 설명을 건너 뛰었지만, 우리는 스케줄러 초기화 과정을 간략히 살펴 보았습니다. 우리는 스케줄러만을 다루는 별도의 파트에서 다루지 못한 모든 세부 사항들을 살펴볼 것입니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널 초기화 과정에 대한 여덟 번째 부분은 끝입니다. 이 파트에서는 스케줄러의 초기화 프로세스를 살펴 보았고 다음 부분에서는 계속해서 리눅스 커널 초기화 프로세스를 살펴보고 [RCU](http://en.wikipedia.org/wiki/Read-copy-update)의 초기화 및 다른 많은 초기화 요소들을 보게 될 것입니다.

만약 질문이나 의견이 있으시다면, [트위터](https://twitter.com/0xAX)에서 저를 핑해주시거나, 코멘트를 달아주세요.

**영어는 제 모국어가 아닙니다, 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수들을 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한국 번역](https://github.com/junsooo/linux-insides-ko)로 PR을 보내주세요.**

참고 링크
--------------------------------------------------------------------------------

* [CPU masks](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-2.html)
* [high-resolution kernel timer](https://www.kernel.org/doc/Documentation/timers/hrtimers.txt)
* [spinlock](http://en.wikipedia.org/wiki/Spinlock)
* [Run queue](http://en.wikipedia.org/wiki/Run_queue)
* [Linux kernel memory manager](https://0xax.gitbooks.io/linux-insides/content/MM/index.html)
* [slub](http://en.wikipedia.org/wiki/SLUB_%28software%29)
* [virtual file system](http://en.wikipedia.org/wiki/Virtual_file_system)
* [Linux kernel hotplug documentation](https://www.kernel.org/doc/Documentation/cpu-hotplug.txt)
* [IRQ](http://en.wikipedia.org/wiki/Interrupt_request_%28PC_architecture%29)
* [Global Descriptor Table](http://en.wikipedia.org/wiki/Global_Descriptor_Table)
* [Per-CPU variables](https://0xax.gitbooks.io/linux-insides/content/Concepts/linux-cpu-1.html)
* [SMP](http://en.wikipedia.org/wiki/Symmetric_multiprocessing)
* [RCU](http://en.wikipedia.org/wiki/Read-copy-update)
* [CFS Scheduler documentation](https://www.kernel.org/doc/Documentation/scheduler/sched-design-CFS.txt)
* [Real-Time group scheduling](https://www.kernel.org/doc/Documentation/scheduler/sched-rt-group.txt)
* [Previous part](https://0xax.gitbooks.io/linux-insides/content/Initialization/linux-initialization-7.html)
