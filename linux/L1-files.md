[← Linux 트랙](README.md)

# 🐧 Linux L1 — 파일 조작 (2026-07-03 · WSL Ubuntu 24.04)

**개념 씨앗**: `mkdir`(폴더) · `touch`(빈 파일) · `cp`(복사 → 원본 남음) · `mv`(이동/이름변경 → 원본 사라짐) · `cat`(내용) · `rm`(삭제 ⚠️ 휴지통 없이 영구). 핵심 대비: **`cp`(원본 O) ↔ `mv`(원본 X)**.

## 예측 — cp vs mv (합격 3/3)
```bash
mkdir project; cd project
touch a.txt b.txt
mkdir backup
cp a.txt backup/        # 복사 → a.txt가 양쪽에
mv b.txt c.txt          # 이동 → b.txt가 c.txt로
```
| 질문 | 내 예측 | 실제 | |
|---|---|---|:-:|
| `ls` (project/) | `a.txt backup c.txt` | 같음 | ✅ |
| `ls backup/` | `a.txt` | 같음 | ✅ |
| a.txt 개수 / b.txt | 2개 / 0개 | 같음 | ✅ |

`cp`는 원본을 남겨 a.txt가 project·backup **양쪽(2개)**, `mv`는 b.txt를 c.txt로 **이름변경**(b.txt 0개).

## 검증 함정 — `rm -rf ~ /tmp_stuff` (공백 하나의 재앙)
- **문제**: 홈의 `tmp_stuff`만 지우려는 의도.
- **내 판정 (그대로)**: "tmp_stuff 및 상위 모든 폴더까지 전체 삭제 / 의도와 다름 / `rm -rf tmp_stuff`로 수정" → **위험은 감지(✅), 이유는 부정확(❌)**.
- **실제**: `rm`은 위로(부모로) 올라가며 지우지 않는다. **공백이 명령을 대상 2개로 쪼갬**:
  - `~` = 홈 전체(`/home/min`) ← 이게 통째로 삭제되는 재앙
  - `/tmp_stuff` = 루트의 폴더 (보통 없음)
  - 안전 확인(echo): `대상1 [/home/min]   대상2 [/tmp_stuff]`
- **교정**: 공백 제거 → `rm -rf ~/tmp_stuff` (대상 하나 `[/home/min/tmp_stuff]`).
- **교훈**: `rm`은 **공백으로 나뉜 각 덩어리를 따로** 삭제 대상으로 본다. `~`가 홀로 떨어지면 = 홈 전체. 치기 전 대상을 눈으로 확인 / `ls`로 먼저 / `rm -i`(하나씩 확인).

## 직접 — 파일 만들고 복사·이름변경
**스펙**: `work` 폴더 → 안에 `memo.txt` → `memo_backup.txt`로 복사 → `work`를 `archive`로 이름변경. `ls archive/` 예측까지.

**내 명령 변천**

**① 1차 — ❌ touch가 엉뚱한 곳에**
```bash
touch memo.txt work/
```
`memo.txt`가 **홈(work 밖)에** 생성 + `work/`는 시간만 갱신 → 이후 `cp work/memo.txt ...`가 *No such file* 로 실패. (touch를 cp/mv처럼 "원본→대상"으로 오해)

**② 2차 — ✅ 파일을 폴더 안에 = 경로로**
```bash
mkdir work
touch work/memo.txt
cp work/memo.txt work/memo_backup.txt
mv work archive
```
- **실행 결과**: `ls archive/` → `memo.txt  memo_backup.txt` (예측 일치 ✅)
- **교훈**: **`touch` ≠ `cp`/`mv`.** cp·mv는 "원본→대상"(2역할), `touch`는 "만들 파일들의 목록"(각각 그 자리에 생성). 파일을 폴더 안에 만들려면 폴더는 **파일의 경로**로 붙인다 → `work/memo.txt`.

## 그날의 핵심
- `cp`(복사·원본 남음) ↔ `mv`(이동·원본 사라짐). 대상이 폴더면 그 안으로.
- **`rm`은 공백으로 대상을 쪼갠다** — `~` 하나 떨어지면 홈 전체 삭제. 치기 전 대상 확인.
- **명령마다 인자 받는 법이 다르다.** `touch`는 "원본→대상"이 아니라 "만들 경로들의 목록". 한 명령 문법을 딴 명령에 이식하면 틀린다 (`groupby≠JOIN`과 같은 함정).

---
[← Linux 트랙](README.md)
