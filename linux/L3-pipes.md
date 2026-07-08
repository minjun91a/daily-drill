[← Linux 트랙](README.md)

# 🐧 Linux L3 — 파이프·리다이렉션 (2026-07-08 · WSL Ubuntu 24.04)

**개념 씨앗**: 프로그램마다 입출력 통로 3개(파일 디스크립터) — **stdin(0)** 입력 · **stdout(1)** 정상출력 · **stderr(2)** 에러출력. `|`(파이프: 앞 stdout → 뒤 stdin) · `>`(stdout을 파일로, **덮어씀**) · `>>`(이어붙임) · `2>`(stderr만) · `/dev/null`(버리는 구멍). **핵심: 그냥 `>`는 사실 `1>`(stdout만). 에러(2)는 안 잡힌다.**

## 예측 — 리다이렉션·파이프  → **4/4 힌트 없이 통과 ✅**
| # | 명령 | 내 예측 | 실제 | |
|:-:|---|---|---|:-:|
| A | `>` 두 번 (apple→banana) | banana만 | `banana` | ✅ |
| B | `>>` 이어쓰기 | apple, banana 둘 다 | `apple / banana` | ✅ |
| C | `ls /없는폴더 > out.txt` — 에러도 잡히나? | 에러는 화면, out.txt 빔 | stderr에 에러, out.txt `[]` | ✅ |
| D | `echo … \| sort \| uniq` | apple, banana, cherry | 동일 | ✅ |

- **C가 핵심**: `>`는 **stdout(1)만** 돌린다 → 에러(stderr, 2)는 파일에 안 담기고 화면에 그대로. 실무에서 `cmd > log.txt` 하고 "에러가 왜 로그에 없지?" 하는 실수의 정체. 잡으려면 `2>` 또는 `2>&1`.

## 검증 함정 — `sort data.txt > data.txt` (데이터 증발)  → **3/3 오답**
- **내 예측 (그대로)**: "1. 정렬된 목록만 남는다. 2. sort가 먼저 실행되고 그다음 정렬된 데이터로 교체. 3. `sort data.txt >> data.txt`로 고침." → **셋 다 ❌.**
- **실제** (WSL 실행): `data.txt` = **빈 파일(0줄), 원본 증발.** 이유 — **쉘은 명령을 실행하기 *전에* 리다이렉션부터 처리**한다. `> data.txt`가 sort 실행 전에 파일을 "쓰기용으로 열며 0으로 잘라(truncate)" → sort가 읽을 땐 이미 텅 빔 → 빈 출력. **"sort 먼저"가 아니라 "`>` 먼저"**가 정답.
- **`>>` 제안도 실패**: 안 자르니 원본은 살지만, 안 정렬된 원본 뒤에 정렬본을 이어붙여 6줄(`banana apple cherry` + `apple banana cherry`) → 목표(정렬본만) 실패.
- **안전한 정답**: `sort data.txt > sorted.txt && mv sorted.txt data.txt` (임시파일 경유) 또는 `sort data.txt -o data.txt` (sort의 안전 옵션). **규칙: 읽는 파일 = `>` 대상 파일이 같으면 위험** → temp+mv 기본.

## 직접 — 파이프+리다이렉션 조립

### A) `fruits.txt`를 정렬+중복제거 → `clean.txt` 저장 (원본 유지)
**① 1차 — ❌ mv 오적용 + uniq 누락**
```bash
sort fruits.txt > clean.txt && mv clean.txt fruits.txt
```
방금 배운 temp+mv 패턴을 가져왔으나, 그건 **같은 파일 되쓰기**용. 여기 대상은 **다른 파일(clean.txt)** → mv 불필요한데 오히려 clean.txt를 없애고 원본까지 덮음. + `uniq` 없어 중복 안 지워짐.

**② 2차 — ❌ 파이프·리다이렉션 순서 엉킴**
```bash
sort fruits.txt > clean.txt fruits.txt | uniq
```
`> clean.txt`를 중간에 둬 sort 출력이 파일로 새고(뒤 `| uniq`는 헛됨), `fruits.txt`가 두 번 나와 sort가 2배로 읽음 → clean.txt 8줄.

**③ 3차 — ✅ 합격**
```bash
sort fruits.txt | uniq > clean.txt      # clean.txt = apple, banana, cherry / 원본 그대로
```
파이프로 **정렬→중복제거**를 잇고, **맨 끝**에 `>`로 저장. 리다이렉션은 언제나 마지막 고리.

### B) `ls /home /없는폴더` — 에러(stderr)만 버리고 정상출력만 보기
**① 이동 명령 나열 — ❌ 스펙 무관** (`ls; cd /home; ls; cd /없는폴더`)
**② `> out.txt` — ❌** 정반대. `>`=stdout이라 정상출력이 파일로 숨고 에러는 화면에 그대로.
**③ `> /dev/null` — ❌** `/dev/null`(버리는 구멍)은 맞혔으나 여전히 `>`(=1, stdout)라 정상출력을 버림.
**④ `2> dev/null` — ❌** 스트림(2)·구멍은 맞았으나 **슬래시 누락**. `dev/null`은 현재 폴더의 상대경로 → `dev` 폴더 없어 리다이렉트 실패.
**⑤ ✅ 합격**
```bash
ls /home /없는폴더 2>/dev/null          # /home 목록만, 에러 버려짐
```

## 그날의 핵심
- **fd 번호가 스트림을 정한다**: 그냥 `>`는 `1>`(stdout). 에러를 버리려면 반드시 **`2>`**. (예측 C에서 이해한 "`>`는 에러 못 잡음"을 *고칠 때* 적용하는 게 관건이었다.)
- **쉘은 명령보다 리다이렉션을 먼저 처리** → `cmd > 읽는파일`은 명령이 읽기도 전에 파일을 비운다(데이터 증발). 같은 파일 되쓰기는 `> temp && mv` 또는 `-o`.
- **파이프라인은 왼→오 컨베이어**: 명령들을 `|`로 다 이은 뒤 **맨 끝에** `>`로 저장. 리다이렉션을 중간에 끼우면 뒤 명령이 헛돈다.
- **패턴은 조건 맞을 때만 쓴다**: temp+mv는 "같은 파일 되쓰기"용 — 다른 파일에 저장할 땐 불필요. 조건 안 보고 패턴만 복사하면 스펙을 깬다.
- **`/dev/null`의 앞 `/`** = 절대경로(루트의 특수 장치). 빼면 "지금 여기 dev 폴더"라는 상대경로 → L0의 절대/상대경로 감각이 그대로 재등장.

---
[← Linux 트랙](README.md)
