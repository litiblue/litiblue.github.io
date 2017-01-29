---
layout: post
title:  "더 나은 Git 사용을 위한 명령어"
date:   2017-01-21 17:45:00 +0900
categories: dev
---

이미 Git을 어느 정도 사용해보았고, branch의 merge개념도 어느정도 익숙한 분들을 위한 내용입니다. 핵심적인 내용만 간단히 정리해놓았으니, 보다 자세한 내용을 원하시는 분들은 뒤에 있는 참고 사이트를 보시길 바랍니다.

```

```

### 1. merging vs rebasing ###

![](/assets/img/merge_before-New-Page.png)

git rebase는 git merge와 같은 문제를 해결하기 위한 것으로, 브랜치의 변경사항을 다른 브랜치에 적용합니다. 위 그림 처럼 Master 브랜치에서 갈라져 나온 Feature 브랜치에서 작업을 하고 있는데, Master 브랜치가 변경 되었다면 변경 사항 반영을 위해 rebase 또는 merge를 택할 수 있습니다.

먼저 merge의 경우를 살펴보겠습니다. 명령어는 다음과 같습니다.

> git checkout feature  
> git merge master

![](/assets/img/merge_after.png)

이런 형태로 추가적인 커밋이 하나 추가된 것을 볼 수 있습니다. 머지의 장점은 비파괴적(non-destructive)이라는 것입니다, 기존 커밋들은 아무것도 바뀌지 않았습니다. rebase가 가지고 있는 잠재적 위험 요소를 제거합니다. 하지만 정돈되지않은 머지커밋들이 늘어나면 다른 개발자가 히스토리를 읽기 어려워 질수도 있습니다.

리베이스의 경우를 살펴 봅시다. 명령어는 다음과 같습니다.

> git checkout feature  
> git rebase master

Feature 브랜치 전체를 Master 브랜치의 끝으로 이동하고, Master 브랜치의 모든 새로운 커밋을 반영했습니다. 머지와 달리 새로운 커밋들을 생성하면서 히스토리를 다시 쓰게되었습니다.

![](/assets/img/rebase.png)

이미지는 위와 같습니다. 장점은 깨끗한 히스토리를 얻게된다는 것입니다. 완전히 선형적인 구조이기 때문에 git log, git bisect, git gitk등으로 탐색하기가 쉽습니다. 이렇게 새 것같은 커밋 히스토리에는 트레이드오프가 있습니다. 히스토리를 다시 씀으로인해 잠재적으로 협업 워크플로우에 큰 지장을 초래할 수도 있기에 주의해야 합니다.

**인터랙티브 리베이싱**이라는 것도 있습니다. 이를 이용하면 현재 작업중인 브랜치에서 여러개의 커밋을 한꺼번에 수정할 수 있습니다. 커밋의 순서를 바꾸거나 합치기 등이 가능합니다. 보통 feature 브랜치를 master 브랜치로 머지하기 전 지저분한 히스토리를 정리하기위해 사용합니다.

명령은 i 옵션을 주면 됩니다. 아래는 최근 3개의 커밋을 수정하기위한 명령입니다.

> git rebase -i HEAD~3

실행하면 편집기가 열리고 아래와 같이 각 각의 커밋 앞에 수행할 명령(pick, fixup등)을 적게 됩니다.

> pick 33d5b7a Message for commit #1  
> pick 9480b3d Message for commit #2  
> pick 5c67e61 Message for commit #3

이 리스트는 리베이스가 끝난 뒤 브랜치가 어떻게 보여질 것인가를 정의하는 것입니다. 순서를 바꿀수도 있고, 아래와 같이 fixup으로 바꾼 뒤 저장하면 두 번째 커밋이 바로 전 커밋인 첫 번째 커밋으로 합쳐지게 됩니다.

> pick 33d5b7a Message for commit #1  
> fixup 9480b3d Message for commit #2  
> pick 5c67e61 Message for commit #3

```


```

### 2. 커밋을 가리키는 상대적인 방법 ###

커밋을 가리키는 다양한 방법들이 있지만 여기서는 쉽고 자주 쓰이는 상대적인 방법에 대해 설명합니다. HEAD^ 과 HEAD~ 둘 다 HEAD를 기준으로 부모를 가리키는데, HEAD^숫자 와 HEAD~숫자인 경우 헷갈리지 말아야할 것은 ^는 머지 커밋이 있는 경우 여러 부모가 있을 때 몇 번째 부모인지를 나타내는 반면 ~는 조부모를 가리킵니다. 다음 예제를 보면 좀 더 명확히 알 수 있습니다.

![](/assets/img/ref.png)

```


```

### 3. 되돌리기 (reset, checkout, revert)
```

```

#### 3.1 reset

브랜치의 끝을 다른 커밋으로 이동하는 것입니다. 다음과 같이 커밋들을 지우는데 사용 가능합니다.

> git checkout hotfix  
> git reset HEAD~2

이렇게 하면 끝의 두 커밋은 댕글링 커밋(가비지 컬렉션 대상)이 됩니다.

![](/assets/img/reset.png)

옵션에 따라 적용되는 범위가 다릅니다.

* soft : 커밋 히스토리에만 영향을 줍니다.
* mixed : 기본값이며, 커밋 히스토리와 스테이지에 영향을 줍니다.
* hard : 커밋 히스토리, 스테이지, 워킹디렉토리 모두에 영향을 줍니다.

가장 많이 쓰는 경우는 다음과 같습니다.

> git reset –mixed HEAD^ : 최근 커밋 취소 (스테이지도 함께 취소)

```

```

#### 3.2 checkout

브랜치를 checkout했던 것처럼 커밋 주소를 통해서도 checkout하는것이 가능합니다. 브랜치의 경우와 마찬가지로 HEAD참조를 특정한 커밋으로 옮깁니다. 예를 들면 다음 명령은 현재 커밋의 조부모로 체크아웃 합니다.

> git checkout HEAD~2

![](/assets/img/checkout.png)

오래된 버전의 프로젝트를 살펴볼때 유용하지만, 현재 HEAD를 가리키는 브랜치 참조가 없어서 detached HEAD 상태가 됩니다. 이 상태에서 커밋을 하고 다른 브랜치로 스위칭을 하면 변경 사항들을 잃어버릴 수가 있습니다. 그래서 커밋을 할 때는 새로운 브랜치를 생성하는게 좋습니다.

```

```

#### 3.3 revert

새로운 커밋을 생성하여 되돌리기를 하는 방법입니다. 커밋 히스토리를 건드리지 않기 때문에 안전합니다.

> git revert HEAD~2

![](/assets/img/revert.png)

위 그림을 보면 B라는 커밋이 새로 생성되었는데, 두 번째 전으로 revert되었기 때문에 A커밋의 상태와 동일합니다. 원격 저장소에 푸쉬한 경우 이 방법으로 되돌리기가 가능합니다.

```

```

### 4. stash

스테이지에 있는 파일들을 스택에 잠시 저장해두는 명령어 입니다. 어떤 브랜치에서 작업중인데 다른 브랜치로 잠시 이동(checkout)해야 할 경우 유용합니다. 다른 브랜치에서의 작업이 끝나면 다시 돌아온 뒤 스택에서 꺼내면 됩니다.

* git stash : 스테이지에 있는 파일들과 tracked 이면서 modified인 파일들을 스택에 저장합니다.
* git list : 스택에 저장된 것들을 목록을 보여줍니다.
* git pop : 스택에서 꺼내서 현재 브랜치에 적용합니다. 이 때 stash 했던 브랜치여야 할 필요는 없습니다.

```

```

### 참고(reference)

- <http://www.atlassian.com/git/tutorials/advanced-overview/>
- <http://git-scm.com/book/ko/v2/>
