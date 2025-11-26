# GitHub 저장소 생성 및 연결 가이드

## 1. GitHub에서 저장소 생성

1. https://github.com 에 로그인 (hslim.engineer@gmail.com)
2. 우측 상단의 "+" 버튼 클릭 → "New repository" 선택
3. 저장소 설정:
   - Repository name: `qsa-auth-solution`
   - Description: `QSA (Quantum Safe API) - JWT 기반 이중 인증 시스템`
   - Visibility: Public 또는 Private (선택)
   - **중요**: "Initialize this repository with a README" 체크 해제 (이미 로컬에 파일이 있음)
   - "Add .gitignore" 선택 안 함 (이미 있음)
   - "Choose a license" 선택 안 함
4. "Create repository" 클릭

## 2. 로컬 저장소와 연결

저장소 생성 후 GitHub에서 표시되는 명령어를 사용하거나, 아래 명령어를 실행:

```bash
cd /Users/hyungsoolim/dev/qsa
git remote add origin https://github.com/hslim.engineer/qsa-auth-solution.git
git branch -M main
git push -u origin main
```

또는 SSH를 사용하는 경우:

```bash
git remote add origin git@github.com:hslim.engineer/qsa-auth-solution.git
git branch -M main
git push -u origin main
```

## 3. 인증

첫 푸시 시 GitHub 인증이 필요할 수 있습니다:
- Personal Access Token 사용 (권장)
- 또는 GitHub Desktop 사용





