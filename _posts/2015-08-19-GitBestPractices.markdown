---
layout: post
title:  "Git 베스트 프랙티스"
date:   2015-08-19 17:45:00 +0900
categories: dev
---
본격적으로 Git을 사용하기에 앞서 Git사용시 베스트 프랙티스가 무엇인지 알아보았습니다.

<br/>

### 1. commit 하기 전에 git diff로 코드 리뷰 (버그 방지)
  - git add --patch (diff를 하나씩 보여주면서 스테이지 여부를 물어봄)
  - git diff --cached (최종 커밋하기 전에 diff로 확인)

<br/>

### 2. 작고, 논리적인 커밋 (일찍 자주하기)
  - 잘 관리된 저장소의 커밋 로그는 하나의 스토리다
  - git status와 git diff로 변경사항 리뷰시 단일한 논리적 단위, 단순한 문장으로 설명 되어야함.
  - 디버깅시에 커밋 단위가 잘게 나눠져있는 편이 어떤 라인에서 버그가 발생했는지 알기 용이함
  - 가이드라인
    - 절대 하나의 커밋에 여러개의 모듈들을 묶어 넣지 않는다. (여러 모듈이 타이트하게 커플링관계가 아닌 이상)
    - 새 모듈을 작성할때 그루터기 함수들을 먼저 작성하고, 돌아와서 그 것들을 채워넣는다.
    - 항상 API레벨의 함수들을 작성하고 커밋한다. (그 것들을 사용하는 함수들을 작성하고 커밋하기전에)

<br/>

### 3. 깨끗한 커밋 히스토리를 남겨라
git merge를 많이 쓰게되면 커밋 히스토리를 읽기 어려워짐
  - git rebase와 git merge의 조합을 사용할 것
  - push 전에 rebase할 것

<br/>

### 4. 절대 공유된 커밋은 리베이스 하지 말라
  - 규칙 : 한 번 원격 저장소에 푸쉬한 커밋은 리베이스하지 않기
  - 이렇게 하면 다른 사람들의 코드를 망가뜨리지 않게 해주고, 내가 수정한 버그나 실수의 기록도 보존시켜준다.

<br/>

### 5. 용도에 알맞게 pull request를 사용하라.
  - 팀원들이 무엇이 어떻게 돌아가는지 알 수 있게 해줌 (변경사항 추적, 커밋 연계, 개발 중 협업, 머지 전 코드 리뷰)
  - 머지 커밋이 눈에 띄는게 유일한 단점이긴 하지만, 팀으로 일할 때 이 점은 상쇄된다.

<br/>

### 6. 명령어를 위한 alise를 사용할 것 (~/.gitconfig에 추가)
```
[alias]
lg = log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s \
%Cgreen(%cr, %cn)%Creset' --abbrev-commit --date=relative

hist = log --graph --pretty=format:'%h %ad | %s%d [%an]' --date=short

last = log -1 HEAD
unstage = reset HEAD --
amend = commit --amend -C HEAD
co = checkout
ci = commit
st = status
br = branch
```

<br/>

### 참고자료
  - <https://sethrobertson.github.io/GitBestPractices/>
  - <http://chriskottom.com/blog/2014/02/a-few-modest-best-practices-for-git/>
  - <https://www.lullabot.com/articles/git-best-practices-workflow-guidelines>
