<!-- generated from vault: 09_strcpy_overflow 코드 분석.md — 편집은 Obsidian에서 -->
---
분류: 코드 분석
버그유형: 힙 버퍼 오버플로 (크기 계산과 복사 범위 불일치)
언어: C
출처: debugging_lab_docker/challenges/09_strcpy_overflow/bug.c
날짜: 2026-09-22
---

# 09_strcpy_overflow 코드 분석

한 줄 시나리오: 문자열 조각 4개("GET ", "/index.html", " HTTP/1.1\r\n\r\n", 200KB body)를 하나로 이어 붙여 길이를 출력한다.

> 증상: `join()`의 `strcpy(out + off, parts[3])`에서 SIGSEGV (libc 내부) · 원인: `joined_size()`가 마지막 조각을 빼고 세서(`i < n - 1`) 버퍼가 199,999바이트 부족



## 함수

### 핵심 로직
```C
/* 필요한 총 바이트 수 = 모든 조각 길이 합 + 종료 문자 1 */
static size_t joined_size(const char *const *parts, int n) {
    size_t total = 1;                        /* '\0' 자리 */
    for (int i = 0; i < n - 1; i++) {        // ← 마지막 조각을 안 셈
        total += strlen(parts[i]);
    }
    return total;
}

static char *join(const char *const *parts, int n) {
    size_t need = joined_size(parts, n);
    char *out = malloc(need);                // 29바이트
    if (!out) { perror("malloc"); exit(1); }

    size_t off = 0;
    for (int i = 0; i < n; i++) {            // 복사는 4개 전부
        strcpy(out + off, parts[i]);         // ← i == 3에서 29바이트 버퍼에 199,999바이트
        off += strlen(parts[i]);
    }
    out[off] = '\0';
    return out;
}
```
- `joined_size()` - 필요한 바이트 수 계산
- `join()` - 그 크기로 `malloc`하고 조각을 차례로 `strcpy`
- 두 함수가 같은 `n`을 받는데 루프 범위가 다르다 (`n - 1` vs `n`)

### 메인 함수
```C
int main(void) {
    static char body[200000];
    memset(body, 'x', sizeof body - 1);
    body[sizeof body - 1] = '\0';

    const char *parts[] = { "GET ", "/index.html", " HTTP/1.1\r\n\r\n", body };
    int n = (int)(sizeof(parts) / sizeof(parts[0]));

    char *msg = join(parts, n);
    printf("joined length = %zu\n", strlen(msg));
    free(msg);
    return 0;
}
```
- `body`는 'x' 199,999개 = HTTP 요청 **본문**. 쓰레기 아님. 붙여야 하는 데이터

실행 흐름
1. `body` 준비, 조각 4개 (4 + 11 + 13 + 199,999 바이트)
2. `joined_size()` **← 원인 지점**: 앞 3개만 더해 `29` 반환
3. `malloc(29)`
4. `join()` 복사 루프 i=0,1,2 → off = 4, 15, 28. 여기까지 정상
5. i=3 `strcpy(out + 28, body)` **← 실제 크래시 지점**: 29바이트 청크에 199,999바이트 → 힙 끝을 넘어 매핑 안 된 페이지까지 쓰다 SIGSEGV


## gdb로 잡기

빌드: `make gdb NAME=09_strcpy_overflow` (= `gdb ./build/09_strcpy_overflow`)

### 1) 죽은 자리 — `run` → `bt`
```
(gdb) run
Program received signal SIGSEGV, Segmentation fault.
0x0000737b9573d9b0 in ?? () from /lib/x86_64-linux-gnu/libc.so.6

(gdb) bt
#0  0x0000737b9573d9b0 in ?? () from /lib/x86_64-linux-gnu/libc.so.6
#1  0x00005861f5ee933b in join (parts=0x7fffb01af340, n=4) at bug.c:54
#2  0x00005861f5ee93fd in main () at bug.c:70
```
`join`함수 54번 줄에 문제가 있다.

### 2) 어떤 데이터인지 — 마지막 반복에서 `need` vs 실제 필요량
```
(gdb) break 54 if i == 3
(gdb) run
(gdb) print need
$1 = 29                                  ← malloc한 크기
(gdb) print off
$2 = 28                                  ← 앞 3개 복사 후 위치
(gdb) print (size_t)strlen(parts[i])
$3 = 199999                              ← 지금 복사하려는 길이
(gdb) print off + (size_t)strlen(parts[i]) + 1
$4 = 200028                              ← 실제 필요한 크기
(gdb) x/8c parts[i]
0x5c2bcb3bf040 <body.0>:  120 'x' 120 'x' 120 'x' ...   ← body, 정상 데이터
```
기대값: `need`는 200,028 실제값: 29
**버퍼가 199,999바이트 모자람**. 복사가 잘못된 게 아니라 크기가 잘못됨

### 3) 왜 그 값인지 — `need`를 누가 만들었나
```
(gdb) break joined_size
(gdb) run
(gdb) finish
Value returned is $1 = 29
```
`joined_size`가 29를 돌려줌. 조각 3개 합(28) + 1. 마지막 조각이 빠짐 → 루프 조건 `i < n - 1`

### 정리
| 단계 | 함수 | 무슨 일 |
|---|---|---|
| 원인 | `joined_size` | `i < n - 1`로 마지막 조각 길이를 안 더함 → 29 반환 |
| 전파 | `join` | 29로 `malloc`. 복사 루프는 `i < n`이라 4개 다 복사하려 함 |
| 증상 | `strcpy` (libc) | 29바이트 청크에 199,999바이트 쓰기 → 힙 넘어 SIGSEGV |


## 첫 시도 — 증상을 고치고 통과인 줄 알았다

```C
// 마지막 조각은 'x'로 채워져있는 쓰레기 데이터??
for (int i = 0; i < n - 1; i++) {        // join의 복사 루프를 줄임
```

| | 결과 |
|---|---|
| 크래시 | 사라짐 |
| valgrind / ASan | clean |
| 출력 | `joined length = 28` ← **틀림** (기대 200,027) |

- 크래시 지점(`strcpy`)을 보고 그 루프를 줄였다. 크기 계산 쪽(`joined_size`)은 안 봤다
- `body`를 쓰레기로 판단했다. 기대 동작 "**모든** 조각을 이어 붙인"을 먼저 읽었으면 안 갔을 길
- 도구가 전부 clean이라 통과라고 생각했다. **크래시가 없어진 것과 고쳐진 것은 다르다** — 기대 출력을 대조하지 않으면 모른다

## 문제
- 크기를 계산하는 함수와 그 크기로 쓰는 함수의 루프 범위가 다르다 (`n - 1` vs `n`)
- 증상은 쓰는 쪽에서 나지만 원인은 계산하는 쪽

**결론 한 문장:** 계산 루프와 복사 루프의 범위를 일치시킨다. 고치기 전에 기대 동작부터 읽는다.


## 수정
- 왜 이 위치에서 고치는가 (책임/소유권): 복사는 4개를 다 하는 게 맞다(기대 동작). 틀린 건 크기를 정하는 `joined_size`. 쓰는 쪽을 줄이면 동작이 바뀌고, 계산하는 쪽을 늘리면 동작은 그대로 버퍼만 맞아진다
- 무엇을 바꾸는가: `joined_size`의 `i < n - 1` → `i < n`

정답 코드
```C
static size_t joined_size(const char *const *parts, int n) {
    size_t total = 1;                        /* '\0' 자리 */
    for (int i = 0; i < n; i++) {            // 원래 i < n - 1 ← 마지막 조각까지 전부 더함
        total += strlen(parts[i]);
    }
    return total;
}

static char *join(const char *const *parts, int n) {
    size_t need = joined_size(parts, n);
    char *out = malloc(need);
    if (!out) { perror("malloc"); exit(1); }

    size_t off = 0;
    for (int i = 0; i < n; i++) {            // 그대로. 계산 루프와 범위 일치
        strcpy(out + off, parts[i]);
        off += strlen(parts[i]);
    }
    out[off] = '\0';
    return out;
}
```

검증: 종료 코드 0 · `joined length = 200027` · valgrind clean · ASan clean

관련: [08_uninitialized_read 코드 분석](../08_uninitialized_read/README.md) (도구가 못 잡는 유형) · 잡지식
