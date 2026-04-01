# re-promote

> Claude Code 플러그인 — 프로젝트 레포를 깨끗하게 리부트하는 자동화 도구

[![Claude Code Plugin](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet)](https://code.claude.com/docs/en/plugins)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

## 기능

### `/re-promote <repo>`

기존 레포를 아카이브하고, 같은 이름으로 새 레포를 생성하여 깨끗한 커밋 히스토리로 마이그레이션합니다.

**8단계 자동화 워크플로우:**

1. **식별 & 검증** — 레포 존재 확인, 메타데이터 캡처
2. **아카이브** — 기존 레포를 `-deprecated_NNN`으로 이름 변경 + private 전환
3. **새 레포 생성** — 같은 이름으로 private 레포 생성
4. **히스토리 마이그레이션** — 커밋을 의미 단위로 스쿼시, 모든 브랜치 마이그레이션
5. **이슈 이전** — 오픈 이슈를 새 레포로 전송
6. **검증 & 푸시** — 커밋 컨벤션 검증, 민감 정보 스캔
7. **공개 준비 점검** — README, LICENSE, .gitignore, 패키지 설정 등
8. **리뷰 & 프로모트** — 사용자 승인 후 public 전환

## 설치

### Claude Code 플러그인으로 설치 (권장)

```bash
/plugin marketplace add https://github.com/tellang/re-promote
/plugin install re-promote
```

### 수동 설치

```bash
git clone https://github.com/tellang/re-promote.git
cp -r re-promote/skills/re-promote ~/.claude/skills/
```

## 사용법

```bash
/re-promote my-project
/re-promote tellang/my-project
/re-promote https://github.com/tellang/my-project
```

## 요구사항

- [Claude Code](https://code.claude.com/) 설치
- [`gh` CLI](https://cli.github.com/) 설치 및 인증 (`gh auth status`)
- Git 설치
- 대상 레포에 대한 쓰기 권한

## 커밋 컨벤션

`re-promote`는 커밋 메시지 작성 시 컨벤션 파일을 참조합니다:

1. `.claude/commit-convention.md` (프로젝트)
2. `~/.claude/commit-convention.md` (유저)
3. 내장 기본 컨벤션 (한글, conventional commits)

## 안전 장치

- 기존 레포는 **절대 삭제하지 않음** — rename + private 처리만
- 모든 단계에서 **사용자 확인** 필요
- 새 레포는 **private으로 시작** → 검토 후 public 전환
- AI attribution 커밋 푸터 **절대 금지**

## Credits

- 원본: [tellang/claude-skills](https://github.com/tellang/claude-skills)

## License

MIT
