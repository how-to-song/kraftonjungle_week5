<!-- generated from vault: 01_use_after_free 코드 분석.md — 편집은 Obsidian에서 -->
아주 작은 GUI 흉내

각 위젯(Widget)은 힙 객체이며 첫 멤버로 "vtable"(render/on_event 함수 포인터 모아두는 구조체)를 가진다.
Screen은 위젯 포인터 배열을 들고 있고, 이벤트를 나눠준 뒤 (dispatch) 한 프레임을 그린다(render).

> 증상: frame 2 렌더 중 SIGSEGV · 원인: 닫기 핸들러가 위젯을 free 했는데 Screen 배열의 포인터는 그대로 (Use-After-Free)


## 구조체

VTable
```C
typedef struct {
    void (*render)(Widget *self);
    void (*on_event)(Widget *self, int code);
} VTable;
```
- `void (*render)(Widget *self);` - 해당 위젯을 그려주는 함수 포인터
	- 각 위젯마다 그려주는 방식이 다름
- `void (*on_event)(Widget *self, int code)` - 해당 위젯이 어떤 행동을 할지 정하는 함수 포인터
	- 각 위젯마다 작동하는 방식이 다르지만 이 코드에서는 DIALOG_VT만 다른 코드를 갖는다.

Widget
```C
struct Widget {
    const VTable *vtbl;
    int id;
    int closed;
    char label[24];
};
```
- `const VTable *vtbl` - 각 위젯 별로 실행해야 하는 함수 포인터 구조체. **첫 멤버**라서 위젯 메모리의 맨 앞 8바이트가 곧 vtbl이다 (아래 gdb에서 이 자리가 덮여 있는 걸 본다)
- `id` - 해당 위젯 id
- `closed` - 해당 위젯이 닫혔는지 판단
- `char label[24]` - 해당 위젯이 출력해야되는 문자열

Screen
```C
#define MAX_WIDGETS 8
typedef struct {
    Widget *items[MAX_WIDGETS];
    int count;
} Screen;
```
- `Widget *items[MAX_WIDGETS]` - 화면에 올라간 위젯 포인터 배열 (최대 위젯은 8개). **위젯의 소유자**는 이 배열이다
- `int count` - 화면에 올라간 위젯의 개수


## 함수

### 위젯 종류별 동작

- 각 위젯별 렌더 함수
```C
	static void button_render(Widget *self) {
		printf("  [Button #%d] \"%s\"\n", self->id, self->label);
	}

	static void label_render(Widget *self) {
		printf("  Label #%d: %s\n", self->id, self->label);
	}

	static void dialog_render(Widget *self) {
	    printf("  <<Dialog #%d>> %s\n", self->id, self->label);
	}
```

  - 각 위젯별 이벤트 함수
```C
// 의도적인 no-op. 버튼·라벨은 이벤트에 반응할 게 없지만
// on_event 포인터가 NULL이면 안 되므로 "아무것도 안 하는 핸들러"를 꽂아 둔다
static void widget_noop_event(Widget *self, int code) { (void)self; (void)code; }

/* 다이얼로그는 이벤트 코드 1(닫기)을 받으면 스스로 정리(파괴)된다 */
static void dialog_on_event(Widget *self, int code);

static void dialog_on_event(Widget *self, int code) {
    if (code == 1) {
        self->closed = 1;
        widget_destroy(self);   // ← 버그의 원인 (아래 참고)
    }
}

// 함수별 책임 분리
static void widget_destroy(Widget *w) {
    free(w);
}
```

#### 각 이벤트 함수들이 이벤트 코드(`int code`)를 인자로 받는 이유
- 인터페이스 통일: `screen_dispatch`에서 위젯 종류는 모르고 `w->vtbl->on_event(w, code)`를 호출하는데 `VTable`구조체에서 같은 함수 포인터 타입 `void (*on_event)(Widget *, int)`이어야 하고, 이 `int code`는 Screen에서 위젯에게 어떤 이벤트를 동작해야하는지 알려주는 인자이다.
- 확장성: 현재는 삭제(`code = 1`)만 있는데 다양한 동작을 하게 하고 싶을 때, `code = 2` 또는 `code = 3` 등 `dialog_on_event()`만 수정하면 `screen_dispatch()`등의 다른 함수들은 수정할 필요가 없다.


- 위젯 생성/삭제 및 위젯별 함수 포인터 연결
```C
// 위젯별 렌더, 이벤트 함수 연결
static const VTable BUTTON_VT = { button_render, widget_noop_event };
static const VTable LABEL_VT  = { label_render,  widget_noop_event };
static const VTable DIALOG_VT = { dialog_render, dialog_on_event  };

// 위젯 생성
static Widget *widget_new(const VTable *vt, int id, const char *label) {

    /* [Thinking Point]
    *   w 에 아직 아무 값도 넣지 않았는데, sizeof *w 로 *w 를 써도 괜찮은 이유는?
    *   tip 1. sizeof 는 피연산자를 '실행(역참조)'하지 않고 '타입'만 본다.
    *          → *w 의 타입(Widget)만 필요할 뿐, w 를 실제로 따라가지 않는다.
    *   tip 2. 그래서 sizeof *w 는 (VLA 제외) 컴파일 타임에 sizeof(Widget) 상수로 치환된다.
    *   생각해보기: sizeof(Widget) 대신 sizeof *w 로 쓰면 어떤 장점이 있을까?
    */
    Widget *w = malloc(sizeof *w);
    if (!w) { perror("malloc"); exit(1); }
    w->vtbl = vt;
    w->id = id;
    w->closed = 0;
    strncpy(w->label, label, sizeof(w->label) - 1);
    w->label[sizeof(w->label) - 1] = '\0';
    return w;
}

// 위젯 삭제
static void widget_destroy(Widget *w) {
    free(w);
}
```

- 스크린 함수
```C
static void screen_add(Screen *s, Widget *w) {
    if (s->count < MAX_WIDGETS) s->items[s->count++] = w;
}

// "dispatch"는 "무언가를 목적지로 보내어 처리하는 행위"
// 스크린의 모든 위젯에 이벤트 코드를 전달 — 각 위젯의 on_event를 호출
static void screen_dispatch(Screen *s, int code) {
    for (int i = 0; i < s->count; i++) {
        Widget *w = s->items[i];
        w->vtbl->on_event(w, code);
    }
}

// 스크린에 있는 모든 위젯들 렌더
static void screen_render(Screen *s) {
    for (int i = 0; i < s->count; i++) {
        Widget *w = s->items[i];
        w->vtbl->render(w);
    }
}
```

- 기타 함수
```C
// 상태 메시지 문자열 생성 (힙)
// ※ Widget 크기로 malloc + 0xAB로 덮어써서, 해제된 위젯 청크를 오염시키는 UAF 재현 장치
static char *app_build_status(const char *text) {
    char *msg = malloc(sizeof(Widget));
    if (!msg) exit(1);
    /* [테스트용 연출] 재사용한 메모리를 0xAB 로 '일부러' 덮어써서 오염시킨다.
     * 실무라면 다른 기능이 우연히 이 자리를 덮어쓰겠지만, 여기서는 UAF 크래시를
     * 매번 똑같이(결정적으로) 재현하기 위해 인위적으로 채운다.
     * glibc(리눅스) 환경 (tcache)에서만 유효하다. 환경&상황에 따라 msg는 새로운 주소로 할당될 수 있다.
     */
    memset(msg, 0xAB, sizeof(Widget));
    snprintf(msg, sizeof(Widget), "STATUS: %s", text);
    return msg;
}
```


- 메인 함수
```C
int main(void) {
    Screen s = { .count = 0 };

    screen_add(&s, widget_new(&LABEL_VT,  10, "Welcome"));
    screen_add(&s, widget_new(&BUTTON_VT, 11, "OK"));
    screen_add(&s, widget_new(&DIALOG_VT, 12, "Are you sure?"));  /* items[2] */
    screen_add(&s, widget_new(&BUTTON_VT, 13, "Cancel"));

    printf("frame 1:\n");
    screen_render(&s);
    screen_dispatch(&s, 1);

    /* TODO 닫힌(closed) 위젯을 여기서 정리(free + 해당 슬롯 NULL)할 필요가 있음 */

    char *status = app_build_status("dialog closed");
    printf("%s\n", status);

    printf("frame 2:\n");
    screen_render(&s);

    free(status);
    for (int i = 0; i < s.count; i++) free(s.items[i]);
    return 0;
}
```

1. 스크린 생성 후, 위젯 생성 및 스크린에 저장
	1. 라벨
	2. 버튼1
	3. 다이얼로그
	4. 버튼2

2. 프레임 1
	1. 위젯 렌더링
	2. 위젯 디스패치(처리) **← 원인 지점**: 다이얼로그가 free 되지만 `items[2]`는 그대로 (dangling)

3. 상태 메세지 생성 및 출력 **← 오염 지점**: 방금 free 된 청크를 재사용해 `vtbl` 자리를 "STATUS: …" 문자열로 덮어씀

4. 프레임 2 **← 실제 크래시 지점 (SIGSEGV)**
	1. 라벨
	2. 버튼1
	3. 삭제된 다이얼로그 → `items[2]->vtbl->render` 역참조에서 죽음
	4. 버튼2 (여기까지 못 옴)

5. 상태 메세지 메모리 해제, 스크린 해제

원인(2)과 증상(4) 사이에 3이 끼어 있어서, 크래시 지점만 봐서는 원인이 안 보인다.


## gdb로 잡기

빌드: `make gdb NAME=01_use_after_free` (= `gdb ./build/01_use_after_free`)

### 1) 죽은 자리 확인 — `run` → `bt`
```
(gdb) run
Program received signal SIGSEGV, Segmentation fault.
screen_render (s=0x7ffe710f5260) at bug.c:123
123	        w->vtbl->render(w);
(gdb) bt
#0  screen_render (s=0x7ffe710f5260) at bug.c:123
#1  0x00005ba35a4996b8 in main () at bug.c:166      ← 두 번째 screen_render (frame 2)
```
frame 2 렌더 루프 안, `w->vtbl->render(w)`에서 죽었다. `w->vtbl`을 읽어서 그 안의 `render`를 꺼내는 곳이다.

### 2) 어떤 위젯인지 — `print i`, `print w`, `print *w`
```
(gdb) print i
$1 = 2                                              ← items[2] = 다이얼로그 슬롯
(gdb) print w
$2 = (Widget *) 0x5ba3626f2300
(gdb) print *w
$3 = {vtbl = 0x203a535554415453, id = 1818323300, closed = 1663068015,
      label = "losed\000", '\253' <repeats 18 times>}
(gdb) print &DIALOG_VT
$5 = (const VTable *) 0x5ba35a49bd70 <DIALOG_VT>    ← 원래 있어야 할 값
```
`vtbl`이 `DIALOG_VT` 주소가 아니라 `0x203a535554415453`. `label` 뒤쪽은 `\253`(=0xAB)로 채워져 있다 — `app_build_status`의 memset 흔적.

### 3) 왜 그 값인지 — `x/s w`
```
(gdb) x/s w
0x5ba3626f2300:	"STATUS: dialog closed"
(gdb) up
(gdb) print status
$8 = 0x5ba3626f2300 "STATUS: dialog closed"        ← main의 status와 같은 주소
```
위젯 메모리를 문자열로 읽으니 상태 메시지가 나온다. `0x203a535554415453`을 리틀엔디언 바이트로 풀면 `53 54 41 54 55 53 3a 20` = `"STATUS: "`. 즉 `vtbl` 8바이트가 문자열 앞 8글자로 덮인 것이고, `status`와 `w`가 같은 주소라는 게 "free 된 청크를 재사용했다"는 증거다.

### 4) 슬롯이 왜 남아 있는지 — `print s->items[2]`
```
(gdb) print s->items[2]
$6 = (Widget *) 0x5ba3626f2300
(gdb) print s->items[2] == w
$7 = 1
```
Screen 배열이 해제된 주소를 그대로 들고 있다 (dangling pointer).

### 5) 누가 언제 free 했는지 — `break widget_destroy`
```
(gdb) break widget_destroy
(gdb) run
Breakpoint 1, widget_destroy (w=0x5cac3259d300) at bug.c:105
(gdb) bt
#0  widget_destroy (w=0x5cac3259d300) at bug.c:105
#1  dialog_on_event (self=0x5cac3259d300, code=1) at bug.c:130
#2  screen_dispatch (s=0x7ffe647043d0, code=1) at bug.c:116
#3  main () at bug.c:158                             ← 첫 번째 dispatch (frame 1 직후)
(gdb) print *w
$2 = {vtbl = 0x5cabf2e42d70 <DIALOG_VT>, id = 12, closed = 1, label = "Are you sure?..."}
(gdb) finish
(gdb) up
(gdb) print s->items[2]
$3 = (Widget *) 0x5cac3259d300                       ← free 직후에도 슬롯은 그대로
```
free 하는 쪽은 `dialog_on_event` → `screen_dispatch`. free 직전엔 멀쩡한 Dialog #12였고, free 직후에도 `items[2]`는 안 지워졌다.

### 정리
| 단계 | 함수 | 무슨 일 |
|---|---|---|
| 원인 | `dialog_on_event` | `free(self)` 하지만 Screen 배열은 손 못 댐 (self만 알기 때문) |
| 오염 | `app_build_status` | 같은 크기 malloc → 같은 청크 재사용 → `vtbl` 자리에 "STATUS: " 덮어씀 |
| 증상 | `screen_render` | `items[2]->vtbl->render` → 쓰레기 주소 역참조 → SIGSEGV |


### dialog_on_event()와 screen_dispatch()함수에서 문제
- dialog_on_event()함수에서 해당 위젯 closed = 1 과 위젯 삭제를 같이 실행한다.
- 해당 위젯은 삭제(메모리 해제)되었지만, 스크린의 해당 위젯 포인터는 쓰레기 값(해제된 메모리)를 가리키게 된다.
- dangling 포인터만으로는 바로 안 죽는다. `app_build_status`가 같은 크기로 malloc 해서 그 청크를 돌려받고 `vtbl` 자리를 덮어쓰기 때문에, 다음 렌더에서 `vtbl->render`를 읽는 순간 죽는다.

**따라서 위젯의 삭제와 위젯 포인터의 처리를 동시에 실행해야 한다.** ("해제 = 소유 포인터 무효화")


### dialog_on_event()와 screen_dispatch() 수정
- 핸들러(`dialog_on_event`)는 `self`만 알고 Screen 배열도 인덱스도 모르므로 슬롯을 지울 수 없다 → 핸들러는 `closed = 1` 표시만
- 소유자인 Screen이 `on_event`가 돌아온 뒤 `closed`를 보고 정리:
	- screen_dispatch()함수에서 해당 위젯의 closed == 1 이면
		- 해당 위젯 삭제
		- 스크린의 해당 위젯 포인터 = NULL
- free를 핸들러에 남겨 두면 `on_event`가 돌아온 시점에 `w`가 이미 해제된 메모리라 `w->closed`를 읽는 것 자체가 UAF다. 그래서 free의 위치를 옮겨야 한다.

#### 추가 수정
- screen_dispatch()와 screen_render()함수 실행 중 해당 위젯이 NULL이면 다음 위젯 실행 (`continue`. `return`이면 그 뒤 위젯이 안 그려짐)


정답 코드
```C
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct Widget Widget;

typedef struct {
    void (*render)(Widget *self);
    void (*on_event)(Widget *self, int code);
} VTable;

struct Widget {
    const VTable *vtbl;
    int id;
    int closed;
    char label[24];
};

#define MAX_WIDGETS 8
typedef struct {
    Widget *items[MAX_WIDGETS];
    int count;
} Screen;

/* ── 위젯 종류별 동작 ─────────────────────────────────────────── */
static void button_render(Widget *self) {
    printf("  [Button #%d] \"%s\"\n", self->id, self->label);
}
static void label_render(Widget *self) {
    printf("  Label #%d: %s\n", self->id, self->label);
}
static void dialog_render(Widget *self) {
    printf("  <<Dialog #%d>> %s\n", self->id, self->label);
}

static void widget_noop_event(Widget *self, int code) { (void)self; (void)code; }

/* 다이얼로그는 이벤트 코드 1(닫기)을 받으면 스스로 정리(파괴)된다 */
static void dialog_on_event(Widget *self, int code);

static const VTable BUTTON_VT = { button_render, widget_noop_event };
static const VTable LABEL_VT  = { label_render,  widget_noop_event };
static const VTable DIALOG_VT = { dialog_render, dialog_on_event  };

static Widget *widget_new(const VTable *vt, int id, const char *label) {
    Widget *w = malloc(sizeof *w);
    if (!w) { perror("malloc"); exit(1); }
    w->vtbl = vt;
    w->id = id;
    w->closed = 0;
    strncpy(w->label, label, sizeof(w->label) - 1);
    w->label[sizeof(w->label) - 1] = '\0';
    return w;
}

static void widget_destroy(Widget *w) {
    free(w);
}


/* ── Screen ──────────────────────────────────────────────────── */
static void screen_add(Screen *s, Widget *w) {
    if (s->count < MAX_WIDGETS) s->items[s->count++] = w;
}

static void screen_dispatch(Screen *s, int code) {
    for (int i = 0; i < s->count; i++) {
        Widget *w = s->items[i];
        if (w == NULL)          // 원래 없었음
            continue;
        w->vtbl->on_event(w, code);
        if (w->closed == 1) {   // 원래 없었음: 소유자(Screen)가 해제 + 슬롯 무효화
            widget_destroy(w);
            s->items[i] = NULL;
        }
    }
}


static void screen_render(Screen *s) {
    for (int i = 0; i < s->count; i++) {
        Widget *w = s->items[i];
        if (w == NULL)          // 원래 없었음
            continue;
        w->vtbl->render(w);
    }
}


static void dialog_on_event(Widget *self, int code) {
    if (code == 1) {
        self->closed = 1;
        // widget_destroy(self);   ← 핸들러는 표시만, free는 Screen이
    }
}

static char *app_build_status(const char *text) {
    char *msg = malloc(sizeof(Widget));
    if (!msg) exit(1);
    memset(msg, 0xAB, sizeof(Widget));
    snprintf(msg, sizeof(Widget), "STATUS: %s", text);
    return msg;
}

int main(void) {
    Screen s = { .count = 0 };

    screen_add(&s, widget_new(&LABEL_VT,  10, "Welcome"));
    screen_add(&s, widget_new(&BUTTON_VT, 11, "OK"));
    screen_add(&s, widget_new(&DIALOG_VT, 12, "Are you sure?"));  /* items[2] */
    screen_add(&s, widget_new(&BUTTON_VT, 13, "Cancel"));

    printf("frame 1:\n");
    screen_render(&s);
    screen_dispatch(&s, 1);

    char *status = app_build_status("dialog closed");
    printf("%s\n", status);

    printf("frame 2:\n");
    screen_render(&s);

    free(status);
    for (int i = 0; i < s.count; i++) free(s.items[i]);
    return 0;
}
```

검증: 정상 종료(0), frame 2에 Label #10 · Button #11 · Button #13 출력, valgrind clean.
