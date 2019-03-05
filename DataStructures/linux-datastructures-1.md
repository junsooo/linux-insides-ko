리눅스 커널의 여러 데이터 구조
================================================================================

더블 링크드 리스트
--------------------------------------------------------------------------------

리눅스 커널은 자체적으로 더블 링크드 리스트에 대한 구현 코드를 제공합니다. 해당 코드는 [include/linux/list.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/list.h)에서 찾아볼 수 있습니다. 이제 `리눅스 커널의 데이터 구조들`에 대한 공부를 더블 링크드 리스트 데이터 구조에서 시작해보죠. 이유는 더블 링크드 리스트가 커널상에서 매우 유명하기 때문입니다. 여기서 [검색](http://lxr.free-electrons.com/ident?i=list_head)해보세요.

첫 번째로, [include/linux/types.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/types.h)에서 주요 구조에 대해 살펴봅시다.

```C
struct list_head {
	struct list_head *next, *prev;
};
```

여태까지 보았던 수많은 더블 링크드 리스트의 구현과는 다르다는 것을 확인할 수 있습니다. 예를 들어, [glib](http://www.gnu.org/software/libc/)의 더블 링크드 리스트 구조는 이렇게 생겼습니다.

```C
struct GList {
  gpointer data;
  GList *next;
  GList *prev;
};
```

일반적으로 링크드 리스트 구조는 각 항목에 대한 포인터를 포함합니다. 하지만 리눅스 커널의 링크드 리스트 구현은 이를 포함하지 않는군요. 따라서 주요 질문은 다음과 같습니다. `과연 리스트는 어디에 실제로 데이터를 저장할까요?` 커널에서의 링크드리스트는 `Intrusive list`로 구현되어 있습니다. `Intrusive linked list`는 노드에 실제 데이터를 포함하지 않습니다. 각 노드는 단지 이전 노드와 다음 노드에 대한 포인터를 가질 뿐입니다. 추가로 각 노드를 리스트에 추가될 실제 데이터(구조체)의 필드로 선언해서, 각 노드에 접근해서 리스트의 데이터를 얻어낼 수 있습니다. 이렇게 하면 데이터 구조를 일반화할 수 있어서 실제 데이터의 타입을 더이상 신경쓰지 않아도 됩니다.

예를 들어보겠습니다.

```C
struct nmi_desc {
    spinlock_t lock;
    struct list_head head;
};
```

`list_head`(노드)가 실제로 어떻게 커널에서 사용되는지 여러 예시를 살펴봅시다. 앞에서 말했듯이, 커널 내부의 굉장히 많은 곳에서 리스트를 사용하고 있습니다. 리눅스 커널의 Misc character 드라이버의 예시를 살펴봅시다. [drivers/char/misc.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/drivers/char/misc.c) 에 있는 misc character 드라이버 API는 간단한 하드웨어나 가상 디바이스를 다루기 위한 작은 드라이버를 작성하는데 사용됩니다. 이 드라이버들은 같은 주요 넘버를 공유합니다. 

```C
#define MISC_MAJOR              10
```

하지만 일부는 자신들만의 넘버를 가지고 있습니다. 아래에서 예시를 확인할 수 있습니다.

```
ls -l /dev |  grep 10
crw-------   1 root root     10, 235 Mar 21 12:01 autofs
drwxr-xr-x  10 root root         200 Mar 21 12:01 cpu
crw-------   1 root root     10,  62 Mar 21 12:01 cpu_dma_latency
crw-------   1 root root     10, 203 Mar 21 12:01 cuse
drwxr-xr-x   2 root root         100 Mar 21 12:01 dri
crw-rw-rw-   1 root root     10, 229 Mar 21 12:01 fuse
crw-------   1 root root     10, 228 Mar 21 12:01 hpet
crw-------   1 root root     10, 183 Mar 21 12:01 hwrng
crw-rw----+  1 root kvm      10, 232 Mar 21 12:01 kvm
crw-rw----   1 root disk     10, 237 Mar 21 12:01 loop-control
crw-------   1 root root     10, 227 Mar 21 12:01 mcelog
crw-------   1 root root     10,  59 Mar 21 12:01 memory_bandwidth
crw-------   1 root root     10,  61 Mar 21 12:01 network_latency
crw-------   1 root root     10,  60 Mar 21 12:01 network_throughput
crw-r-----   1 root kmem     10, 144 Mar 21 12:01 nvram
brw-rw----   1 root disk      1,  10 Mar 21 12:01 ram10
crw--w----   1 root tty       4,  10 Mar 21 12:01 tty10
crw-rw----   1 root dialout   4,  74 Mar 21 12:01 ttyS10
crw-------   1 root root     10,  63 Mar 21 12:01 vga_arbiter
crw-------   1 root root     10, 137 Mar 21 12:01 vhci
```

이제 misc 디바이스 드라이버에서 실제로 리스트를 어떻게 사용하는지에 대해 자세히 살펴봅시다. 우선 `miscdevice` 구조체를 살펴봅시다.

```C
struct miscdevice
{
      int minor;
      const char *name;
      const struct file_operations *fops;
      struct list_head list;
      struct device *parent;
      struct device *this_device;
      const char *nodename;
      mode_t mode;
};
```

`miscdevice` 구조체의 네 번째 필드를 보세요. (역주: 실제로 리스트에 저장될 데이터에 list_head 구조체를 삽입하는 것을 확인 가능) 여기서 `list`는 등록된 장치들의 리스트를 뜻합니다. 소스 코드의 시작 부분에서 misc_list의 정의를 볼 수 있습니다. 

```C
static LIST_HEAD(misc_list);
```

which expands to the definition of variables with `list_head` type:

```C
#define LIST_HEAD(name) \
	struct list_head name = LIST_HEAD_INIT(name)
```

and initializes it with the `LIST_HEAD_INIT` macro, which sets previous and next entries with the address of variable - name:

```C
#define LIST_HEAD_INIT(name) { &(name), &(name) }
```

Now let's look on the `misc_register` function which registers a miscellaneous device. At the start it initializes `miscdevice->list` with the `INIT_LIST_HEAD` function:

```C
INIT_LIST_HEAD(&misc->list);
```

which does the same as the `LIST_HEAD_INIT` macro:

```C
static inline void INIT_LIST_HEAD(struct list_head *list)
{
	list->next = list;
	list->prev = list;
}
```

In the next step after a device is created by the `device_create` function, we add it to the miscellaneous devices list with:

```
list_add(&misc->list, &misc_list);
```

Kernel `list.h` provides this API for the addition of a new entry to the list. Let's look at its implementation:

```C
static inline void list_add(struct list_head *new, struct list_head *head)
{
	__list_add(new, head, head->next);
}
```

It just calls internal function `__list_add` with the 3 given parameters:

* new  - new entry.
* head - list head after which the new item will be inserted.
* head->next - next item after list head.

Implementation of the `__list_add` is pretty simple:

```C
static inline void __list_add(struct list_head *new,
			      struct list_head *prev,
			      struct list_head *next)
{
	next->prev = new;
	new->next = next;
	new->prev = prev;
	prev->next = new;
}
```

Here we add a new item between `prev` and `next`. So `misc` list which we defined at the start with the `LIST_HEAD_INIT` macro will contain previous and next pointers to the `miscdevice->list`.

There is still one question: how to get list's entry. There is a special macro:

```C
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

which gets three parameters:

* ptr - the structure list_head pointer;
* type - structure type;
* member - the name of the list_head within the structure;

For example:

```C
const struct miscdevice *p = list_entry(v, struct miscdevice, list)
```

After this we can access to any `miscdevice` field with `p->minor` or `p->name` and etc... Let's look on the `list_entry` implementation:

```C
#define list_entry(ptr, type, member) \
	container_of(ptr, type, member)
```

As we can see it just calls `container_of` macro with the same arguments. At first sight, the `container_of` looks strange:

```C
#define container_of(ptr, type, member) ({                      \
    const typeof( ((type *)0)->member ) *__mptr = (ptr);    \
    (type *)( (char *)__mptr - offsetof(type,member) );})
```

First of all you can note that it consists of two expressions in curly brackets. The compiler will evaluate the whole block in the curly braces and use the value of the last expression.

For example:

```
#include <stdio.h>

int main() {
	int i = 0;
	printf("i = %d\n", ({++i; ++i;}));
	return 0;
}
```

will print `2`.

The next point is `typeof`, it's simple. As you can understand from its name, it just returns the type of the given variable. When I first saw the implementation of the `container_of` macro, the strangest thing I found was the zero in the `((type *)0)` expression. Actually this pointer magic calculates the offset of the given field from the address of the structure, but as we have `0` here, it will be just a zero offset along with the field width. Let's look at a simple example:

```C
#include <stdio.h>

struct s {
        int field1;
        char field2;
		char field3;
};

int main() {
	printf("%p\n", &((struct s*)0)->field3);
	return 0;
}
```

will print `0x5`.

The next `offsetof` macro calculates offset from the beginning of the structure to the given structure's field. Its implementation is very similar to the previous code:

```C
#define offsetof(TYPE, MEMBER) ((size_t) &((TYPE *)0)->MEMBER)
```

Let's summarize all about `container_of` macro. The `container_of` macro returns the address of the structure by the given address of the structure's field with `list_head` type, the name of the structure field with `list_head` type and type of the container structure. At the first line this macro declares the `__mptr` pointer which points to the field of the structure that `ptr` points to and assigns `ptr` to it. Now `ptr` and `__mptr` point to the same address. Technically we don't need this line but it's useful for type checking. The first line ensures that the given structure (`type` parameter) has a member called `member`. In the second line it calculates offset of the field from the structure with the `offsetof` macro and subtracts it from the structure address. That's all.

Of course `list_add` and `list_entry` is not the only functions which `<linux/list.h>` provides. Implementation of the doubly linked list provides the following API:

* list_add
* list_add_tail
* list_del
* list_replace
* list_move
* list_is_last
* list_empty
* list_cut_position
* list_splice
* list_for_each
* list_for_each_entry

and many more.

번역자 추천 참고 문서

* Intrusive List의 이해([pintos](https://web.stanford.edu/class/cs140/projects/pintos/pintos.html)/src/lib/kernel/list.c, list.h) <br>
* 스택오버플로우: [Intrusive list란?](https://stackoverflow.com/questions/3361145/intrusive-lists)
