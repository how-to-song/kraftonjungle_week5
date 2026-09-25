<!-- generated from vault — 편집은 Obsidian에서 -->
# 코드 분석 노트

각 챌린지 폴더의 `README.md`는 Obsidian에서 작성한 분석 노트를 내보낸 것.

| 챌린지 | 버그 유형 |
|---|---|
| [01_use_after_free](01_use_after_free/README.md) |  |
| [05_null_return_deref](05_null_return_deref/README.md) |  |
| [06_null_deref](06_null_deref/README.md) |  |
| [07_stack_use_after_return](07_stack_use_after_return/README.md) |  |
| [08_uninitialized_read](08_uninitialized_read/README.md) | 초기화되지 않은 값 읽기 (uninitialized read) |
| [09_strcpy_overflow](09_strcpy_overflow/README.md) | 힙 버퍼 오버플로 (크기 계산과 복사 범위 불일치) |
| [10_realloc_dangling](10_realloc_dangling/README.md) | dangling pointer (realloc 이동 후 옛 주소 사용) → double free |
| [11_global_overflow](11_global_overflow/README.md) | 전역 버퍼 오버플로 (bump 할당 경계 미검사) → 인접 전역(arena_off) 훼손 |
| [12_free_non_heap](12_free_non_heap/README.md) | 비힙 포인터 free (strtok가 준 내부 포인터를 해제) → invalid pointer |
| [13_linked_list_uaf](13_linked_list_uaf/README.md) | Use-After-Free (해제한 노드의 next를 읽음) |
| [14_integer_overflow_alloc](14_integer_overflow_alloc/README.md) | 정수 오버플로(int 곱셈 wrap) → 과소할당 → 힙 오버플로 |
| [15_dangling_in_struct](15_dangling_in_struct/README.md) | 구조체 멤버 dangling pointer (해제 후 s->user 사용) — UAF |
| [16_unused_cap_overflow](16_unused_cap_overflow/README.md) | 스택 버퍼 오버플로 (cap 인자를 받고도 경계 검사에 안 씀) |
| [17_ownership_uaf](17_ownership_uaf/README.md) | 소유권 이중화 → Use-After-Free / double free |
| [18_cleanup_double_free](18_cleanup_double_free/README.md) | goto 정리 경로의 double free (+ state 누수) |
| [19_realloc_shrink_overflow](19_realloc_shrink_overflow/README.md) | realloc 축소 후 len 미갱신 → 힙 범위 밖 읽기 |
| [20_vector_stale_pointer](20_vector_stale_pointer/README.md) | 성장하는 배열 원소의 주소 캐시 → realloc 후 dangling pointer |
