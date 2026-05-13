---
layout: single
title: "Git 설치 및 Github 설정"

categories:
  - tech
tags:
  - tech

toc: false
toc_sticky: true

date: 2026-05-13
last_modified_at: 2026-05-13 17:33:00
---

# Git 설치 및 Github 설정

## 1. Git 설치
```bash
winget install --id Git.Git -e --source winget

git --version
```

```bash
git config --global user.name "kim.wt"
git config --global user.email "kolder2@gmail.com"
```

## 2. Github Repository Checkout
```bash
git clone https://github.com/kolder2/ktr26-dev.git
```

`git checkout`을 통해 저장소를 가져온 후, Jekyll(정적 사이트 생성기)을 설치하고 GitHub에 다시 올리는(push) 전체 과정은 다음과 같습니다.

### 2-1. Jekyll 프로젝트 생성

기존 저장소 폴더 안에서 Jekyll 환경을 구성합니다. 윈도우 환경에서는 미리 [Ruby+Devkit](https://rubyinstaller.org/)이 설치되어 있어야 합니다. [1, 2, 3]

- 터미널(PowerShell 또는 CMD)에 `ruby -v`를 입력해 보세요.
- 만약 `ruby`도 인식되지 않는다면, 아직 설치되지 않은 것이니 [RubyInstaller](https://rubyinstaller.org/downloads/)에서 **Ruby+Devkit** 버전을 다운로드하여 설치해야 합니다.
- **중요:** 설치 과정 중 **"Add Ruby executables to your PATH"** 체크박스에 반드시 체크해야 합니다.
    ```bash
    ruby -v
    gem -v
    
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
    ridk install
    ridk enable
    ```
    
1. Jekyll 설치 (이미 설치되어 있다면 생략): 터미널에 아래 명령어를 입력합니다.
    
    ```bash
    gem install jekyll bundler
    ruby -e "puts Gem.user_dir"    
    ```
    - **환경 변수 추가**:
	    - `Win + S` 누르고 **'시스템 환경 변수 편집'** 검색 후 실행.
	    - **[환경 변수]** > '사용자 변수'의 **Path** 선택 > **[편집]** 클릭.
	    - **[새로 만들기]** 클릭 후, 1번에서 확인한 경로에 `\bin`을 붙여 추가합니다. (예: `C:\Users\사용자명\.local\share\gem\ruby\3.x.x\bin`)
	    - 모든 창에서 **확인**을 눌러 저장합니다.

2. 터미널 재시작 (필수)
3. 새 사이트 생성: 현재 폴더(`.`)에 Jekyll 파일을 생성합니다. 폴더가 비어 있지 않다면 `--force` 옵션이 필요할 수 있습니다.
    
    ```bash
	jekyll new . --force
	```
    
4. 의존성 설치: `Gemfile`에 정의된 패키지들을 설치합니다.
    
    ```bash
    bundle install
    ```

### 2-2. 로컬 실행 확인 (선택 사항) [7]

GitHub에 올리기 전, 사이트가 정상적으로 만들어졌는지 확인합니다.

```bash
bundle exec jekyll serve
```

터미널에 표시된 주소(보통 `http://127.0.0.1:4000`)에 접속하여 사이트를 확인하세요.

### 2-3. Git에 추가 및 Push

생성된 Jekyll 파일들을 스테이징하고 GitHub 원격 저장소로 보냅니다.

1. 변경사항 추가:
    
    ```bash
    git add .
    ```
    
2. 커밋 메시지 작성:
    
    ```bash
    git commit -m "Add Jekyll scaffold"
    ```
    
3. GitHub로 Push:
    
    ```bash
    git push origin main
    ```
    
    _(기본 브랜치 이름이 `master`라면 `main` 대신 `master`를 입력하세요)_.

---

### 💡 팁: GitHub Pages 자동 배포

GitHub 저장소의 Settings > Pages 메뉴에서 'Build and deployment' 소스를 GitHub Actions로 설정하면, 이후 코드를 `push`할 때마다 자동으로 Jekyll 사이트가 빌드되어 웹에 게시.

이제 사이트의 _config.yml 파일을 수정하여 블로그 이름이나 설명 변경, 테마 적용 가능

<details>
  <summary>참고</summary>  
  <pre>

  </pre>
</details>
