[← Linux 트랙](README.md)

# 🐧 Linux L0 — 탐색·이동 (2026-07-03 · Git Bash)

**개념 씨앗**: `pwd`(현재 경로) · `ls`(목록) · `cd 경로`(이동). 경로 기호 `/`(루트) `~`(홈) `.`(현재) `..`(부모). 절대경로(`/`·`~`로 시작, 어디서든 같은 곳) vs 상대경로(현재 위치 기준).

트리:
```
/home/mint/
├── projects/  ├── web/   └── data/
└── docs/
```

## 예측 — cd 연쇄 (시작: `/home/mint/projects/web`)
| # | 명령(이어서) | 내 예측 | 실제 | |
|:-:|---|---|---|:-:|
| 1 | `cd ..` | `/home/mint/projects` | 같음 | ✅ |
| 2 | `cd data` | `/home/mint/projects/data` | 같음 | ✅ |
| 3 | `cd ../../docs` | `/home/mint/docs` | 같음 | ✅ |
| 4 | `cd ~` | **`/home`** | `/home/mint` | ❌ |

**오답 4 — `~` vs `/home`**: `~` = 내 홈 폴더(`/home/mint`) 하나. `/home`은 그 홈들을 담는 **부모**. `cd ~`는 내 집으로 가지, 한 칸 위 동네(`/home`)로 가는 게 아님.
- 증거: `echo ~` = `/c/Users/최민준` — `~`는 늘 "특정 사용자 전용 폴더"의 전체 경로.

## 개념 오해 교정 — `~`는 최상위가 아니다
> 내 질문: "cd ~ 는 폴더트리 최상위로 가는 거 아냐?" → **❌**
- **최상위 = `/`(루트).** `cd /` → `pwd` = `/`.
- `~` = `/home/mint` = 루트에서 **두 칸 아래**(`/` → `home` → `mint`)의 내 홈.
```
/              ← 최상위 (cd /)
└── home/
    └── mint/  ← ~ (cd ~) · 꼭대기 아님, 두 칸 아래
```
- 터미널이 홈에서 시작해 "기준점"처럼 느껴질 뿐. 꼭대기로는 `cd /`.

## 직접 — `/home/mint/docs`에서 `web`으로 (3방법)
| 방법 | 내가 쓴 명령 | 판정 |
|---|---|---|
| (a) 상대 | `cd projects/web` | ❌ docs 안엔 projects 없음(형제) → **`cd ../projects/web`** |
| (b) 절대 | `cd /home/mint/projects/web` | ✅ |
| (c) `~` | `cd ~/projects/web` | ✅ (`~=/home/mint` 정확 적용) |

실행 확인: `cd projects/web` → '그런 폴더 없음' / `cd ../projects/web` → `/home/mint/projects/web` 도착.

## 그날의 핵심
- 상대경로는 **지금 위치 기준**. 이동 전 "지금 어디? 목표가 자식/형제/부모?"를 먼저 판단.
- 목표가 **형제**면 `..`로 부모까지 올라갔다 내려온다(`../형제/...`).
- 세 개는 전부 다른 곳: `~`(내 홈=`/home/mint`) ≠ `/home`(홈들의 부모) ≠ `/`(최상위 루트).

---
[← Linux 트랙](README.md)
