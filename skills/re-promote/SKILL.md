---
name: re-promote
description: "Project reboot skill: archive the old repo (rename to -deprecated_NNN, make private), create a fresh repo with the same name, migrate with clean squash history, scan and remove files that shouldn't be public (secrets, internal docs, binaries), transfer open issues, verify and push, then promote to public after user approval. Use this skill when the user wants to reboot, restart, re-launch, re-promote, or clean-slate a repository. Also supports extracting features from the current session into a standalone project, and promoting private repos to public with readiness checks."
disable-model-invocation: true
argument-hint: "<repo-name or repo-url> [options]"
---

# Re-Promote: Project Reboot

Reboot a project repository with a clean history while archiving the old one safely.

## Current context

- GitHub user: `!gh api user --jq .login`
- Current directory: `!pwd`

## Why this skill exists

Sometimes a project needs a fresh start — accumulated cruft, broken CI history, sensitive data in old commits, or just a desire for a clean slate. This skill automates the delicate process of archiving the old repo and standing up a clean replacement, ensuring nothing is lost and the transition is smooth.

## Instructions

Follow these phases in order. Each phase has a checkpoint — confirm with the user before moving to the next.

---

### Phase 0: 모드 선택 & 사전 점검

#### 0a: 인터랙티브 모드 선택

`$ARGUMENTS`가 비어있거나 레포명이 아닌 경우, AskUserQuestion으로 모드를 선택받는다:

```
어떤 작업을 하시겠습니까?

1. 레포 리부트 — 기존 레포 아카이브 → 새 레포 → 클린 히스토리
2. 세션에서 추출 — 현재 대화에서 언급된 코드/기능을 독립 프로젝트로 추출
3. 퍼블릭 프로모트 — 기존 private 레포를 공개 준비 점검 후 public 전환
```

- **1 선택** → Phase 1로 진행 (기존 리부트 플로우)
- **2 선택** → Phase 0b: 세션 추출 플로우
- **3 선택** → Phase 7로 직행 (Promote Readiness Check)

`$ARGUMENTS`에 레포명이 명시되어 있으면 이 단계를 건너뛰고 바로 Phase 1로 진행한다.

#### 0b: 세션에서 프로젝트 추출

현재 대화 컨텍스트를 분석하여 추출 가능한 항목을 식별한다:

1. **세션 스캔** — 대화에서 생성/수정한 파일, 논의한 기능, 코드 블록을 파악
2. **추출 후보 제시** — AskUserQuestion으로 선택:
   ```
   현재 세션에서 추출 가능한 항목:

   1. <항목 A> — <설명>
   2. <항목 B> — <설명>
   3. 직접 설명하기
   ```
3. **선택된 항목의 관련 파일을 수집** — 코드, 설정, 문서 등
4. **새 레포 생성** → Phase 3 (Create the New Repo)으로 분기
5. 수집한 파일을 새 레포에 커밋 → Phase 6 (Verify and Push)으로 분기

#### 0c: Co-Authored-By 설정 점검

리부트 또는 추출 시작 전에 Claude Code 설정을 확인한다:

```bash
cat ~/.claude/settings.json 2>/dev/null
```

`"includeCoAuthoredBy": false`가 **없으면** 사용자에게 권유:

```
⚠️ Claude Code 설정에 includeCoAuthoredBy: false가 없습니다.
   이 설정이 없으면 커밋에 "Co-Authored-By: Claude" 푸터가 자동 추가됩니다.

   지금 설정할까요? (Y/n)
```

동의하면:
```bash
# settings.json에 "includeCoAuthoredBy": false 추가
```

거절하면 경고만 남기고 진행한다.

---

### Phase 1: Identify and Validate

1. **Parse input**: Extract the repo name from `$ARGUMENTS`. Accept formats:
   - `repo-name` (assumes current GitHub user as owner)
   - `owner/repo-name`
   - `https://github.com/owner/repo-name`

2. **Verify the repo exists and capture metadata**:
   ```bash
   gh repo view <owner>/<repo> --json name,owner,visibility,defaultBranchRef,description,homepageUrl,repositoryTopics,hasIssuesEnabled
   ```

3. **Capture all branch names**:
   ```bash
   gh api repos/<owner>/<repo>/branches --paginate --jq '.[].name'
   ```

4. **Capture open issues count**:
   ```bash
   gh issue list --repo <owner>/<repo> --state open --json number --jq 'length'
   ```

5. **Clone the repo with all branches**:
   ```bash
   git clone --mirror <repo-url> /tmp/re-promote-<repo-name>.git
   git clone /tmp/re-promote-<repo-name>.git /tmp/re-promote-<repo-name>
   cd /tmp/re-promote-<repo-name>
   # Checkout all remote branches as local
   for branch in $(git branch -r | grep -v HEAD | sed 's/origin\///'); do
     git checkout -b "$branch" "origin/$branch" 2>/dev/null || true
   done
   ```

6. **Show the user a summary**:
   ```
   ## Re-Promote: <owner>/<repo>
   - Visibility: public
   - Default branch: main
   - Branches: main, develop, feature/xxx (총 N개)
   - Description: ...
   - Topics: ...
   - Commits on default branch: N
   - Open issues: N개
   ```

   Ask: "이 레포를 리부트합니다. 진행할까요?"

---

### Phase 2: Archive the Old Repo

1. **Determine the deprecated suffix** — sequential numbering:
   ```bash
   gh repo list <owner> --limit 1000 --json name --jq '.[].name' | grep "^<repo>-deprecated_" | sort -V | tail -1
   ```
   Extract the number, increment by 1. If none exist, start with `001`. Always zero-pad to 3 digits.

2. **Rename the repo**:
   ```bash
   gh repo rename <repo>-deprecated_<NNN> --repo <owner>/<repo> --yes
   ```

3. **Make it private**:
   ```bash
   gh repo edit <owner>/<repo>-deprecated_<NNN> --visibility private --accept-visibility-change-consequences
   ```

4. **Confirm to user**:
   ```
   Old repo archived: <owner>/<repo>-deprecated_<NNN> (private)
   ```

---

### Phase 3: Create the New Repo

1. **Create a new repo with the original name** (private first for safety):
   ```bash
   gh repo create <owner>/<repo> --private --description "<original-description>"
   ```

2. **Restore metadata**:
   ```bash
   gh repo edit <owner>/<repo> --add-topic <topic1> --add-topic <topic2> ...
   ```
   Restore homepage URL if it existed.

3. **Clone the new repo for work**:
   ```bash
   git clone https://github.com/<owner>/<repo>.git /tmp/re-promote-<repo>-new
   cd /tmp/re-promote-<repo>-new
   ```

---

### Phase 4: Migrate History (Squash & Clean)

This is the most critical phase. The goal is to create a clean, meaningful commit history across all branches.

#### 4a: Analyze and Plan (Default Branch First)

1. **Analyze the old repo's commit history on the default branch**:
   ```bash
   git -C /tmp/re-promote-<repo> log --oneline --first-parent <default-branch> | head -200
   git -C /tmp/re-promote-<repo> log --oneline --merges <default-branch> | head -100
   ```

2. **Detect natural grouping boundaries** — use this priority order:
   - **PR merge commits**: If merge commits exist, each PR is a natural squash unit
   - **Feature branches merged**: Group by merged branch names visible in merge commit messages
   - **Semantic grouping**: If no PRs/merges, analyze commit messages for related changes (same prefix, same files touched, same time window) and group them into logical features/fixes
   - **Chronological chunks**: As last resort, group by reasonable time windows

3. **Present the plan**:
   ```
   ## Proposed Commit Plan — <default-branch>

   | # | Squash Group | Original Commits | Proposed Message |
   |---|-------------|-----------------|-----------------|
   | 1 | Initial setup | abc1234..def5678 (12 commits) | chore: initial project scaffolding |
   | 2 | Auth feature | ghi9012..jkl3456 (8 commits) | feat: add user authentication |
   | ... | ... | ... | ... |

   총 N개 원본 커밋 → M개 정제 커밋으로 스쿼시됩니다.
   조정이 필요하면 말씀해주세요.
   ```

4. **Wait for user approval** before executing.

#### 4b: Execute Migration (Default Branch)

For each approved squash group, in chronological order:

```bash
cd /tmp/re-promote-<repo>-new

# For each group: checkout the boundary state from old repo and commit
git -C /tmp/re-promote-<repo> diff <group-start>^..<group-end> | git apply --allow-empty
git add -A
git commit -m "<approved commit message>"
```

If `diff | apply` fails (binary files, renames, etc.), use the snapshot approach:
```bash
# Checkout the target commit in the old repo
git -C /tmp/re-promote-<repo> checkout <group-end>

# Clear new repo (except .git) and copy everything including hidden files
find /tmp/re-promote-<repo>-new -mindepth 1 -maxdepth 1 ! -name '.git' -exec rm -rf {} +
cp -r /tmp/re-promote-<repo>/* /tmp/re-promote-<repo>-new/
# Hidden files must be copied separately (cp * skips dotfiles)
for f in /tmp/re-promote-<repo>/.[!.]* /tmp/re-promote-<repo>/..?*; do
  [ -e "$f" ] && [ "$(basename "$f")" != ".git" ] && cp -r "$f" /tmp/re-promote-<repo>-new/
done

git -C /tmp/re-promote-<repo>-new add -A
git -C /tmp/re-promote-<repo>-new commit -m "<message>"
```

> **Windows 호환성**: `rsync`는 Windows Git Bash에 기본 설치되지 않는다. 위의 `find + cp` 방식을 사용한다.
> Hidden files (`.gitignore`, `.github/` 등)는 `cp -r *`로 복사되지 않으므로 별도로 복사해야 한다.

#### 4c: Migrate Other Branches

For each non-default branch:

1. **Analyze the branch**:
   ```bash
   git -C /tmp/re-promote-<repo> log --oneline <default-branch>..<branch> | head -50
   ```

2. **If the branch has unique commits** (not already merged to default):
   - Find the diverge point from default branch
   - Apply the same squash strategy
   - Create the branch in the new repo from the appropriate base point

3. **If the branch is fully merged** into default, skip it (the commits are already in the default branch history).

4. **Create and push the branch**:
   ```bash
   cd /tmp/re-promote-<repo>-new
   git checkout -b <branch>
   # apply squashed commits for this branch
   git checkout <default-branch>
   ```

5. **Show branch migration summary** to the user.

#### 4d: Commit Convention Resolution

Commit messages follow a convention file. Resolve the convention in this priority order:

1. **Project-level**: Look for `.claude/commit-convention.md` in the repo root
2. **User-level**: Look for `~/.claude/commit-convention.md`
3. **Default**: Use `references/commit-convention-default.md` bundled with this skill

Read the resolved convention file and apply its rules to all commit messages.

Regardless of which convention is used, these rules are **absolute and non-negotiable**:
- **NEVER include `Co-Authored-By:` lines with Claude, Anthropic, AI, or any AI tool name**
- **NEVER include any AI attribution footer of any kind**
- **NEVER include `Generated by`, `Assisted by`, or similar AI markers**
- Preserve original human author attribution where meaningful

---

### Phase 5: Transfer Issues

Open issues from the deprecated repo can be transferred to the new repo since they share the same owner.

1. **List open issues**:
   ```bash
   gh issue list --repo <owner>/<repo>-deprecated_<NNN> --state open --json number,title --jq '.[] | "\(.number)\t\(.title)"'
   ```

2. **If there are open issues, ask the user**:
   ```
   ## Open Issues (N개)

   | # | Title |
   |---|-------|
   | 1 | Fix login bug |
   | 2 | Add dark mode |
   | ... | ... |

   오픈 이슈를 새 레포로 이전할까요?
   - 전체 이전
   - 선택 이전 (번호 지정)
   - 건너뛰기 (deprecated 레포에 유지)
   ```

3. **Transfer approved issues**:
   ```bash
   gh issue transfer <issue-number> <owner>/<repo> --repo <owner>/<repo>-deprecated_<NNN>
   ```
   Transfer each issue one by one. Labels and assignees are preserved. Issue numbers will change in the new repo.

4. **Report transfer results**:
   ```
   ## Issue Transfer

   | Old # | New # | Title | Status |
   |-------|-------|-------|--------|
   | 5 | 1 | Fix login bug | transferred |
   | 12 | 2 | Add dark mode | transferred |

   클로즈된 이슈는 deprecated 레포에 아카이브로 보관됩니다.
   ```

---

### Phase 6: Verify and Push

1. **Verify all commit messages** across all branches:
   ```bash
   # Check default branch
   git log --format="--- %n%H%n%B" <default-branch> | head -500

   # Check other branches
   for branch in $(git branch --format='%(refname:short)'); do
     echo "=== $branch ==="
     git log --format="%h %s" <default-branch>..$branch 2>/dev/null
   done
   ```

2. **Automated verification checks**:
   - Conventional commit format present
   - No `Co-Authored-By` lines containing Claude/Anthropic/AI
   - No `Generated by` or `Assisted by` markers
   - Summary under 72 chars
   - Correct type prefix

3. **If any violations found**, fix them before proceeding.

4. **Show the final history**:
   ```
   ## Final Commit History

   ### <default-branch>
   | # | Hash | Message |
   |---|------|---------|
   | 1 | abc1234 | chore: initial project scaffolding |
   | 2 | def5678 | feat: add user authentication |

   ### Other Branches
   | Branch | Commits | Summary |
   |--------|---------|---------|
   | develop | 3 | ahead of main by 3 commits |
   | feature/xxx | 2 | new feature in progress |

   총 N개 커밋 (M개 브랜치). 푸시할까요?
   ```

5. **Push all branches**:
   ```bash
   git push -u origin --all
   ```

---

### Phase 7: Promote Readiness Check

Public 전환 전에 레포가 공개 준비가 되었는지 점검한다. 각 항목을 자동으로 검사하고 결과를 리포트한다.

#### 7a: 민감 정보 스캔

```bash
# .env, credentials, secrets, API keys 등 검사
git log --all --diff-filter=A --name-only --pretty=format: | grep -iE '\.(env|pem|key|p12|pfx|credentials)$' | sort -u
grep -rn --include='*.{js,ts,py,go,java,rb,rs,yml,yaml,json,toml}' -iE '(api_key|apikey|secret|password|token|private_key)\s*[:=]' . 2>/dev/null | grep -v node_modules | grep -v '.git/' | head -20
```

발견되면 **즉시 경고**하고 해당 파일/라인을 보여준다. 사용자가 확인할 때까지 진행하지 않는다.

#### 7b: 필수 파일 점검

| 파일 | 상태 | 조치 |
|------|------|------|
| `README.md` | 존재 여부 + 내용 충실도 | 없거나 빈약하면 경고 |
| `LICENSE` | 존재 여부 + 유효한 라이센스 | 없으면 사용자에게 라이센스 선택 요청 (MIT, Apache-2.0, etc.) |
| `.gitignore` | 존재 여부 + 적절한 패턴 | 없으면 언어에 맞는 기본 생성 제안 |

```bash
# 파일 존재 확인
for f in README.md LICENSE .gitignore; do
  test -f "$f" && echo "OK: $f" || echo "MISSING: $f"
done
```

#### 7c: README 품질 점검

README가 존재하면 다음 요소를 검사:

| 요소 | 검사 방법 |
|------|----------|
| 프로젝트 설명 | 첫 단락이 비어있지 않은지 |
| 설치 방법 | `install`, `setup`, `getting started` 등 섹션 존재 |
| 사용법 | `usage`, `example`, `how to` 등 섹션 존재 |
| 배지 | 상단에 shield.io 등 배지 존재 여부 (권장) |
| 데모/스크린샷 | 이미지나 GIF 포함 여부 (권장) |

빈약한 항목이 있으면 개선 제안. 사용자가 원하면 즉석에서 보완한다.

#### 7d: GitHub 레포 설정 점검

```bash
gh repo view <owner>/<repo> --json description,homepageUrl,repositoryTopics,hasIssuesEnabled,hasWikiEnabled
```

| 설정 | 점검 |
|------|------|
| Description | 비어있으면 작성 제안 |
| Topics | 비어있으면 적절한 토픽 제안 |
| Homepage URL | 있으면 유지, 없으면 물어봄 |
| Issues | 활성화 여부 |

#### 7e: 패키지 레지스트리 준비 (해당 시)

프로젝트 유형에 따라 자동 감지하고 점검:

| 파일 | 레지스트리 | 점검 |
|------|-----------|------|
| `package.json` | npm | name, version, description, main/exports, license 필드 |
| `setup.py` / `pyproject.toml` | PyPI | name, version, description, classifiers |
| `Cargo.toml` | crates.io | name, version, description, license |
| `go.mod` | Go modules | module path 확인 |

해당 없으면 건너뛴다.

#### 7f: Release / Tag 준비

1. **패키지 버전 감지** — 프로젝트의 manifest에서 현재 버전을 읽는다:
   ```bash
   # Python
   grep -m1 'version' pyproject.toml setup.cfg setup.py 2>/dev/null
   # Node
   node -e "console.log(require('./package.json').version)" 2>/dev/null
   # Rust
   grep -m1 'version' Cargo.toml 2>/dev/null
   ```

2. **기존 태그 확인**:
   ```bash
   git tag -l | head -20
   ```

3. **버전 일치 확인**: 릴리스 태그 생성 시 manifest 버전과 태그 버전이 일치하는지 검증한다.
   불일치 시 사용자에게 경고하고 어떤 버전으로 통일할지 물어본다:
   ```
   ⚠️ 버전 불일치 감지
   - pyproject.toml: 1.2.0b2
   - 제안 태그: v2.0.0

   어떤 버전으로 통일할까요?
   1. 태그를 manifest에 맞춤 (v1.2.0b2)
   2. manifest를 태그에 맞춤 (2.0.0) — pyproject.toml 수정 + 커밋
   3. 새 버전 지정
   ```

4. **사용자가 릴리스를 원하면**:
   ```bash
   git tag -a v<version> -m "<release message>"
   git push origin v<version>
   gh release create v<version> --title "v<version>" --notes "<release notes>" --repo <owner>/<repo>
   ```

#### 7g: Promote Readiness Report

모든 점검 결과를 종합하여 리포트:

```
## Promote Readiness Report

| 항목 | 상태 | 비고 |
|------|------|------|
| 민감 정보 | PASS | 감지 없음 |
| README.md | WARN | 배지 없음 |
| LICENSE | PASS | MIT |
| .gitignore | PASS | Python |
| Description | PASS | 설정됨 |
| Topics | WARN | 없음 — 추천: python, cli, whisper |
| 패키지 (PyPI) | PASS | pyproject.toml 확인 |
| Release | SKIP | 사용자 미선택 |

PASS: N개 | WARN: N개 | FAIL: N개
```

FAIL이 있으면 해결할 때까지 promote하지 않는다.
WARN은 사용자에게 보여주고 진행 여부를 물어본다.

---

### Phase 8: User Review and Promote

1. **Show the new repo status**:
   ```
   ## Re-Promote 준비 완료 (Private)

   - New repo: https://github.com/<owner>/<repo> (private)
   - Archived: https://github.com/<owner>/<repo>-deprecated_<NNN> (private)
   - Branches: N개
   - Clean commits: N개
   - Transferred issues: N개
   - Readiness: PASS N / WARN N / FAIL N

   레포를 확인해주세요. Public으로 전환할까요?
   ```

2. **Wait for user feedback**. They might want:
   - Commit message adjustments
   - Branch cleanup
   - README or other file changes before going public
   - Additional issues transferred
   - License addition
   - Topic/description changes

3. **Apply any requested changes**, then re-run affected readiness checks.

4. **When user approves, make public**:
   ```bash
   gh repo edit <owner>/<repo> --visibility public --accept-visibility-change-consequences
   ```

5. **Final report**:
   ```
   ## Re-Promote 완료!

   - Public repo: https://github.com/<owner>/<repo>
   - Archived: https://github.com/<owner>/<repo>-deprecated_<NNN> (private)
   - Branches: N개
   - Clean commits: N개
   - Transferred issues: N개

   deprecated 레포는 private으로 보관 중입니다.
   ```

---

### Cleanup

After everything is confirmed, clean up temp files:
```bash
rm -rf /tmp/re-promote-<repo>*
```

---

## Important Notes

- **No destructive operations without confirmation** — every phase has a user checkpoint
- **The old repo is never deleted** — only renamed and made private
- **Private first, public later** — the new repo starts private for review
- **No AI footers in commits** — absolute rule, no exceptions
- **All branches are migrated** — not just the default branch
- **Issue numbers will change** after transfer — this is a GitHub limitation
- **Closed issues stay in the deprecated repo** as archive
- **`gh` CLI is required** — if not installed, stop and tell the user
- If anything goes wrong mid-process, explain what happened and what state things are in

## Error Recovery

| Situation | Recovery |
|-----------|----------|
| Rename succeeded but new repo creation failed | The old repo is at `-deprecated_NNN`, rename it back: `gh repo rename <repo> --repo <owner>/<repo>-deprecated_<NNN>` |
| Push failed | Fix the issue locally, retry push. The old repo is safely archived. |
| Issue transfer failed for some issues | Report which ones failed, user can transfer manually later |
| Process interrupted mid-way | Report current state: which repo is named what, what's been pushed, what hasn't |

## Prerequisites

- `gh` CLI installed and authenticated (`gh auth status`)
- Git installed
- Write access to the target repository
- Permission to create repositories in the target owner/org
