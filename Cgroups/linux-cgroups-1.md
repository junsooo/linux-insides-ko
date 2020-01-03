제어 그룹
================================================================================

개요
--------------------------------------------------------------------------------

이번 파트는 [linux insides](https://0xax.gitbooks.io/linux-insides/content/) book의 새로운 장의 첫 부분이며 파트 제목으로 추측 할 수 있듯이 이번 파트는 Linux 커널의 [control groups](https://en.wikipedia.org/wiki/Cgroups) 다른 말로는 `cgroups` 메커니즘을 다룹니다.

`Cgroups`는 리눅스 커널이 제공하는 특별한 메커니즘으로, 프로세서 시간, 그룹당 프로세스 수, 제어 그룹당 메모리 양 또는 프로세스 또는 프로세스 집합에 대한 리소스 조합과 같은 일종의 '리소스'(`resources`)를 할당할 수 있게 해줍니다. `Cgroups`는 일반적인 프로세스가 계층적인 것과 유사하게 계층적으로 구성되어 있으며 자식 `cgroups`는 부모로부터 특정 매개 변수 집합을 상속합니다. 그러나 실제로 그들이 완전히 동일하지는 않습니다. `cgroup`과 (여러 프로세스 그룹의 계층이 동시에 존재할 수 있는) 일반 프로세스와의 주요 차이점은 일반 프로세스 트리는 항상 단일이라는 점입니다. 이(*cgroup*)는 쉬운 단계가 아니었는데, 각 제어 그룹 계층 구조가 '서브시스템'(`subsystems`) 제어 그룹 집합에 붙어 있기 때문입니다.

하나의 '제어 그룹 서브시스템'(`control group subsystem`)은 프로세서 시간 또는 [pids](https://en.wikipedia.org/wiki/Process_identifier), 다른 말로는 `control group`에 대한 프로세스 수 등과 같이 한 종류의 리소스를 나타냅니다. 리눅스 커널은 다음과 같은 12 개의 `control group subsystem`을 지원합니다.

* `cpuset` - 개별 프로세서 및 메모리 노드를 그룹의 작업에 할당합니다;
* `cpu` -스케줄러를 사용하여 cgroup 작업에 프로세서 리소스에 대한 액세스 권한을 제공합니다;
* `cpuacct` - 그룹 별 프로세서 사용량에 대한 보고를 생성합니다;
* `io` -[block devices](https://en.wikipedia.org/wiki/Device_file)에서 읽기/쓰기 제한을 설정합니다;
* `memory` - 그룹의 작업에 의한 메모리 사용 제한을 설정합니다;
* `devices` - 그룹의 작업이 장치에 액세스 할 수 있도록 합니다;
* `freezer` - 그룹에서 작업을 일시중지/재개 할 수 있도록 합니다;
* `net_cls` - 그룹의 작업에서 네트워크 패킷을 표시(mark) 할 수 있게 합니다;
* `net_prio` - 그룹의 네트워크 인터페이스마다 네트워크 트래픽 우선 순위를 동적으로 설정하는 방법을 제공합니다;
* `perf_event` - 그룹에 [perf events](https://en.wikipedia.org/wiki/Perf_\(Linux\))에 대한 액세스를 제공합니다;
* `hugetlb` - 그룹의 [huge pages](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)에 대한 지원을 활성화합니다;
* `pid` - 그룹의 프로세스 수에 제한을 설정합니다.

이러한 각 제어 그룹 하위 시스템은 관련한 구성 옵션에 따라 다릅니다. 예를 들어 `cpuset` 서브시스템은`CONFIG_CPUSETS` 커널 구성 옵션을 통해, `io` 서브 시스템은 `CONFIG_BLK_CGROUP` 커널 구성 옵션 등을 통해 활성화해야합니다. 이러한 모든 커널 구성 옵션은 `General setup → Control Group`에서 찾을 수 있습니다:

![menuconfig](http://oi66.tinypic.com/2rc2a9e.jpg)

[proc](https://en.wikipedia.org/wiki/Procfs) 파일 시스템을 통해서:

```
$ cat /proc/cgroups 
#subsys_name	hierarchy	num_cgroups	enabled
cpuset	8	1	1
cpu	7	66	1
cpuacct	7	66	1
blkio	11	66	1
memory	9	94	1
devices	6	66	1
freezer	2	1	1
net_cls	4	1	1
perf_event	3	1	1
net_prio	4	1	1
hugetlb	10	1	1
pids	5	69	1
```

혹은 [sysfs](https://en.wikipedia.org/wiki/Sysfs)을 통해서 컴퓨터에서 활성화 된 제어 그룹을 볼 수 있습니다.:

```
$ ls -l /sys/fs/cgroup/
total 0
dr-xr-xr-x 5 root root  0 Dec  2 22:37 blkio
lrwxrwxrwx 1 root root 11 Dec  2 22:37 cpu -> cpu,cpuacct
lrwxrwxrwx 1 root root 11 Dec  2 22:37 cpuacct -> cpu,cpuacct
dr-xr-xr-x 5 root root  0 Dec  2 22:37 cpu,cpuacct
dr-xr-xr-x 2 root root  0 Dec  2 22:37 cpuset
dr-xr-xr-x 5 root root  0 Dec  2 22:37 devices
dr-xr-xr-x 2 root root  0 Dec  2 22:37 freezer
dr-xr-xr-x 2 root root  0 Dec  2 22:37 hugetlb
dr-xr-xr-x 5 root root  0 Dec  2 22:37 memory
lrwxrwxrwx 1 root root 16 Dec  2 22:37 net_cls -> net_cls,net_prio
dr-xr-xr-x 2 root root  0 Dec  2 22:37 net_cls,net_prio
lrwxrwxrwx 1 root root 16 Dec  2 22:37 net_prio -> net_cls,net_prio
dr-xr-xr-x 2 root root  0 Dec  2 22:37 perf_event
dr-xr-xr-x 5 root root  0 Dec  2 22:37 pids
dr-xr-xr-x 5 root root  0 Dec  2 22:37 systemd
```

이미 예상 하셨겠지만 `control groups` 메커니즘은 직접적인 리눅스 커널에 대한 요구사항으로가 아니라 주로 사용자 공간 요구에 대한 요구사항으로 개발 된 메커니즘입니다. `control group`을 사용하려면 먼저 컨트롤 그룹을 만들어야합니다. 우리는 두 가지 방법으로 `cgroup`을 만들 수 있습니다.

첫 번째 방법은 `/sys/fs/cgroup`에서 서브시스템에 서브 디렉토리(subdirectory)를 생성하고 서브 디렉토리를 생성 한 직후 자동으로 생성되는`tasks` 파일에 작업의 pid를 추가하는 것입니다.

두 번째 방법은 `libcgroup` 라이브러리 (Fedora는 `libcgroup-tools`)에서 utils로 `cgroups`을 생성/파기/관리하는 것입니다.

간단한 예를 생각해 봅시다. [bash](https://www.gnu.org/software/bash/) 스크립트에 따르면 현재 프로세스의 제어 터미널을 나타내는 `/dev/tty` 디바이스에 한 줄이 출력될 것입니다:

```shell
#!/bin/bash

while :
do
    echo "print line" > /dev/tty
    sleep 5
done
```

따라서 이 스크립트를 실행하면 다음과 같은 결과가 나타납니다.

```
$ sudo chmod +x cgroup_test_script.sh
~$ ./cgroup_test_script.sh 
print line
print line
print line
...
...
...
```

이제 컴퓨터에 `cgroupfs`가 마운트되어있는 곳으로 갑시다. 방금 봤듯이, 이 디렉토리는 `/sys/fs/cgroup` 디렉토리이지만 원한다면 원하는 곳 어디든 마운트 할 수 있습니다.

```
$ cd /sys/fs/cgroup
```

이제 `cgroup`의 작업의 장치에 대한 액세스를 허용하거나 거부하는 종류의 리소스를 나타내는 `devices` 서브 디렉토리로 이동하겠습니다.

```
# cd devices
```

그리고 거기에 `cgroup_test_group` 디렉토리를 만듭니다:

```
# mkdir cgroup_test_group
```

`cgroup_test_group` 디렉토리가 생성되면 다음 파일이 생성됩니다:

```
/sys/fs/cgroup/devices/cgroup_test_group$ ls -l
total 0
-rw-r--r-- 1 root root 0 Dec  3 22:55 cgroup.clone_children
-rw-r--r-- 1 root root 0 Dec  3 22:55 cgroup.procs
--w------- 1 root root 0 Dec  3 22:55 devices.allow
--w------- 1 root root 0 Dec  3 22:55 devices.deny
-r--r--r-- 1 root root 0 Dec  3 22:55 devices.list
-rw-r--r-- 1 root root 0 Dec  3 22:55 notify_on_release
-rw-r--r-- 1 root root 0 Dec  3 22:55 tasks
```

여기서 우리의 관심사는 `tasks`와 `devices.deny` 파일입니다. 첫 번째 `tasks` 파일은`cgroup_test_group`에 첨부 될 프로세스의 pid를 포함할 것입니다. 두 번째로 `devices.deny` 파일은 거부된 장치 목록을 포함합니다. 기본적으로 새로 만든 그룹에는 장치 액세스에 대한 제한이 없습니다. 장치를 금지하려면 (이 경우`/dev/tty`) `devices.deny`에 다음 명령줄을 작성합니다:

```
# echo "c 5:0 w" > devices.deny
```

이 명령줄을 단계별로 살펴 보겠습니다. 첫 번째 `c`문자는 장치의 유형을 나타냅니다. 우리의 경우 `/dev/tty`는`char device`입니다. `ls` 명령어의 출력에서 이를 확인할 수 있습니다.

```
~$ ls -l /dev/tty
crw-rw-rw- 1 root tty 5, 0 Dec  3 22:48 /dev/tty
```

권한 목록에서 첫 번째 `c` 문자를 살펴보세요. 두 번째 부분 `5:0`은 장치의 부(minor) 번호와 주요(major) 번호입니다. 이 숫자는 `ls`의 결과에서도 볼 수 있습니다. 그리고 마지막 `w` 문자는 작업이 지정된 장치에 쓰는 것을 금지합니다. 그럼  `cgroup_test_script.sh` 스크립트를 시작해 봅시다 :

```
~$ ./cgroup_test_script.sh 
print line
print line
print line
...
...
```

그리고 이 프로세스의 pid를 우리의 그룹의 `devices/tasks` 파일에 추가합니다:

```
# echo $(pidof -x cgroup_test_script.sh) > /sys/fs/cgroup/devices/cgroup_test_group/tasks
```

그 결과는 예상한 대로:

```
~$ ./cgroup_test_script.sh 
print line
print line
print line
print line
print line
print line
./cgroup_test_script.sh: line 5: /dev/tty: Operation not permitted
```

[docker](https://en.wikipedia.org/wiki/Docker_\(software\)) 컨테이너를 실행할 때도 비슷한 상황이 발생합니다. 예를 들어:

```
~$ docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
fa2d2085cd1c        mariadb:10          "docker-entrypoint..."   12 days ago         Up 4 minutes        0.0.0.0:3306->3306/tcp   mysql-work

~$ cat /sys/fs/cgroup/devices/docker/fa2d2085cd1c8d797002c77387d2061f56fefb470892f140d0dc511bd4d9bb61/tasks | head -3
5501
5584
5585
...
...
...
```

따라서, `docker` 컨테이너를 시작하는 동안 `docker`는 이 컨테이너 안의 프로세스에 대한 `cgroup`을 만듭니다.

```
$ docker exec -it mysql-work /bin/bash
$ top
  PID USER      PR  NI    VIRT    RES    SHR S  %CPU %MEM     TIME+ COMMAND                                                                                   1 mysql     20   0  963996 101268  15744 S   0.0  0.6   0:00.46 mysqld
   71 root      20   0   20248   3028   2732 S   0.0  0.0   0:00.01 bash
   77 root      20   0   21948   2424   2056 R   0.0  0.0   0:00.00 top
```

그리고 호스트 컴퓨터에서 이 `cgroup`을 볼 수 있습니다 :

```C
$ systemd-cgls

Control group /:
-.slice
├─docker
│ └─fa2d2085cd1c8d797002c77387d2061f56fefb470892f140d0dc511bd4d9bb61
│   ├─5501 mysqld
│   └─6404 /bin/bash
```

이제 우리는 `control group` 메커니즘과 수동으로 이를 사용하는 방법 및 이 메커니즘의 목적에 대해 약간 알게 되었습니다. 리눅스 커널 소스 코드를 살펴보고이 메커니즘의 구현을 시작해봅시다.

제어 그룹의 초기 초기화
--------------------------------------------------------------------------------

이제 `제어 그룹` 리눅스 커널 메커니즘에 대한 약간의 이론을 본 후, 이 메커니즘에 더 친숙해지기 위해 리눅스 커널의 소스 코드를 살펴보겠습니다. 항상 그랬듯이 우리는 `control groups`의 초기화부터 시작할 것입니다. `cgroups`의 초기화는 리눅스 커널에서 두 부분으로 나뉩니다 : 초기(early)와 후기(late). 이 부분에서는 초기(`early`)부분 만 고려하고 후기(`late`)부분은 다음 파트에서 다루겠습니다.

`cgroups`의 초기 초기화는 리눅스 커널의 초기 초기화 시기동안 [init/main.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/init/main.c)의:

```C
cgroup_init_early();
```

함수 호출에서 시작합니다. 이 함수는 [kernel/cgroup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cgroup.c) 소스 코드 파일에 정의되어 있으며 다음 두 가지 지역 변수를 정의하면서 시작합니다:

```C
int __init cgroup_init_early(void)
{
	static struct cgroup_sb_opts __initdata opts;
	struct cgroup_subsys *ss;
    ...
    ...
    ...
}
```

`cgroup_sb_opts` 구조체는 동일한 소스 코드 파일에 정의되어 있으며 `cgroupfs`의 마운트 옵션을 나타내고 다음과 같이 생겼습니다:

```C
struct cgroup_sb_opts {
	u16 subsys_mask;
	unsigned int flags;
	char *release_agent;
	bool cpuset_clone_children;
	char *name;
	bool none;
};
```

예를 들어 `name=`옵션이 있고 서브시스템은 없이 (`my_cgrp`로) 이름이 지정된 cgroup 계층(hierarchy)을 작성할 수 있습니다.

```
$ mount -t cgroup -oname=my_cgrp,none /mnt/cgroups
```

두 번째 변수- `ss`는 [include/linux/cgroup-defs.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cgroup-defs.h) 헤더 파일에 정의된 `cgroup_subsys` 구조체 타입을 가지며, 형식 이름에서 알 수 있듯이 이는 `cgroup` 서브시스템을 나타냅니다. 이 구조체는 다음과 같은 다양한 필드와 콜백 함수를 가지고 있습니다:

```C
struct cgroup_subsys {
    int (*css_online)(struct cgroup_subsys_state *css);
    void (*css_offline)(struct cgroup_subsys_state *css);
    ...
    ...
    ...
    bool early_init:1;
    int id;
    const char *name;
    struct cgroup_root *root;
    ...
    ...
    ...
}
```

예를 들어 여기서 `ccs_online` 및 `css_offline` 콜백은 cgroup이 모든 할당을 성공적으로 완료한 후 호출되고 cgroup은 각각 해제되기 전에 호출됩니다. `early_init` 플래그는 초기에 초기화 될 수 있는 서브시스템을 표시합니다. `id` 및 `name` 필드는 각각 등록 된 서브 시스템 배열의 고유 식별자와 서브 시스템의 `name`을 나타냅니다. 마지막 `root` 필드는 cgroup 계층의 루트에 대한 포인터를 나타냅니다.

물론 `cgroup_subsys` 구조체는 더 크고 다른 필드들도 가지고 있지만 현재로서는 이걸로 충분합니다. 이제 `cgroups` 메커니즘과 관련된 중요한 구조체에 대해 알았으니 이제 `cgroup_init_early` 함수로 돌아갑시다. 이 함수의 주요 목적은 몇몇 서브 시스템을 초기 초기화하는 것입니다. 이미 짐작하셨겠지만, 이 '초기'(`early`) 서브 시스템은 `cgroup_subsys-> early_init = 1`을 가지고 있어야합니다. 어떤 서브 시스템이 초기에 초기화 될 수 있는지 살펴봅시다.

두 개의 로컬 변수를 정의한 후 다음 코드 줄을 볼 수 있습니다.

```C
init_cgroup_root(&cgrp_dfl_root, &opts);
cgrp_dfl_root.cgrp.self.flags |= CSS_NO_REF;
```

여기서는 기본 통합 계층(default unified hierarchy)의 초기화를 실행하는 `init_cgroup_root` 함수의 호출을 볼 수 있으며, 그 후 이 css의 참조 카운트를 비활성화하기 위해 이 기본 `cgroup` 상태에서 `CSS_NO_REF` 플래그를 설정합니다. `cgrp_dfl_root`는 동일한 소스 코드 파일에 정의되어 있습니다 :

```C
struct cgroup_root cgrp_dfl_root;
```

이미 추측하셨듯 `cgrp` 필드는 `cgroup` 구조체로 나타내어지고 `cgroup`을 나타내며 [include/linux/cgroup-defs.h] (https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cgroup-defs.h) 헤더 파일에 정의되어 있습니다. 우리는 이미 리눅스 커널에서 `task_struct`로 표현되는 프로세스를 알고 있습니다. `task_struct`에는 이 작업이 연결된 `cgroup`에 대한 직접적인 링크가 없습니다. 그러나 `task_struct`의 `css_set` 필드를 통해 이에 도달 할 수 있습니다. 이 `css_set` 구조체는 서브 시스템 상태의 배열에 대한 포인터를 가지고 있습니다.

```C
struct css_set {
    ...
    ...
    ....
    struct cgroup_subsys_state *subsys[CGROUP_SUBSYS_COUNT];
    ...
    ...
    ...
}
```

또한 `cgroup_subsys_state`를 통해 프로세스는 그 프로세스가 연결된 `cgroup`을 얻을 수 있습니다:

```C
struct cgroup_subsys_state {
    ...
    ...
    ...
    struct cgroup *cgroup;
    ...
    ...
    ...
}
```

따라서 `cgroups`과 관련된 데이터 구조의 전체적인 그림은 다음과 같습니다.

```                                                 
+-------------+         +---------------------+    +------------->+---------------------+          +----------------+
| task_struct |         |       css_set       |    |              | cgroup_subsys_state |          |     cgroup     |
+-------------+         |                     |    |              +---------------------+          +----------------+
|             |         |                     |    |              |                     |          |     flags      |
|             |         |                     |    |              +---------------------+          |  cgroup.procs  |
|             |         |                     |    |              |        cgroup       |--------->|       id       |
|             |         |                     |    |              +---------------------+          |      ....      | 
|-------------+         |---------------------+----+                                               +----------------+
|   cgroups   | ------> | cgroup_subsys_state | array of cgroup_subsys_state
|-------------+         +---------------------+------------------>+---------------------+          +----------------+
|             |         |                     |                   | cgroup_subsys_state |          |      cgroup    |
+-------------+         +---------------------+                   +---------------------+          +----------------+
                                                                  |                     |          |      flags     |
                                                                  +---------------------+          |   cgroup.procs |
                                                                  |        cgroup       |--------->|        id      |
                                                                  +---------------------+          |       ....     |
                                                                  |    cgroup_subsys    |          +----------------+
                                                                  +---------------------+
                                                                             |
                                                                             |
                                                                             ↓
                                                                  +---------------------+
                                                                  |    cgroup_subsys    |
                                                                  +---------------------+
                                                                  |         id          |
                                                                  |        name         |
                                                                  |      css_online     |
                                                                  |      css_ofline     |
                                                                  |        attach       |
                                                                  |         ....        |
                                                                  +---------------------+
```



따라서 `init_cgroup_root`는 `cgrp_dfl_root`를 기본값으로 채 웁니다. 그 다음은 초기 `css_set`을 시스템의 첫 번째 프로세스를 나타내는 `init_task`에 할당하는 것입니다.

```C
RCU_INIT_POINTER(init_task.cgroups, &init_css_set);
```

그리고 `cgroup_init_early` 함수에서 마지막으로 짚어 봐야 할 것은 `early cgroups`의 초기화입니다. 여기서는 등록된 모든 서브 시스템을 살펴보고 고유 ID, 서브 시스템 이름을 지정하고 초기로 표시(mark)된 서브 시스템에 대해 `cgroup_init_subsys` 함수를 호출합니다.

```C
for_each_subsys(ss, i) {
		ss->id = i;
		ss->name = cgroup_subsys_name[i];

        if (ss->early_init)
			cgroup_init_subsys(ss, true);
}
```

여기서 `for_each_subsys`는 [kernel/cgroup.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cgroup.c) 소스 코드 파일에 정의 된 매크로이며 그저 `cgroup_subsys` 배열에 대한 `for` 루프로 확장됩니다. 그 배열의 정의는 동일한 소스 코드 파일에서 찾을 수 있으며 약간 특이한 방식같아 보입니다:

```C
#define SUBSYS(_x) [_x ## _cgrp_id] = &_x ## _cgrp_subsys,
    static struct cgroup_subsys *cgroup_subsys[] = {
        #include <linux/cgroup_subsys.h>
};
#undef SUBSYS
```

그것은 하나의 인자(서브 시스템의 이름)를 취하고 cgroup 서브시스템의 `cgroup_subsys` 배열을 정의하는`SUBSYS` 매크로로 정의됩니다. 또한 배열이 [linux/cgroup_subsys.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/cgroup_subsys.h) 헤더 파일의 내용으로 초기화 된 것을 볼 수 있습니다. 이 헤더 파일을 살펴보면 주어진 서브시스템 이름을 가진 `SUBSYS` 매크로 세트가 다시 나타납니다.

```C
#if IS_ENABLED(CONFIG_CPUSETS)
SUBSYS(cpuset)
#endif

#if IS_ENABLED(CONFIG_CGROUP_SCHED)
SUBSYS(cpu)
#endif
...
...
...
```

이것은 `SUBSYS` 매크로를 처음 정의한 뒤에 있는 `#undef` 문 때문에 작동합니다. `&_x##_cgrp_subsys` 표현식을보면 `##`연산자는 `C` 매크로에서 오른쪽과 왼쪽 표현식을 연결합니다. 그러므로 `cpuset`,`cpu` 등을`SUBSYS` 매크로에 전달할 때 `cpuset_cgrp_subsys`,`cp_cgrp_subsys`가 정의되었어야 할 것입니다. 그리고 그것은 사실입니다. [kernel/cpuset.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/kernel/cpuset.c) 소스 코드 파일을 보면 다음 정의가 표시됩니다.

```C
struct cgroup_subsys cpuset_cgrp_subsys = {
    ...
    ...
    ...
	.early_init	= true,
};
```

따라서 `cgroup_init_early` 함수의 마지막 단계는 `cgroup_init_subsys` 함수를 호출하여 초기 서브시스템을 초기화하는 것입니다. 다음과 같은 초기 서브시스템이 초기화될 것입니다.

* `cpuset`;
* `cpu`;
* `cpuacct`.

`cgroup_init_subsys` 함수는 주어진 서브 시스템을 기본값으로 초기화합니다. 예를 들어 계층의 루트를 설정하고, `css_alloc` 콜백 함수를 호출하여 지정된 서브 시스템에 공간을 할당하고, 서브시스템이 있는 경우 서브시스템을 부모와 연결하고, 할당 된 서브 시스템을 초기 프로세스에 추가하는 등입니다.

이것으로 이 순간 초기 서브 시스템이 초기화되었습니다.

결론
--------------------------------------------------------------------------------

이것으로 리눅스 커널에서 `Control Groups` 메커니즘을 소개하는 것은 첫 번째 파트는 끝입니다. 우리는 `Control Group`메커니즘과 관련된 것들을 초기화하는 몇 가지 이론과 그 첫번재 단계를 다루었습니다. 다음 부분에서는 제어 그룹(`control groups`)의 보다 실용적인 측면을 계속해서 살펴볼 것입니다.

질문이나 제안 사항이 있으면 [twitter](https://twitter.com/0xAX)에 의견을 보내거나 핑 (Ping) 해주십시오.

**영어는 제 모국어가 아닙니다, 그리고 여타 불편하셨던 점에 대해서 정말로 사과드립니다. 만약 실수들을 찾아내셨다면 부디 [linux-insides 원본](https://github.com/0xAX/linux-internals)으로, 번역에 대해서는 [linux-insides 한국 번역](https://github.com/junsooo/linux-insides-ko)로 PR을 보내주세요.**

참고 링크
--------------------------------------------------------------------------------

* [control groups](https://en.wikipedia.org/wiki/Cgroups)
* [PID](https://en.wikipedia.org/wiki/Process_identifier)
* [cpuset](http://man7.org/linux/man-pages/man7/cpuset.7.html)
* [block devices](https://en.wikipedia.org/wiki/Device_file)
* [huge pages](https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt)
* [sysfs](https://en.wikipedia.org/wiki/Sysfs)
* [proc](https://en.wikipedia.org/wiki/Procfs)
* [cgroups kernel documentation](https://www.kernel.org/doc/Documentation/cgroup-v1/cgroups.txt)
* [cgroups v2](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)
* [bash](https://www.gnu.org/software/bash/)
* [docker](https://en.wikipedia.org/wiki/Docker_\(software\))
* [perf events](https://en.wikipedia.org/wiki/Perf_\(Linux\))
* [Previous chapter](https://0xax.gitbooks.io/linux-insides/content/MM/linux-mm-1.html)
