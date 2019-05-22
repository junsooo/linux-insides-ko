# 리눅스 커널의 동기화 기본기능.

이 챕터는 리눅스 커널의 동기화 기본기능들을 설명합니다.

* [Introduction to spinlocks](linux-sync-1.md) - 이 챕터의 첫번째 부분에선 리눅스 커널 내의 스핀락 메커니즘 구현을 설명합니다.
* [Queued spinlocks](linux-sync-2.md) - 두번째 부분에선 스핀락의 다른 종류 (queued spinlock) 를 설명합니다.
* [Semaphores](linux-sync-3.md) - 여기선 리눅스 커널의 `semaphore` 동기화 기능의 구현을 설명합니다.
* [Mutual exclusion](linux-sync-4.md) - 이 부분은 리눅스 커널의 `mutex` 를 설명합니다.
* [Reader/Writer semaphores](linux-sync-5.md) - 여기선 세마포어의 특수한 종류를 설명합니다 - `reader/writer` 세마포어.
* [Sequential locks](linux-sync-6.md) - 여기선 리눅스 커널의 seuqential 락을 설명합니다.
