리눅스 커널 구축 과정
================================================================================

소개
--------------------------------------------------------------------------------

머신에 커스텀 리눅스 커널을 빌드하고 설치하는 방법을 알려 드리지 않겠습니다. 이와 관련하여 도움이 필요하면 [resources](https://encrypted.google.com/search?q=building+linux+kernel#q=building+linux+kernel+from+source+code)를 찾을 수 있습니다. 대신 리눅스 커널 소스 코드의 루트 디렉토리에서 `make`를 실행할 때 어떤 일이 발생하는지 배웁니다.

Linux 커널의 소스 코드를 연구하기 시작했을 때 [makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile)은 내가 처음 연 파일입니다. 그리고 그것은 무서웠습니다 :). [makefile](https://en.wikipedia.org/wiki/Make_%28software%29)에는이 부분을 작성할 때 `1591` 줄의 코드가 포함되어 있으며 커널은 [4.2.0-rc3](https://github.com/torvalds/linux/commit/52721d9d3334c1cb1f76219a161084094ec634dc) 릴리스 입니다.

이 makefile은 Linux 커널 소스 코드에서 최상위 makefile이며 여기서 커널 빌드가 시작됩니다. 그렇습니다. 이것은 매우 크지만 Linux 커널의 소스 코드를 읽은 경우 소스 코드를 포함하는 모든 디렉토리에 자체 메이크 파일이 있음을 알 수 있습니다. 물론 각 소스 파일을 컴파일하고 링크하는 방법을 설명 할 수 없으므로 표준 컴파일 사례 만 연구 할 것입니다. 커널 문서 작성, 커널 소스 코드 정리, [tags](https://en.wikipedia.org/wiki/Ctags) 생성, [cross-compilation](https://en.wikipedia.org/wiki/Cross_compiler) 관련 사항들 ... 우리는 표준 커널 설정 파일로 `make` 실행부터 시작해서 [bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage)를 완성 할 것입니다. 

[make](https://en.wikipedia.org/wiki/Make_%28software%29) 유틸리티에 대해 잘 알고 있다면 더 좋을 것입니다. 그러나 이 부분의 모든 코드를 설명하려고합니다.

시작합시다.

커널 컴파일 전 준비
---------------------------------------------------------------------------------

커널 컴파일을 시작하기 전에 준비해야 할 것이 많습니다. 여기서 중요한 점은 찾아서 구성하는 것입니다.
`make` 등으로 전달되는 명령 행 인수를 파싱하기위한 컴파일 유형 ... 리눅스 커널의 최상위`Makefile '을 살펴봅시다.

Linux 커널의 최상위`Makefile`은 두 가지 주요 제품인 [vmlinux](https://en.wikipedia.org/wiki/Vmlinux) (상주 커널 이미지)와 모듈 (모든 모듈 파일)을 빌드합니다. Linux 커널의 [Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile)은 다음 변수의 정의로 시작합니다.

```Makefile
VERSION = 4
PATCHLEVEL = 2
SUBLEVEL = 0
EXTRAVERSION = -rc3
NAME = Hurr durr I'ma sheep
```

이 변수들은 리눅스 커널의 현재 버전을 결정하고 다른 장소에서 사용됩니다. 예를 들어 동일한 `Makefile`에서 `KERNELVERSION` 변수를 형성 할 때 :

```Makefile
KERNELVERSION = $(VERSION)$(if $(PATCHLEVEL),.$(PATCHLEVEL)$(if $(SUBLEVEL),.$(SUBLEVEL)))$(EXTRAVERSION)
```

그 후에 우리는 `make`에 전달 된 일부 매개 변수를 확인하는 몇 가지 `ifeq` 조건을 볼 수 있습니다. 리눅스 커널 `makefiles`는 사용 가능한 모든 대상과 `make`에 전달할 수있는 일부 명령 행 인수를 인쇄하는 특수한 `make help` 대상을 제공합니다. 예를 들면 다음과 같습니다. `make V = 1` => verbose build. 첫 번째 `ifeq`는 `V = n` 옵션이 `make`에 전달되는지 확인합니다 :

```Makefile
ifeq ("$(origin V)", "command line")
  KBUILD_VERBOSE = $(V)
endif
ifndef KBUILD_VERBOSE
  KBUILD_VERBOSE = 0
endif

ifeq ($(KBUILD_VERBOSE),1)
  quiet =
  Q =
else
  quiet=quiet_
  Q = @
endif

export quiet Q KBUILD_VERBOSE
```

이 옵션이 `make`에 전달되면 `KBUILD_VERBOSE` 변수를 `V` 옵션의 값으로 설정합니다. 그렇지 않으면 우리는 `KBUILD_VERBOSE` 변수를 0으로 설정합니다. 그런 다음  `KBUILD_VERBOSE` 변수의 값을 확인하고 `KBUILD_VERBOSE` 변수의 값에 따라 `quiet` 및 `Q` 변수의 값을 설정합니다. `@`기호는 명령 출력을 억제합니다. 그리고 명령 이전에 존재한다면, 출력은 `Compiling .... scripts / mod / empty.o` 대신 `CC scripts / mod / empty.o`가 될 것입니다. 결국 우리는 이러한 모든 변수를 내 보냅니다. 다음 `ifeq` 문은 `O = / dir` 옵션이 `make`에 전달되었는지 확인합니다. 이 옵션을 사용하면 주어진 `dir`에서 모든 출력 파일을 찾을 수 있습니다 :

```Makefile
ifeq ($(KBUILD_SRC),)

ifeq ("$(origin O)", "command line")
  KBUILD_OUTPUT := $(O)
endif

ifneq ($(KBUILD_OUTPUT),)
saved-output := $(KBUILD_OUTPUT)
KBUILD_OUTPUT := $(shell mkdir -p $(KBUILD_OUTPUT) && cd $(KBUILD_OUTPUT) \
								&& /bin/pwd)
$(if $(KBUILD_OUTPUT),, \
     $(error failed to create output directory "$(saved-output)"))

sub-make: FORCE
	$(Q)$(MAKE) -C $(KBUILD_OUTPUT) KBUILD_SRC=$(CURDIR) \
	-f $(CURDIR)/Makefile $(filter-out _all sub-make,$(MAKECMDGOALS))

skip-makefile := 1
endif # ifneq ($(KBUILD_OUTPUT),)
endif # ifeq ($(KBUILD_SRC),)
```

커널 소스 코드의 최상위 디렉토리를 나타내는 `KBUILD_SRC`와 비어 있는지 (makefile이 처음 실행될 때 비어 있음) 확인합니다. 그런 다음 `KBUILD_OUTPUT` 변수를`O` 옵션으로 전달 된 값으로 설정합니다 (이 옵션이 전달 된 경우). 다음 단계에서 우리는 이 `KBUILD_OUTPUT` 변수를 점검하고 설정되면 다음과 같이합니다 : 

* KBUILD_OUTPUT 값을 임시 `saved-output` 변수에 저장합니다.
* 주어진 출력 디렉토리를 만듭니다.
* 다른 방법으로 인쇄 오류 메시지가 생성 된 디렉토리를 확인합니다.
* 커스텀 출력 디렉토리가 성공적으로 생성 되었다면, 새로운 디렉토리로`make`를 다시 실행합니다 (`-C` 옵션 참조).

다음 `ifeq` 문은 `C` 또는 `M` 옵션이 `make`에 전달되었는지 확인합니다.

```Makefile
ifeq ("$(origin C)", "command line")
  KBUILD_CHECKSRC = $(C)
endif
ifndef KBUILD_CHECKSRC
  KBUILD_CHECKSRC = 0
endif

ifeq ("$(origin M)", "command line")
  KBUILD_EXTMOD := $(M)
endif
```

`C` 옵션은 `makefile`에게 `$ CHECK` 환경 변수가 제공하는 도구로 모든 `c` 소스 코드를 점검해야한다고 기본적으로 [sparse](https://en.wikipedia.org/wiki/Sparse))입니다. 두 번째 `M` 옵션은 외부 모듈을 위한 빌드를 제공합니다 (이 부분에서는 이 경우를 보지 않을 것입니다). 또한 `KBUILD_SRC` 변수가 설정되어 있는지 확인하고, 그렇지 않으면 `srctree` 변수를 `.`로 설정합니다 :

```Makefile
ifeq ($(KBUILD_SRC),)
        srctree := .
endif
		
objtree	:= .
src		:= $(srctree)
obj		:= $(objtree)

export srctree objtree VPATH
```

이것이 커널 소스 트리가 `make`가 실행 된 현재 디렉토리에 있다고 `Makefile`에 알려줍니다. 그런 다음 `objtree`와 다른 변수를이 디렉토리로 설정하고 내보냅니다. 다음 단계는 기본 아키텍처가 무엇인지 나타내는 `SUBARCH` 변수의 값을 얻는 것입니다.

```Makefile
SUBARCH := $(shell uname -m | sed -e s/i.86/x86/ -e s/x86_64/x86/ \
				  -e s/sun4u/sparc64/ \
				  -e s/arm.*/arm/ -e s/sa110/arm/ \
				  -e s/s390x/s390/ -e s/parisc64/parisc/ \
				  -e s/ppc.*/powerpc/ -e s/mips.*/mips/ \
				  -e s/sh[234].*/sh/ -e s/aarch64.*/arm64/ )
```

보시다시피, 기계, 운영 체제 및 아키텍처에 대한 정보를 인쇄하는 [uname](https://en.wikipedia.org/wiki/Uname) 유틸리티를 실행합니다. `uname`의 출력을 얻으면 출력을 구문 분석하고 결과를 `SUBARCH` 변수에 할당합니다. 이제 우리는 `SUBARCH`를 가지고 있으므로, 특정 아키텍처의 디렉토리를 제공하는 `SRCARCH` 변수와 헤더 파일의 디렉토리를 제공하는 `hdr-arch`를 설정합니다 :

```Makefile
ifeq ($(ARCH),i386)
        SRCARCH := x86
endif
ifeq ($(ARCH),x86_64)
        SRCARCH := x86
endif

hdr-arch  := $(SRCARCH)
```

`ARCH`는 `SUBARCH`의 alias입니다. 다음 단계에서 우리는 커널 설정 파일의 경로를 나타내는 `KCONFIG_CONFIG` 변수를 설정했고, 이전에 설정되지 않았다면 기본적으로 `.config`로 설정됩니다 :

```Makefile
KCONFIG_CONFIG	?= .config
export KCONFIG_CONFIG
```

커널 컴파일 중에 사용될 [shell](https://en.wikipedia.org/wiki/Shell_%28computing%29) :

```Makefile
CONFIG_SHELL := $(shell if [ -x "$$BASH" ]; then echo $$BASH; \
	  else if [ -x /bin/bash ]; then echo /bin/bash; \
	  else echo sh; fi ; fi)
```

다음 변수 세트는 Linux 커널 컴파일 중에 사용되는 컴파일러와 관련이 있습니다. 우리는 `c`와 `c ++`에 대한 호스트 컴파일러와 함께 사용될 플래그를 설정했습니다 :

```Makefile
HOSTCC       = gcc
HOSTCXX      = g++
HOSTCFLAGS   = -Wall -Wmissing-prototypes -Wstrict-prototypes -O2 -fomit-frame-pointer -std=gnu89
HOSTCXXFLAGS = -O2
```

다음으로 컴파일러를 나타내는`CC` 변수에 도달하는데 왜`HOST *` 변수가 필요할까? `CC`는 커널 컴파일 과정에서 사용될 타겟 컴파일러이지만, `HOST` 프로그램은 `host` 프로그램 세트를 컴파일하는 동안 사용될 것입니다 (곧 보게 될 것입니다). 그런 다음 컴파일 대상 (모듈, 커널 또는 둘 다)을 결정하는 데 사용되는 `KBUILD_MODULES` 및 `KBUILD_BUILTIN` 변수의 정의를 볼 수 있습니다.

```Makefile
KBUILD_MODULES :=
KBUILD_BUILTIN := 1

ifeq ($(MAKECMDGOALS),modules)
  KBUILD_BUILTIN := $(if $(CONFIG_MODVERSIONS),1)
endif
```

여기서 우리는 이러한 변수의 정의를 볼 수 있으며,`KBUILD_BUILTIN` 변수의 값은 `modules` 만 `make`에 전달하면 `CONFIG_MODVERSIONS` 커널 구성 매개 변수에 따라 달라집니다. 다음 단계는 `kbuild` 파일을 포함시키는 것입니다.

```Makefile
include scripts/Kbuild.include
```

[Kbuild](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kbuild/kbuild.txt) 또는 `Kernel Build System`은 커널 및 해당 모듈의 구축을 관리하기위한 특별한 인프라입니다. `kbuild` 파일은 makefile과 같은 구문을 가지고 있습니다. [scripts/Kbuild.include](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/Kbuild.include) 파일은 `kbuild` 시스템에 대한 일반적인 정의를 제공합니다. 이 `kbuild` 파일 ([makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile)로 돌아 가기)을 포함한 후에는 사용되는 다른 도구와 관련된 변수의 정의를 볼 수 있습니다. 커널 및 모듈 컴파일 (예 : [binutils](http://www.gnu.org/software/binutils/)의 링커, 컴파일러, utils 등) :

```Makefile
AS		= $(CROSS_COMPILE)as
LD		= $(CROSS_COMPILE)ld
CC		= $(CROSS_COMPILE)gcc
CPP		= $(CC) -E
AR		= $(CROSS_COMPILE)ar
NM		= $(CROSS_COMPILE)nm
STRIP		= $(CROSS_COMPILE)strip
OBJCOPY		= $(CROSS_COMPILE)objcopy
OBJDUMP		= $(CROSS_COMPILE)objdump
AWK		= awk
...
...
...
```

그런 다음 헤더 파일 디렉토리에 대한 경로를 지정하는 두 개의 다른 변수 인 `USERINCLUDE`와 `LINUXINCLUDE`를 정의합니다 (첫 번째 경우 사용자에게 공개하고 두 번째 경우 커널에 대해 공개).

```Makefile
USERINCLUDE    := \
		-I$(srctree)/arch/$(hdr-arch)/include/uapi \
		-Iarch/$(hdr-arch)/include/generated/uapi \
		-I$(srctree)/include/uapi \
		-Iinclude/generated/uapi \
        -include $(srctree)/include/linux/kconfig.h

LINUXINCLUDE    := \
		-I$(srctree)/arch/$(hdr-arch)/include \
		...
```

C 컴파일러의 표준 플래그는 다음과 같습니다.

```Makefile
KBUILD_CFLAGS   := -Wall -Wundef -Wstrict-prototypes -Wno-trigraphs \
		   -fno-strict-aliasing -fno-common \
		   -Werror-implicit-function-declaration \
		   -Wno-format-security \
		   -std=gnu89
```

이것들은 다른 makefile에서 업데이트 될 수 있기 때문에 최종 컴파일 플래그가 아닙니다 (예 :`arch /`의 kbuilds). 이 모든 후에는 모든 변수가 다른 makefile에서 사용 가능하도록 내보내집니다. `RCS_FIND_IGNORE` 및 `RCS_TAR_IGNORE` 변수에는 버전 제어 시스템에서 무시 될 파일이 포함됩니다.

```Makefile
export RCS_FIND_IGNORE := \( -name SCCS -o -name BitKeeper -o -name .svn -o    \
			  -name CVS -o -name .pc -o -name .hg -o -name .git \) \
			  -prune -o
export RCS_TAR_IGNORE := --exclude SCCS --exclude BitKeeper --exclude .svn \
			 --exclude CVS --exclude .pc --exclude .hg --exclude .git
```

이것으로 모든 준비를 마쳤습니다. 다음 단계는`vmlinux` 대상을 빌드하는 것입니다.

직접 커널 빌드
--------------------------------------------------------------------------------

이제 모든 준비를 마쳤으며 기본 makefile의 다음 단계는 커널 빌드와 관련이 있습니다. 이 시점 이전에는`make`에 의해 터미널에 아무것도 인쇄되지 않았습니다. 그러나 이제 컴파일의 첫 단계가 시작됩니다. 리눅스 커널 상단 메이크 파일의 [598](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile#L598) 행으로 이동하면`vmlinux` 대상이 있습니다.

```Makefile
all: vmlinux
	include arch/$(SRCARCH)/Makefile
```

Makefile에서 `export RCS_FIND_IGNORE .....` 와 `all : vmlinux .....`사이에있는 많은 행을 놓쳤다 고 걱정하지 마십시오. makefile의 이 부분은 `make * .config` 대상을 담당하며 이 부분의 시작 부분에서 쓴 것처럼 일반적인 방식으로 만 커널을 빌드하는 것을 볼 수 있습니다.

`all :` 대상은 명령 행에 대상이 없을 때의 기본값입니다. 여기에 아키텍처 별 makefile이 포함되어 있음을 알 수 있습니다 (이 경우 [arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)). 이 순간부터 우리는이 makefile을 계속할 것입니다. 보시다시피 `all` 대상은 최상위 makefile에서 조금 더 낮은 `vmlinux` 대상에 따라 다릅니다.
```Makefile
vmlinux: scripts/link-vmlinux.sh $(vmlinux-deps) FORCE
```

`vmlinux`는 정적으로 링크 된 실행 파일 형식의 Linux 커널입니다. [scripts/link-vmlinux.sh](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/link-vmlinux.sh) 스크립트는 서로 다른 컴파일 된 하위 시스템을 vmlinux에 연결하고 결합합니다. 두 번째 대상은 `vmlinux-deps`이며 다음과 같이 정의됩니다.

```Makefile
vmlinux-deps := $(KBUILD_LDS) $(KBUILD_VMLINUX_INIT) $(KBUILD_VMLINUX_MAIN)
```

Linux 커널의 각 최상위 디렉토리에있는 `built-in.o` 세트로 구성됩니다. 나중에 리눅스 커널의 모든 디렉토리를 살펴보면 `Kbuild`가 모든 `$ (obj-y)`파일을 컴파일합니다. 그런 다음 `$ (LD) -r`을 호출하여 이들 파일을 하나의 `built-in.o` 파일로 병합합니다. 현재로서는 `vmlinux-deps`가 없으므로 `vmlinux` 대상이 지금 실행되지 않습니다. 이 상황을 위해 `vmlinux-deps`에는 다음 파일이 포함되어 있습니다.

```
arch/x86/kernel/vmlinux.lds arch/x86/kernel/head_64.o
arch/x86/kernel/head64.o    arch/x86/kernel/head.o
init/built-in.o             usr/built-in.o
arch/x86/built-in.o         kernel/built-in.o
mm/built-in.o               fs/built-in.o
ipc/built-in.o              security/built-in.o
crypto/built-in.o           block/built-in.o
lib/lib.a                   arch/x86/lib/lib.a
lib/built-in.o              arch/x86/lib/built-in.o
drivers/built-in.o          sound/built-in.o
firmware/built-in.o         arch/x86/pci/built-in.o
arch/x86/power/built-in.o   arch/x86/video/built-in.o
net/built-in.o
```

다음으로 실행할 수있는 대상은 다음과 같습니다.

```Makefile
$(sort $(vmlinux-deps)): $(vmlinux-dirs) ;
$(vmlinux-dirs): prepare scripts
	$(Q)$(MAKE) $(build)=$@
```

우리가 볼 수 있듯이 `vmlinux-dirs`는 `prepare`와 `scripts`의 두 대상에 의존합니다. `prepare`는 Linux 커널의 최상위 `Makefile`에 정의되어 있으며 세 단계의 준비를 수행합니다.

```Makefile
prepare: prepare0
prepare0: archprepare FORCE
	$(Q)$(MAKE) $(build)=.
archprepare: archheaders archscripts prepare1 scripts_basic

prepare1: prepare2 $(version_h) include/generated/utsrelease.h \
                   include/config/auto.conf
	$(cmd_crmodverdir)
prepare2: prepare3 outputmakefile asm-generic
```

첫 번째`prepare0`은`x86_64` 특정 [Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)에 정의 된`archheaders` 및 `archscripts`로 확장되는 `archprepare`로 확장됩니다. 살펴봅시다. `x86_64` 특정 메이크 파일은 아키텍처 특정 설정과 관련된 변수의 정의에서 시작합니다 ([defconfig](https://github.com/torvalds/linux/tree/master/arch/x86/configs), 기타...). 그런 다음 [16-bit](https://en.wikipedia.org/wiki/Real_mode) 코드 컴파일을 위한 플래그를 정의하고 `i386`에 대해 `32 BITS` 변수를 계산하거나 어셈블리 소스 코드에 대한 `x86_64` 플래그에 대한`64`, 링커에 대한 플래그. 더 많은 것들은 ([arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)에서 찾을 수 있습니다)). 첫 번째 대상은 makefile에서 syscall 테이블을 생성하는 `archheaders`입니다.

```Makefile
archheaders:
	$(Q)$(MAKE) $(build)=arch/x86/entry/syscalls all
```

And the second target is `archscripts` in this makefile is:

```Makefile
archscripts: scripts_basic
	$(Q)$(MAKE) $(build)=arch/x86/tools relocs
```

상단의 [Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile)에 `scripts_basic` 대상에 의존한다는 것을 알 수 있습니다. 처음에는 [scripts/basic](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/basic/Makefile) make 파일을 실행하는 `scripts_basic` 대상을 확인할 수 있습니다.

```Makefile
scripts_basic:
	$(Q)$(MAKE) $(build)=scripts/basic
```

`scripts / basic / Makefile`에는 `fixdep`와 `bin2`라는 두 호스트 프로그램의 컴파일 대상이 포함되어 있습니다 :

```Makefile
hostprogs-y	:= fixdep
hostprogs-$(CONFIG_BUILD_BIN2C)     += bin2c
always		:= $(hostprogs-y)

$(addprefix $(obj)/,$(filter-out fixdep,$(always))): $(obj)/fixdep
```

첫번째 프로그램은 `fixdep` 입니다 - 소스 코드 파일을 언제 리메이크 할 것인지 알려주는 [gcc](https://gcc.gnu.org/)에 의해 생성 된 의존성 목록을 최적합니다. 두번째 프로그램은 `bin2c`인데, 이것은 `CONFIG_BUILD_BIN2C` 커널 설정 옵션의 값에 의존하며 stdin의 바이너리를 stdout의 C include로 변환 할 수있는 매우 작은 C 프로그램입니다. 여기에서 이상한 표기법인 `hostprogs-y` 등을 확인할 수 있습니다. 이 표기법은 모든 `kbuild` 파일에서 사용되며 [documentation](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kbuild/makefiles.txt)을 참고하세요. 우리의 경우 `hostprogs-y`는 `kbuild`에게 `makedep.c`에서 빌드 될 `fixdep`라는 하나의 호스트 프로그램이 `Makefile`과 같은 디렉토리에 있다고 말한다. 터미널에서 `make`를 실행 한 후의 첫 번째 출력은 이 `kbuild` 파일의 결과입니다 :

```
$ make
  HOSTCC  scripts/basic/fixdep
```

`script_basic` 대상이 실행되면 `archscripts` 대상은  `relocs` 대상이있는 [arch / x86 / tools](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/tools/MakeFile)에 대해 `make`를 실행합니다.

```Makefile
$(Q)$(MAKE) $(build)=arch/x86/tools relocs
```

`relocs_32.c`와`relocs_64.c`는 [relocation](https://en.wikipedia.org/wiki/Relocation_%28computing%29) 정보를 포함하도록 컴파일되며 우리는 `make`의 출력 결과를 볼 것 입니다.

```Makefile
  HOSTCC  arch/x86/tools/relocs_32.o
  HOSTCC  arch/x86/tools/relocs_64.o
  HOSTCC  arch/x86/tools/relocs_common.o
  HOSTLD  arch/x86/tools/relocs
```

`relocs.c`를 컴파일 한 후`version.h`를 점검합니다 :

```Makefile
$(version_h): $(srctree)/Makefile FORCE
	$(call filechk,version.h)
	$(Q)rm -f $(old_version_h)
```

출력에서 볼 수 있습니다.

```
CHK     include/config/kernel.release
```

그리고 리눅스 커널의 최상위 Makefile에서 생성 된 `arch / x86 / include / generated / asm`의 `asm-generic` 대상을 가진 `generic` 어셈블리 헤더의 빌드. `asm-generic` 대상 이후에 `archprepare`가 수행되어 `prepare0` 타겟이 실행됩니다. 위에서 쓴 것처럼 :

```Makefile
prepare0: archprepare FORCE
	$(Q)$(MAKE) $(build)=.
```

`빌드`에 유의하십시오. [scripts/Kbuild.include](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/Kbuild.include)에 정의되어 있으며 다음과 같습니다.

```Makefile
build := -f $(srctree)/scripts/Makefile.build obj
```

또는 우리의 경우에는 현재 소스 디렉토리입니다.`.` :

```Makefile
$(Q)$(MAKE) -f $(srctree)/scripts/Makefile.build obj=.
```

[scripts / Makefile.build](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/Makefile.build)는 `obj` 매개 변수를 통해 주어진 디렉토리에서 `Kbuild` 파일을 찾으려고합니다. 이 `Kbuild` 파일을 포함 시키십시오 :

```Makefile
include $(kbuild-file)
```

이 경우`.`에는 `kernel / bounds.s'와` `arch/x86/kernel/asm-offsets.s`을 생성하는 [Kbuild](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Kbuild) 파일이 포함되어 있습니다. 그런 다음 `준비` 타겟이 작동을 완료했습니다. `vmlinux-dirs`는 또한 다음 프로그램을 컴파일하는 두 번째 대상인 `scripts`에 의존합니다 :`file2alias`, `mk_elfconfig`, `modpost` 등 ... 스크립트 / 호스트 프로그램 컴파일 후 `vmlinux- dirs` 대상을 실행할 수 있습니다. 우선 `vmlinux-dirs`에 무엇이 포함되어 있는지 이해하려고합시다. 필자의 경우 다음 커널 디렉토리의 경로가 포함되어 있습니다.

```
init usr arch/x86 kernel mm fs ipc security crypto block
drivers sound firmware arch/x86/pci arch/x86/power
arch/x86/video net lib arch/x86/lib
```

리눅스 커널의 최상위 [Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile)에서 `vmlinux-dirs`의 정의를 찾을 수 있습니다 :

```Makefile
vmlinux-dirs	:= $(patsubst %/,%,$(filter %/, $(init-y) $(init-m) \
		     $(core-y) $(core-m) $(drivers-y) $(drivers-m) \
		     $(net-y) $(net-m) $(libs-y) $(libs-m)))

init-y		:= init/
drivers-y	:= drivers/ sound/ firmware/
net-y		:= net/
libs-y		:= lib/
...
...
...
```

여기서는`patsubst` 및 `filter` 함수를 사용하여 각 디렉토리에서 `/`기호를 제거하고 `vmlinux-dirs`에 넣습니다. `vmlinux-dirs`에 디렉토리 목록과 다음 코드가 있습니다 :

```Makefile
$(vmlinux-dirs): prepare scripts
	$(Q)$(MAKE) $(build)=$@
```

`$ @`는 여기서 vmlinux-dirs를 나타내며, 이는`vmlinux-dirs`의 모든 디렉토리와 그 내부 디렉토리 (구성에 따라 다름) 에서 재귀 적으로 진행되고 거기에서 `make`를 실행한다는 의미입니다. 출력에서 볼 수 있습니다.

```
  CC      init/main.o
  CHK     include/generated/compile.h
  CC      init/version.o
  CC      init/do_mounts.o
  ...
  CC      arch/x86/crypto/glue_helper.o
  AS      arch/x86/crypto/aes-x86_64-asm_64.o
  CC      arch/x86/crypto/aes_glue.o
  ...
  AS      arch/x86/entry/entry_64.o
  AS      arch/x86/entry/thunk_64.o
  CC      arch/x86/entry/syscall_64.o
```

각 디렉토리의 소스 코드는 컴파일되고`built-in.o`에 연결됩니다 :

```
$ find . -name built-in.o
./arch/x86/crypto/built-in.o
./arch/x86/crypto/sha-mb/built-in.o
./arch/x86/net/built-in.o
./init/built-in.o
./usr/built-in.o
...
...
```

모든 buint-in.o가 빌드되었으므로 이제`vmlinux` 대상으로 돌아갈 수 있습니다. 기억 하시듯이`vmlinux` 대상은 Linux 커널의 최상위 Makefile에 있습니다. `vmlinux`를 연결하기 전에 [samples](https://github.com/torvalds/linux/tree/master/samples), [Documentation] (https://github.com/torvalds/linux/tree/master/Documentation)을 빌드합니다. ) 등 ...하지만 이 부분의 시작 부분에 쓴대로 여기에 설명하지 않겠습니다.

```Makefile
vmlinux: scripts/link-vmlinux.sh $(vmlinux-deps) FORCE
    ...
    ...
    +$(call if_changed,link-vmlinux)
```

주된 목적은 정적으로 링크 된 실행 파일과 [System.map](https://en.wikipedia.org/wiki/System.map)의 생성에 대한 모든 `buil-in-o`를 링킹하는 [scripts/link-vmlinux.sh](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/scripts/link-vmlinux.sh)를 호출하는 것입니다. 결국 우리는 다음과 같은 결과를 보게 될 것입니다 :
```
  LINK    vmlinux
  LD      vmlinux.o
  MODPOST vmlinux.o
  GEN     .version
  CHK     include/generated/compile.h
  UPD     include/generated/compile.h
  CC      init/version.o
  LD      init/built-in.o
  KSYM    .tmp_kallsyms1.o
  KSYM    .tmp_kallsyms2.o
  LD      vmlinux
  SORTEX  vmlinux
  SYSMAP  System.map
```

Linux 커널 소스 트리의 루트에 있는 `vmlinux` 및 `System.map` :

```
$ ls vmlinux System.map
System.map  vmlinux
```

그게 전부입니다.`vmlinux`가 준비되었습니다. 다음 단계는 [bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage)를 만드는 것입니다.

bzImage 빌드
--------------------------------------------------------------------------------

bzImage 파일은 압축 된 리눅스 커널 이미지입니다. `vmlinux`를 만든 후에 `make bzImage`를 실행하면 얻을 수 있습니다. 또는 인자 없이 `make`를 실행할 수 있으며 기본 이미지로 `bzImage`를 얻습니다.

```Makefile
all: bzImage
```

[arch/x86/kernel/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)에서 이 대상을 살펴보면이 이미지가 어떻게 구축되는지 이해하는 데 도움이됩니다. 이미 말했듯이 [arch/x86/kernel/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile)에 정의 된 `bzImage` 대상은 다음과 같습니다.

```Makefile
bzImage: vmlinux
	$(Q)$(MAKE) $(build)=$(boot) $(KBUILD_IMAGE)
	$(Q)mkdir -p $(objtree)/arch/$(UTS_MACHINE)/boot
	$(Q)ln -fsn ../../x86/boot/bzImage $(objtree)/arch/$(UTS_MACHINE)/boot/$@
```

여기서 부트 디렉토리에 대해 `make`라고 불리는 것을 볼 수 있습니다.

```Makefile
boot := arch/x86/boot
```

이제 주요 목표는 `arch/x86/boot` 및 `arch/x86/boot/compressed` 디렉토리에 소스 코드를 작성하고 결국 그들로부터 `setup.bin` 및 `vmlinux.bin`을 빌드하고 `bzImage`를 빌드하는 것입니다.  [arch/x86/boot/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/Makefile)의 첫 번째 대상은 `$ (obj)/setup.elf`입니다. :

```Makefile
$(obj)/setup.elf: $(src)/setup.ld $(SETUP_OBJS) FORCE
	$(call if_changed,ld)
```

`arch/x86/boot` 디렉토리에 `setup.ld` 링커 스크립트와 `boot` 디렉토리의 모든 소스 파일로 확장되는 `SETUP_OBJS` 변수가 이미 있습니다. 우리는 첫 번째 출력을 볼 수 있습니다 :

```Makefile
  AS      arch/x86/boot/bioscall.o
  CC      arch/x86/boot/cmdline.o
  AS      arch/x86/boot/copy.o
  HOSTCC  arch/x86/boot/mkcpustr
  CPUSTR  arch/x86/boot/cpustr.h
  CC      arch/x86/boot/cpu.o
  CC      arch/x86/boot/cpuflags.o
  CC      arch/x86/boot/cpucheck.o
  CC      arch/x86/boot/early_serial_console.o
  CC      arch/x86/boot/edd.o
```

다음 소스 파일은 [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/header.S)이지만 이제 이 대상은 다음 두 헤더 파일에 의존하기 때문에 빌드 할 수 없습니다 

```Makefile
$(obj)/header.o: $(obj)/voffset.h $(obj)/zoffset.h
```

첫 번째는 `sed` 스크립트에 의해 생성 된 `voffset.h`이며 `nm` 유틸리티를 사용하여 `vmlinux`에서 두 개의 주소를 가져옵니다.

```C
#define VO__end 0xffffffff82ab0000
#define VO__text 0xffffffff81000000
```

그것들은 커널의 시작과 끝입니다. 두 번째는 `zoffset.h`이며 [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/Makefile)에서`vmlinux `대상에 의존합니다:

```Makefile
$(obj)/zoffset.h: $(obj)/compressed/vmlinux FORCE
	$(call if_changed,zoffset)
```

`$ (obj) / compressed / vmlinux` 대상은 [arch/x86/boot/compressed](https://github.com/torvalds/linux/tree/master/arch/x86/boot/compressed)에서 소스 코드 파일을 컴파일하는 `vmlinux-objs-y`에 의존합니다 ) 디렉토리에서 `vmlinux.bin`, `vmlinux.bin.bz2`를 생성하고 프로그램 -mkpiggy를 컴파일합니다. 출력에서 이것을 볼 수 있습니다.

```Makefile
  LDS     arch/x86/boot/compressed/vmlinux.lds
  AS      arch/x86/boot/compressed/head_64.o
  CC      arch/x86/boot/compressed/misc.o
  CC      arch/x86/boot/compressed/string.o
  CC      arch/x86/boot/compressed/cmdline.o
  OBJCOPY arch/x86/boot/compressed/vmlinux.bin
  BZIP2   arch/x86/boot/compressed/vmlinux.bin.bz2
  HOSTCC  arch/x86/boot/compressed/mkpiggy
```

`vmlinux.bin`은 디버깅 정보와 주석이 제거 된 `vmlinux.bin` 파일이며 `vmlinux.bin.bz2` 압축 `vmlinux.bin.all` + `u32` 크기 `vmlinux.bin.all`의 크기입니다. `vmlinux.bin.all`은 `vmlinux.bin + vmlinux.relocs`입니다. 여기서 `vmlinux.relocs`는 `relocs` 프로그램에서 처리 한 `vmlinux`입니다 (위 참조). 이 파일들을 얻었으면 `piggy.S` 어셈블리 파일은 `mkpiggy` 프로그램으로 생성되어 컴파일됩니다 :

```Makefile
  MKPIGGY arch/x86/boot/compressed/piggy.S
  AS      arch/x86/boot/compressed/piggy.o
```

이 어셈블리 파일에는 압축 커널에서 계산 된 오프셋이 포함됩니다. 그런 다음 `zoffset`이 생성되었음을 알 수 있습니다.

```Makefile
  ZOFFSET arch/x86/boot/zoffset.h
```

`zoffset.h`와`voffset.h`가 생성되면  [arch/x86/boot](https://github.com/torvalds/linux/tree/master/arch/x86/boot/)에서 소스 코드 파일을 컴파일을 계속할 수 있습니다 :
```Makefile
  AS      arch/x86/boot/header.o
  CC      arch/x86/boot/main.o
  CC      arch/x86/boot/mca.o
  CC      arch/x86/boot/memory.o
  CC      arch/x86/boot/pm.o
  AS      arch/x86/boot/pmjump.o
  CC      arch/x86/boot/printf.o
  CC      arch/x86/boot/regs.o
  CC      arch/x86/boot/string.o
  CC      arch/x86/boot/tty.o
  CC      arch/x86/boot/video.o
  CC      arch/x86/boot/video-mode.o
  CC      arch/x86/boot/video-vga.o
  CC      arch/x86/boot/video-vesa.o
  CC      arch/x86/boot/video-bios.o
```

모든 소스 코드 파일이 컴파일되면 `setup.elf`에 링크됩니다.

```Makefile
  LD      arch/x86/boot/setup.elf
```

또는:

```
ld -m elf_x86_64   -T arch/x86/boot/setup.ld arch/x86/boot/a20.o arch/x86/boot/bioscall.o arch/x86/boot/cmdline.o arch/x86/boot/copy.o arch/x86/boot/cpu.o arch/x86/boot/cpuflags.o arch/x86/boot/cpucheck.o arch/x86/boot/early_serial_console.o arch/x86/boot/edd.o arch/x86/boot/header.o arch/x86/boot/main.o arch/x86/boot/mca.o arch/x86/boot/memory.o arch/x86/boot/pm.o arch/x86/boot/pmjump.o arch/x86/boot/printf.o arch/x86/boot/regs.o arch/x86/boot/string.o arch/x86/boot/tty.o arch/x86/boot/video.o arch/x86/boot/video-mode.o arch/x86/boot/version.o arch/x86/boot/video-vga.o arch/x86/boot/video-vesa.o arch/x86/boot/video-bios.o -o arch/x86/boot/setup.elf
```

마지막 두 가지는`arch / x86 / boot / *`디렉토리에서 컴파일 된 코드를 포함하는`setup.bin` 생성입니다 :

```
objcopy  -O binary arch/x86/boot/setup.elf arch/x86/boot/setup.bin
```

`vmlinux`에서`vmlinux.bin` 생성 :

```
objcopy  -O binary -R .note -R .comment -S arch/x86/boot/compressed/vmlinux arch/x86/boot/vmlinux.bin
```

결국 우리는 호스트 프로그램을 컴파일합니다 : [arch/x86/boot/tools/build.c](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/tools/build.c) `setup.bin`과 `vmlinux.bin`에서 `bzImage`를 생성합니다 :

```
arch/x86/boot/tools/build arch/x86/boot/setup.bin arch/x86/boot/vmlinux.bin arch/x86/boot/zoffset.h arch/x86/boot/bzImage
```

실제로`bzImage`는 연결된 `setup.bin`과 `vmlinux.bin`입니다. 결국 우리는 소스에서 리눅스 커널을 만든 모든 사람들에게 친숙한 결과를 보게 될 것입니다 :

```
Setup is 16268 bytes (padded to 16384 bytes).
System is 4704 kB
CRC 94a88f9a
Kernel: arch/x86/boot/bzImage is ready  (#5)
```

이게 전부입니다..

결론
================================================================================

이 부분의 끝이며 여기서 `make` 명령 실행부터 `bzImage` 생성까지 모든 단계를 보았습니다. Linux 커널 makefile과 Linux 커널 빌드 프로세스는 언뜻 보기에는 혼란 스러울 수 있지만 그렇게 어렵지는 않습니다. 이 부분이 리눅스 커널 구축 과정을 이해하는 데 도움이되기를 바랍니다.

Links
================================================================================

* [GNU make util](https://en.wikipedia.org/wiki/Make_%28software%29)
* [Linux kernel top Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Makefile)
* [cross-compilation](https://en.wikipedia.org/wiki/Cross_compiler)
* [Ctags](https://en.wikipedia.org/wiki/Ctags)
* [sparse](https://en.wikipedia.org/wiki/Sparse)
* [bzImage](https://en.wikipedia.org/wiki/Vmlinux#bzImage)
* [uname](https://en.wikipedia.org/wiki/Uname)
* [shell](https://en.wikipedia.org/wiki/Shell_%28computing%29)
* [Kbuild](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kbuild/kbuild.txt)
* [binutils](http://www.gnu.org/software/binutils/)
* [gcc](https://gcc.gnu.org/)
* [Documentation](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/Documentation/kbuild/makefiles.txt)
* [System.map](https://en.wikipedia.org/wiki/System.map)
* [Relocation](https://en.wikipedia.org/wiki/Relocation_%28computing%29)
