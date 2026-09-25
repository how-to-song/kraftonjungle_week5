<!-- generated from vault: 20_vector_stale_pointer 코드 분석.md — 편집은 Obsidian에서 -->
---
분류: 코드 분석
버그유형: 성장하는 배열 원소의 주소 캐시 → realloc 후 dangling pointer
언어: C
출처: debugging_lab_docker/challenges/20_vector_stale_pointer/bug.c
날짜: 2026-09-25
---

# 20_vector_stale_pointer

한 줄 시나리오: 키별 빈도를 세는 히스토그램. 버킷들을 동적 배열(Histogram.data)에 담는다. 자주 갱신되는 버킷 하나의 "주소"를 hot 포인터로 캐시해 두고 빠르게 증가시킨다. (인덱스 재조회 없이 hot->count 만 바로 갱신하는 흔한 최적화).

> 기대 동작: 스트림의 키들을 모두 반영하고, hot 버킷을 갱신한 값과 전체 합을 출력.


## 구조체

### `Bucket`
```C
typedef struct {
    int  key;
    long count;
} Bucket;
```
- `Histogram`의 데이터를 담는 구조체
- `int key` - 키 값
- `long count` - 키의 개수

### `Histogram`
```C
typedef struct {
    Bucket *data;
    size_t  len, cap;
} Histogram;
```
- `Bucket *data` - 저장할 데이터(`Bucket`) 배열
- `size_t len, cap` - 길이와 용량


## 함수

### 핵심 로직
```C
static void hist_grow(Histogram *h) {
    h->cap = h->cap ? h->cap * 2 : 16;
    Bucket *p = realloc(h->data, h->cap * sizeof(Bucket));   /* 큰 배열은 이동(mmap 재배치) */
    if (!p) { perror("realloc"); free(h->data); exit(1); }
    h->data = p;
}

/* 키를 추가하고, 그 버킷의 주소를 돌려준다(성장이 일어날 수 있음). */
static Bucket *hist_add(Histogram *h, int key) {
    if (h->len == h->cap) hist_grow(h);
    Bucket *b = &h->data[h->len++];
    b->key = key;
    b->count = 0;
    return b;
}

static long hist_total(const Histogram *h) {
    long t = 0;
    for (size_t i = 0; i < h->len; i++) t += h->data[i].count;
    return t;
}
```
- `static void hist_grow(Histogram *h)` - `Bucket`데이터 배열 용량 증가 (용량 2배로 `realloc` → 큰 배열은 새 위치로 이동)
- `static Bucket *hist_add(Histogram *h, int key)` - `Bucket`데이터 배열에 데이터 추가
- `static long hist_total(const Histogram *h)` - 모든 데이터의 `count`총합

### 메인 함수
```C
int main(void) {
    Histogram h = { .data = NULL, .len = 0, .cap = 0 };

    for (int k = 0; k < 200000; k++) hist_add(&h, k);

    Bucket *hot = &h.data[100000];
    hot->count = 1;

    for (int k = 200000; k < 600000; k++) hist_add(&h, k);

    hot->count += 1000;

    printf("hot=%ld total=%ld len=%zu\n", hot->count, hist_total(&h), h.len);
    free(h.data);
    return 0;
}
```

실행 흐름
1. 히스토그램 선언 및 데이터 추가
2. `hot`버킷의 `count = 1`
3. 히스토그램의 데이터 추가 **← 원인 지점**: 늘어난 데이터들로 인해 히스토그램의 버킷 배열 `realloc`
4. `hot`의 주소가 메모리 해제됨 **← 오염/전파 지점**
5. 해제된 메모리 `hot`에 접근 **← 실제 크래시 지점**: SIGSEGV


## gdb로 잡기

빌드: `make gdb NAME=` (= `gdb ./build/`)

### 1) 죽은 자리 — `run` → `bt`
```
(gdb) run
Program received signal SIGSEGV, Segmentation fault.
0x00005555555553e9 in main () at challenges/20_vector_stale_pointer/bug.c:83
83          hot->count += 1000;

(gdb) bt
#0  0x00005555555553e9 in main () at challenges/20_vector_stale_pointer/bug.c:83
```
`hot`에서 SIGSEGV

### 2) 어떤 데이터인지 — `print`, `print *`, `info locals`
```
(gdb) p h->data[100000]
$3 = {key = 100000, count = 1}
(gdb) p &h->data[100000]
$4 = (Bucket *) 0x7ffff5f61a10
(gdb) p hot
$5 = (Bucket *) 0x7ffff7763a10
```
`hot`을 초기화 했을 때 `h->data[100000]`의 주소와 `hot`의 주소가 다르다

### 3) 누가 언제 그렇게 만들었는지 — `break`, `watch`
```
(gdb) break 78
(gdb) break 83

(gdb) run
Breakpoint 1, main () at challenges/20_vector_stale_pointer/bug.c:78
78          Bucket *hot = &h.data[100000];
(gdb) n
79          hot->count = 1;
(gdb) print hot
$9 = (Bucket *) 0x7ffff7763a10
(gdb) print &h.data[100000]
$10 = (Bucket *) 0x7ffff7763a10
(gdb) c
Continuing.

Breakpoint 2, main () at challenges/20_vector_stale_pointer/bug.c:83
83          hot->count += 1000;
(gdb) print hot
$11 = (Bucket *) 0x7ffff7763a10
(gdb) print &h.data[100000]
$12 = (Bucket *) 0x7ffff5f61a10

(gdb) print hot->count
Cannot access memory at address 0x7ffff7763a18
```
`hot`초기화 시 주소와 히스토그램에 데이터를 추가 한 후 `&h->data[100000]`의 주소가 변했다.
`hot`의 주소에 접근할 수 없다.

### 정리
| 단계 | 함수 | 무슨 일 |
|---|---|---|
| 원인 | `main` | 성장하는 배열의 원소 `data[100000]`의 **주소**를 `hot`에 캐시 |
| 전파 | `hist_add` → `hist_grow` → `realloc` | 배열이 커지며 `realloc`이 새 위치로 옮기고 옛 블록 해제 → `hot`은 옛 주소를 가리키는 dangling |
| 증상 | `main` (`hot->count += 1000`) | 해제된 옛 주소에 쓰기 → SIGSEGV |


## 문제
- 히스토그램의 데이터가 추가되면서 `Bucket *data`의 메모리 주소가 `realloc`되며 `hot`의 주소가 메모리 해제 되었다.

**결론 한 문장:** 해당 위치를 주소로 기억하는 것이 아니라 인덱스로 기억한다.


## 수정
- 무엇을 바꾸는가: 인덱스로 접근해 주소가 바뀌더라도 같은 데이터에 접근할 수 있게
	- `Bucket *hot = &h.data[100000];` → `size_t idx = 100000;` (주소 대신 인덱스 저장)
	- `hot->count` → `h.data[idx].count` (접근할 때마다 최신 `h.data` 기준으로 계산)

### 다른 방법
- 꼭 포인터가 필요하면 성장이 끝난 직후 `hot = &h.data[100000];`로 다시 계산한다 (TODO의 두 번째 방법). 다만 성장이 일어나는 곳을 하나라도 놓치면 다시 dangling이 되므로 인덱스가 더 안전하다.

정답 코드
```C
#include <stdio.h>
#include <stdlib.h>

typedef struct {
    int  key;
    long count;
} Bucket;

typedef struct {
    Bucket *data;
    size_t  len, cap;
} Histogram;

static void hist_grow(Histogram *h) {
    h->cap = h->cap ? h->cap * 2 : 16;
    Bucket *p = realloc(h->data, h->cap * sizeof(Bucket));   /* 큰 배열은 이동(mmap 재배치) */
    if (!p) { perror("realloc"); free(h->data); exit(1); }
    h->data = p;
}

/* 키를 추가하고, 그 버킷의 주소를 돌려준다(성장이 일어날 수 있음). */
static Bucket *hist_add(Histogram *h, int key) {
    if (h->len == h->cap) hist_grow(h);
    Bucket *b = &h->data[h->len++];
    b->key = key;
    b->count = 0;
    return b;
}

static long hist_total(const Histogram *h) {
    long t = 0;
    for (size_t i = 0; i < h->len; i++) t += h->data[i].count;
    return t;
}

int main(void) {
    Histogram h = { .data = NULL, .len = 0, .cap = 0 };

    for (int k = 0; k < 200000; k++) hist_add(&h, k);

    size_t idx = 100000;                        // 원래 없었음: 주소 대신 인덱스
    h.data[idx].count = 1;
    // Bucket *hot = &h.data[100000];
    // hot->count = 1;

    for (int k = 200000; k < 600000; k++) hist_add(&h, k);

    // realloc 이후 주소 재지정 (포인터 방식일 때)
    // hot = &h.data[100000];
    // hot->count += 1000;
    h.data[idx].count += 1000;                  // 원래 없었음: 최신 h.data 기준 접근

    // printf("hot=%ld total=%ld len=%zu\n", hot->count, hist_total(&h), h.len);
    printf("hot=%ld total=%ld len=%zu\n", h.data[idx].count, hist_total(&h), h.len);
    free(h.data);
    return 0;
}
```

검증: 종료 코드 0 · `hot=1001 total=1001 len=600000` · valgrind clean · ASan clean


---
<!-- 📋 사용법
     - 맨 위 "기대 동작"은 코드를 읽기 전에 문제 설명에서 옮겨 적는다. 수정 후 검증의 기준이 된다
     - 증상·원인은 풀고 난 뒤 "정리" 표와 "문제"에 쓴다 (풀기 전에 쓰면 힌트가 됨)
     - "증상"과 "원인"의 위치가 다르면 실행 흐름에 둘 다 표시한다 (이게 핵심)
     - gdb 출력은 실제로 찍은 걸 붙이고, 각 블록 아래에 "이 값이 왜 이상한지" 한 줄
     - 정답 코드에서 바꾼 줄에는 // 원래 없었음 / // ← 이유 를 남긴다
     - 검증은 크래시 안 남 + 기대 출력 + valgrind 세 가지를 다 적는다
     - 관련 노트: GDB Cheat Sheet · [10_realloc_dangling 코드 분석](../10_realloc_dangling/README.md)
-->
