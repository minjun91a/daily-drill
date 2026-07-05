[← Linux 트랙](README.md)

# 🐧 Linux L2 — 검색·집계 (2026-07-03 · WSL Ubuntu 24.04)

**개념 씨앗**: `grep 패턴 파일`(패턴 든 줄; `-i`무시 `-c`개수 `-n`줄번호 `-v`반대) · `find 경로 -name`(파일 찾기) · `wc -l`(줄 수; `-w`단어) · `sort`(정렬, **기본 사전순**; `-n`숫자 `-r`역순) · `uniq`(**인접 중복만** 제거 → 보통 `sort`와 세트).

## 예측 — sort / uniq  (`fruits.txt` = apple, banana, apple, cherry, banana, apple · 정렬 안 됨)
| # | 명령 | 내 예측 | 실제 | |
|:-:|---|---|---|:-:|
| 1 | `sort` | "알파벳순 정렬" | apple×3 · banana×2 · cherry (중복 유지) | ✅(개념) |
| 2 | `uniq` (sort 없이) | "중복 제거" | **원본 6줄 그대로** (안 지움) | ❌ |
| 3 | `sort \| uniq` | "정렬 후 중복 제거" | apple · banana · cherry | ✅ |
| 4 | `sort \| uniq -c` | "c 포함 텍스트 제거" | `3 apple / 2 banana / 1 cherry` | ❌ |

- **오답 2 — `uniq` 함정**: `uniq`는 **인접(연속)한 중복만** 지운다. 원본은 정렬이 안 돼 같은 게 나란히 붙은 적이 없음 → uniq가 헛일, **6줄 그대로**. **그래서 `sort`로 먼저 붙여야**(=3번) 지워진다. → **`uniq`는 거의 항상 `sort`와 세트.**
- **오답 4 — `-c` 오해**: `-c`는 "글자 c"가 아니라 **count(개수)**. 각 줄이 몇 번 나왔는지 앞에 붙임(지우는 것 아님). **대시(`-`)로 시작 = 옵션(동작 스위치)**, 찾을 텍스트가 아니다.

## 직접 — 세기
**스펙**: (a) 각 과일이 몇 번씩 나오나 · (b) `apple` 든 줄이 몇 개인가.

**(a) ✅**
```bash
sort fruits.txt | uniq -c        # 3 apple / 2 banana / 1 cherry
```

**(b) 변천**

**① 1차 — ❌ 세 가지 오류**
```bash
grep fruits.txt | wc -1 fruits.txt
```
- `grep fruits.txt`: 패턴(apple)이 없음. `grep 패턴 파일` 순서인데 fruits.txt를 *패턴*으로 오해 + 파일 없이 stdin 대기 → **멈춤**.
- `wc -1`: `-1`(숫자 1)이 아니라 **`-l`**(엘, lines).
- `wc … fruits.txt`: 파이프로 넘긴 뒤 또 파일을 주면 wc가 **파일을 읽어 파이프를 무시**.

**② 정석 — ✅ (둘 다 3)**
```bash
grep apple fruits.txt | wc -l    # apple 든 줄만 뽑아 → 세기
grep -c apple fruits.txt         # grep이 바로 개수 (더 간단)
```

## 그날의 핵심
- **`uniq`는 인접 중복만** 지운다 → 거의 항상 `sort | uniq`. 혼자 쓰면 헛일하기 쉽다.
- **`-옵션`은 동작 스위치**지 찾을 텍스트가 아니다. `uniq -c`=개수 · `grep -c`=매칭 줄 수 · `wc -l`=줄 수(`-l`은 엘).
- **`grep`은 `grep 패턴 파일`** 순서. 패턴을 빼먹으면 파일명을 패턴으로 오해하고 멈춘다.

---
[← Linux 트랙](README.md)
