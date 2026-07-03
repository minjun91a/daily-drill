[← 메인 인덱스](../README.md)

# 🐧 Linux (실무) 트랙

코드 검증 드릴(SQL·pandas·Python)과 **별도로**, **Linux를 손으로** 기초부터 레벨 순서로 익히는 트랙. 예측 → 대조 → 검증 → 직접에, 기초라서 각 명령에 **개념 씨앗**(1~2줄)을 얹는다. 트리거: **"오늘 리눅스"**.

> 운영체제 *원리* 자체(프로세스·메모리·스케줄링 이론)는 **별개 트랙** → [`../os-theory/`](../os-theory/README.md). 여기(Linux)는 그 개념들을 *명령으로 손에 익히는* 실무 쪽.

## ⚙️ 연습 환경 — WSL2 준비 완료 ✅
- **WSL2 + Ubuntu 24.04 이미 설치·동작** (사용자 `min`). **재부팅·설치 불필요.** → **전 레벨(L0~L10) 진짜 리눅스로 진행 가능** (권한·프로세스·sudo 실제 동작 확인 완료: `chmod 640 → -rw-r-----`, `ps -e` OK).
- **실행법**: 터미널/PowerShell에서 `wsl` 입력, 또는 시작 메뉴 → **Ubuntu**.
- (Git Bash로도 L0~L3·L6~L8은 되지만, 이제 굳이 안 나눠도 됨.)

## 📚 로드맵 (L0 → L10)
| L | 주제 | 핵심 명령 | 숨은 OS 개념 | 환경 | 상태 |
|:-:|------|-----------|--------------|:---:|:---:|
| [0](L0-navigation.md) | 탐색·이동 | `pwd ls cd` 경로(`~ . ..`) | 파일시스템 트리 | Git Bash OK | ✅ |
| 1 | 파일 조작 | `mkdir cp mv rm cat less` | 경로·inode | Git Bash OK | ⬜ |
| 2 | 검색·집계 | `grep find wc sort uniq` | 텍스트 스트림 | Git Bash OK | ⬜ |
| 3 | 파이프·리다이렉션 | `\| > >> 2> /dev/null` | stdin/stdout/stderr | Git Bash OK | ⬜ |
| 4 | 권한·소유권 | `chmod chown` rwx `644/755` | OS 권한 모델 | **WSL ✅** | ⬜ |
| 5 | 프로세스 | `ps top kill &` `jobs fg/bg` | 프로세스·시그널 | **WSL ✅** | ⬜ |
| 6 | 텍스트 처리 | `cut tr sed awk`(기초) | 스트림 가공 | Git Bash OK | ⬜ |
| 7 | 환경·쉘 | `env export PATH .bashrc` | 환경변수·쉘 | Git Bash OK | ⬜ |
| 8 | 쉘 스크립트 | shebang 변수 `if/for $1` 종료코드 | 자동화 | Git Bash OK | ⬜ |
| 9 | 네트워크·원격 | `ssh scp curl ss` | 포트·원격 | WSL ✅ | ⬜ |
| 10 | 실무 | `cron tar gzip apt systemctl` | 서비스 운영 | **WSL ✅** | ⬜ |

각 레벨 1~2일. 완료하면 상태 ⬜ → ✅, 아래에 `linux/L<n>-<주제>.md` 링크 추가.

## 하루 흐름
개념 씨앗(1~2줄) → **예측**(명령이 낼 결과) → 실행·대조 → **검증 함정**(위험/틀린 명령의 문제 찾기) → **직접**(스펙대로 명령·한 줄 스크립트 작성) → 오답만 레벨 파일에 기록.

> ⚠️ **안전**: 위험 명령(`rm -rf` 등)은 **예측·분석만**. 실제 실행은 버려도 되는 임시 폴더에서만.

## 진행 기록
| 레벨 | 파일 | 완료일 |
|------|------|--------|
| L0 탐색·이동 | [L0-navigation.md](L0-navigation.md) | 2026-07-03 |
