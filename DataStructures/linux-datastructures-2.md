리눅스 커널의 데이터 구조
================================================================================

기수 트리
--------------------------------------------------------------------------------

이미 알고 있듯이 리눅스 커널은 다양한 데이터 구조와 알고리즘을 구현하는 다양한 라이브러리와 함수를 제공합니다. 이 부분에서는 다음 데이터 구조 중 하나 인 [기수 트리](http://en.wikipedia.org/wiki/Radix_tree)를 고려할 것입니다. 리눅스 커널의 `기수 트리` 구현과 API와 관련된 두 개의 파일이 있습니다 :

* [include/linux/radix-tree.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/radix-tree.h)
* [lib/radix-tree.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/lib/radix-tree.c)

`기수 트리`가 무엇인지 이야기 해 봅시다. Radix 트리는 압축 된 트리입니다. [trie](http://en.wikipedia.org/wiki/Trie)는 연관 배열의 인터페이스를 구현하고 값을 `키-밸류` 형태로 저장할 수있는 데이터 구조입니다. 키는 일반적으로 문자열이지만 모든 데이터 유형을 사용할 수 있습니다. 트리는 노드 때문에 n-트리와 다릅니다. 트리의 노드는 키를 저장하지 않습니다. 대신 트리의 노드는 단일 문자 레이블을 저장합니다. 주어진 노드와 관련된 키는 트리의 루트에서 이 노드로 순회하여 파생됩니다. 예를 들면 다음과 같습니다.


```
               +-----------+
               |           |
               |    " "    |
               |           |
        +------+-----------+------+
        |                         |
        |                         |
   +----v------+            +-----v-----+
   |           |            |           |
   |    g      |            |     c     |
   |           |            |           |
   +-----------+            +-----------+
        |                         |
        |                         |
   +----v------+            +-----v-----+
   |           |            |           |
   |    o      |            |     a     |
   |           |            |           |
   +-----------+            +-----------+
                                  |
                                  |
                            +-----v-----+
                            |           |
                            |     t     |
                            |           |
                            +-----------+
```

따라서이 예에서는 키가있는`trie`, `go` 및 `cat`을 볼 수 있습니다. 압축 된 trie 또는`radix tree`는 자식이 하나만있는 모든 중간 노드가 제거된다는 점에서 `trie`와 다릅니다.

리눅스 커널의 기수 트리는 값을 정수 키에 매핑하는 데이터 구조입니다. 파일 [include/linux/radix-tree.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/radix-tree.h)에서 다음 구조체로 표시됩니다.

```C
struct radix_tree_root {
         unsigned int            height;
         gfp_t                   gfp_mask;
         struct radix_tree_node  __rcu *rnode;
};
```

이 구조체는 기수의 루트를 나타내며 세 개의 필드를 포함합니다.

* `height`   - 트리의 높이를 나타냄;
* `gfp_mask` - 메모리 할당 수행 방법을 알려줌;
* `rnode`    - 하위 노드로의 포인터.

우리가 논의 할 첫 번째 필드는 `gfp_mask`입니다.

저수준 커널 메모리 할당 함수는 플래그 집합을`gfp_mask`로 가져 와서 할당 방법을 설명합니다. 할당 프로세스를 제어하는 이러한 `GFP_` 플래그는 다음 값을 가질 수 있습니다. 프로세스는 우선 순위가 높고 슬립 할 수 없습니다.

* `GFP_NOIO` - 메모리에서 슬립과 대기를 할 수 없음;
* `__GFP_HIGHMEM` - 높은 메모리 사용 가능;
* `GFP_ATOMIC` - 할당 프로세스는 우선 순위가 높으며 슬립할 수 없음;

etc.

다음 필드는`rnode`입니다 :

```C
struct radix_tree_node {
        unsigned int    path;
        unsigned int    count;
        union {
                struct {
                        struct radix_tree_node *parent;
                        void *private_data;
                };
                struct rcu_head rcu_head;
        };
        /* For tree user */
        struct list_head private_list;
        void __rcu      *slots[RADIX_TREE_MAP_SIZE];
        unsigned long   tags[RADIX_TREE_MAX_TAGS][RADIX_TREE_TAG_LONGS];
};
```

이 구조체에는 부모의 오프셋 및 자식으로부터의 높이, 자식 노드 수 와 노드의 액세스 및 해제를 위한 필드에 대한 정보가 포함됩니다. 이 필드는 아래에 설명되어 있습니다.

* `path` - 바닥에서 부모 및 높이의 오프셋;
* `count` - 자식 노드의 개수
* `parent` - 부모 노드의 포인터
* `private_data` - 트리의 사용자가 사용함
* `rcu_head` - 트리의 노드를 해제하는데 사용
* `private_list` - 트리의 사용자가 사용함

`radix_tree_node` - `tags` 및 `slots` 의 마지막 두 필드는 중요하고 흥미롭습니다. 모든 노드는 데이터에 대한 포인터를 저장하는 슬롯 세트를 포함 할 수 있습니다. 리눅스 커널 기수 트리 구현 저장소 `NULL`에 빈 슬롯이 있습니다. 리눅스 커널의 기수 트리는 `radix_tree_node` 구조의 `tags` 필드와 관련된 태그도 지원합니다. 태그를 사용하면 기수 트리에 저장된 레코드에 개별 비트를 설정할 수 있습니다.

기수 트리 구조체에 대해 알았으므로 이제 API를 살펴볼 차례입니다.

리눅스 커널 기수 트리 API
---------------------------------------------------------------------------------

데이터 구조의 초기화부터 시작합니다. 새 기수 트리를 초기화하는 두 가지 방법이 있습니다. 첫 번째는 `RADIX_TREE` 매크로를 사용하는 것입니다 :

```C
RADIX_TREE(name, gfp_mask);
````

보시다시피 `name` 매개 변수를 전달하므로 `RADIX_TREE` 매크로를 사용하면 주어진 이름으로 기수 트리를 정의하고 초기화 할 수 있습니다. `RADIX_TREE`의 구현은 쉽습니다 :

```C
#define RADIX_TREE(name, mask) \
         struct radix_tree_root name = RADIX_TREE_INIT(mask)

#define RADIX_TREE_INIT(mask)   { \
        .height = 0,              \
        .gfp_mask = (mask),       \
        .rnode = NULL,            \
}
```

`RADIX_TREE` 매크로의 시작에서 우리는 주어진 이름으로 `radix_tree_root` 구조의 인스턴스를 정의하고 주어진 마스크로 `RADIX_TREE_INIT` 매크로를 호출합니다. `RADIX_TREE_INIT` 매크로는 기본값과 주어진 마스크로 `radix_tree_root` 구조를 초기화합니다.

두 번째 방법은 `radix_tree_root` 구조를 손으로 정의하고 마스크와 함께 `INIT_RADIX_TREE` 매크로에 전달하는 것입니다.

```C
struct radix_tree_root my_radix_tree;
INIT_RADIX_TREE(my_tree, gfp_mask_for_my_radix_tree);
```

어디에:

```C
#define INIT_RADIX_TREE(root, mask)  \
do {                                 \
        (root)->height = 0;          \
        (root)->gfp_mask = (mask);   \
        (root)->rnode = NULL;        \
} while (0)
```

`RADIX_TREE_INIT` 매크로와 동일한 기본값으로 초기화합니다.

다음은 기수 트리에서 레코드를 삽입하고 삭제하는 두 가지 기능입니다.

* `radix_tree_insert`;
* `radix_tree_delete`;

첫 번째`radix_tree_insert` 함수는 세 가지 파라미터를 취합니다 :

* 기수 트리의 루트
* 인덱스 키
* 삽입할 데이터

`radix_tree_delete` 함수는`radix_tree_insert`와 동일한 매개 변수 세트를 취하지 만 데이터는 없습니다.

기수 트리에서의 검색은 두 가지 방법으로 구현됩니다.

* `radix_tree_lookup`;
* `radix_tree_gang_lookup`;
* `radix_tree_lookup_slot`.

첫 번째 `radix_tree_lookup` 함수는 두 가지 파라미터를 취합니다 :

* 기수 트리의 루트
* 인덱스 키

이 함수는 트리에서 지정된 키를 찾고이 키와 관련된 레코드를 반환합니다. 두 번째 `radix_tree_gang_lookup` 함수에는 다음과 같은 시그니처가 있습니다
```C
unsigned int radix_tree_gang_lookup(struct radix_tree_root *root,
                                    void **results,
                                    unsigned long first_index,
                                    unsigned int max_items);
```

그리고 첫 번째 인덱스부터 키별로 정렬 된 레코드 수를 반환합니다. 반환 된 레코드의 수는 `max_items` 값보다 크지 않습니다.

마지막 `radix_tree_lookup_slot` 함수는 데이터를 포함 할 슬롯을 반환합니다.

링크
---------------------------------------------------------------------------------

* [Radix tree](http://en.wikipedia.org/wiki/Radix_tree)
* [Trie](http://en.wikipedia.org/wiki/Trie)
