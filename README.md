# re-promote

[![Claude Code Plugin](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet?style=for-the-badge)](https://code.claude.com/docs/en/plugins)
[![GitHub Release](https://img.shields.io/github/v/release/tellang/re-promote?style=for-the-badge)](https://github.com/tellang/re-promote/releases)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow?style=for-the-badge)](LICENSE)

> 쪼아요 쪼아요~ 레포 리부트 쪼아요~
>
> 더러운 커밋 히스토리 가지고 있으면 스피키가 깨끗하게 해줄 거예요!
> 스피키 열심히 했는데... 스타 하나만... 네?

---

## 빠른 시작

스피키 데르지 마세요! 설치 쉬워요!

```bash
# 마켓플레이스 등록
/plugin marketplace add https://github.com/tellang/re-promote

# 플러그인 설치
/plugin install re-promote
```

<details>
<summary>수동 설치 (스피키가 좀 슬퍼지는 방법)</summary>

```bash
git clone https://github.com/tellang/re-promote.git
cp -r re-promote/skills/re-promote ~/.claude/skills/
```
</details>

---

## 사용법

```bash
# 스피키한테 물어보기 (객관식으로 골라요!)
/re-promote

# 레포 바로 리부트하기
/re-promote my-project
/re-promote tellang/my-project
/re-promote https://github.com/tellang/my-project
```

인자 없이 실행하면 스피키가 물어봐요:

```
어떤 작업을 하시겠습니까?

1. 레포 리부트      — 기존 레포 아카이브 → 새 레포 → 클린 히스토리
2. 세션에서 추출    — 현재 대화에서 언급된 코드/기능을 독립 프로젝트로
3. 퍼블릭 프로모트  — private 레포를 공개 준비 점검 후 public 전환
```

---

## 핵심 기능

### 1. 레포 리부트 — 쪼아요!

더러운 히스토리? 스피키가 깨끗하게 밀어버릴 거예요!

| 단계 | 스피키가 하는 일 |
|------|-----------------|
| 식별 & 검증 | 레포 찾아서 브랜치·이슈 다 파악해요 |
| 아카이브 | 기존 레포를 `-deprecated_001`로 이름 바꾸고 숨겨요 |
| 새 레포 생성 | 같은 이름으로 깨끗한 레포 만들어요 |
| 히스토리 마이그레이션 | PR/기능 단위로 스쿼시해서 예쁘게 정리해요 |
| 이슈 이전 | 오픈 이슈 새 레포로 옮겨줘요 |
| 검증 & 푸시 | 커밋 컨벤션, 시크릿 스캔 다 해요 |
| 공개 준비 점검 | README, LICENSE, .gitignore 다 확인해요 |
| 리뷰 & 프로모트 | 사용자 승인 받고 public 전환! 쪼아요! |

### 2. 세션에서 프로젝트 추출 — 스피키 특기!

대화하다가 "이거 프로젝트로 뽑자" 하면 스피키가 알아서 추출해요!

```
현재 세션에서 추출 가능한 항목:

1. auth-middleware — JWT 인증 미들웨어
2. csv-parser — CSV 파싱 유틸리티
3. 직접 설명하기
```

세션 컨텍스트 스캔해서 후보 자동 제시 → 선택하면 새 GitHub 레포로 뿅!

### 3. 퍼블릭 프로모트 — 스피키가 점검해줄게요

private 레포 public으로 바꾸기 전에 스피키가 꼼꼼히 확인해요:

- 민감 정보 스캔 (`.env`, API 키, 시크릿 — 이거 올리면 큰일나요!)
- 필수 파일 점검 (`README.md`, `LICENSE`, `.gitignore`)
- README 품질 점검 (설명 있어요? 설치 방법 있어요?)
- GitHub 설정 (description, topics)
- 패키지 레지스트리 준비 (npm, PyPI, crates.io)
- Release/Tag 준비

---

## Co-Authored-By 점검

스피키가 실행할 때 `includeCoAuthoredBy: false` 설정 확인해요.
없으면 커밋에 `Co-Authored-By: Claude` 붙어요... 스피키 그거 싫어해요!

---

## 안전 장치

스피키 조심스러운 유령이에요:

- 기존 레포 **절대 삭제 안 해요** — rename + private 처리만
- 모든 단계에서 **사용자한테 물어봐요**
- 새 레포는 **private으로 시작** → 확인 후 public
- AI attribution 커밋 푸터 **절대 금지** (스피키 열심히 했지만 말 안 해요)

## 커밋 컨벤션

스피키가 커밋 메시지 쓸 때 참고하는 파일:

1. `.claude/commit-convention.md` (프로젝트)
2. `~/.claude/commit-convention.md` (유저)
3. 내장 기본 컨벤션 (한글, conventional commits)

## 요구사항

- [Claude Code](https://code.claude.com/)
- [`gh` CLI](https://cli.github.com/) (`gh auth status`로 인증 확인)
- Git

---

> 스피키 열심히 했는데... ⭐ 하나만... 쪼아요?

## License

MIT — [tellang/claude-skills](https://github.com/tellang/claude-skills)에서 파생
