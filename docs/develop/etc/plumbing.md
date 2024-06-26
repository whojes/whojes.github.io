---
layout: default
title: plumbing commands
parent: etc
grand_parent: 개발
created_at: 2019.08.22
print_title: true
share_enable: true
permalink: develop/etc/plumbing
---
### [porcelain and plumbing commands](https://git-scm.com/book/ko/v2/Git%EC%9D%98-%EB%82%B4%EB%B6%80-Plumbing-%EB%AA%85%EB%A0%B9%EA%B3%BC-Porcelain-%EB%AA%85%EB%A0%B9){:target="_blank"}  
<p style='margin-top: 15px;'>
  &nbsp;&nbsp;Git 에는 두가지 타입의 명령어가 존재하는데, porcelain commands 와 plumbing commands 가 그것이다. porcelain 커맨드는 우리가 흔히 알고 있는 user level 의 커맨드들을 통칭하는 것이고, plumbing 커맨드는 low level 의 명령어를 말한다.
  low level 의 커맨드를 공부하는 것은 git 의 구조를 이해하는데 도움이 많이 된다.
</p>

---

track 되고 있지 않은 test.sh 이라는 이름의 파일을 git에 추가하고 커밋하는 명령어를 보면

```bash
// porcelain 명령어
:~$ git add test.sh
:~$ git commit -m "test.sh commit"
```
로 user level 에서는 표현할 수 있지만, plumbing commands 들로 표현하면 다음과 같다.

```bash
// plumbing 명령어
:~$ git hash-object -w test.sh
4b31a2ba75e0a892e230088949eb08ce6d091ed2
:~$ git update-index --add test.sh
:~$ git write-tree
0900f4103708966f97d2e0d5c079cba4ffa71a07
:~$ git commit-tree -p ${마지막 commit id} -m "test.sh commit" 0900f4103708966f97d2e0d5c079cba4ffa71a07
b14f6c92f4ec6345c52a06a3fec51d38d091ba4a
:~$ git update-ref refs/heads/master b14f6c92f4ec6345c52a06a3fec51d38d091ba4a
```

&nbsp;&nbsp;git은 새로이 파일시스템을 만들어 그 스냅샷을 commit 이라는 이름으로 기록한다. 이 git 의 파일시스템에서 디렉토리는 tree, 나머지 모든 파일은 blob (binary large object) 형태로 저장하고, Git 에서 이 모든 것, commit/tree/blob 에 tag 를 더해 4가지 오브젝트를 저장하고 있다. 저장의 형태는 해당 오브젝트의 sha1 해시값 (40자리 16진수 중 2자리 directory 에 38자리 파일 이름으로 관리하고 있다. `${git-project}/.git/objects` 경로에서 확인할 수 있다.), 내용은 zlib 을 이용한 압축으로 관리된다.


처음에 추가된 파일을 git에서 관리할 수 있는 sha1 hash object 로 만들어서 저장(-w 옵션)한다.
```bash
:~$ git hash-object -w test.sh
4b31a2ba75e0a892e230088949eb08ce6d091ed2
```

그리고 git staging 을 위해 index 에 추가한다. (현재 상태 index 는 .git/index 파일에 명시되어있다.)
```bash
:~$ git update-index --add test.sh
```

이 후 git 에서 관리하는 파일시스템의 루트 트리를 저장한다.
```bash
:~$ git write-tree
```
이 때, 각 commit 오브젝트는 자신의 루트 트리의 해시를 갖고 있으므로 (commit-tree 명령어 사용 시에 루트 트리의 hash를 넣어준다.) 이 때 저장된 루트 트리는 현재 마지막 커밋의 루트 트리와는 다르다.


이 후에 commit-tree 명령어를 통해서 parent 커밋과 root tree 의 해시를 지정하여 새로운 커밋을 찍는다.
```bash
// -p 옵션은 옵셔널
:~$ git commit-tree -p ${마지막 commit id} -m "test.sh commit" 0900f4103708966f97d2e0d5c079cba4ffa71a07
b14f6c92f4ec6345c52a06a3fec51d38d091ba4a
```

그리고 마지막으로 git 의 포인터가 바라보고 있는 커밋을 바꿔준다.
```bash
:~$ git update-ref refs/heads/master b14f6c92f4ec6345c52a06a3fec51d38d091ba4a
```

--- 
* git cat-file 명령어는 git 에서 관리하는 object 들의 정보/상태/종류/내용 등을 볼 수 있는 명령어이다.

<p align="center">
  <br><img alt="img-name" src="/assets/images/git/cat-file.png" class="content-image-1"><br>
</p>

* git 에서 관리하는 objects 에 대한 정보

| type | size | content |
|:----:|:----:|:-------:|
| tree | 트리 오브젝트의 용량을 bytes로 표시 | 하위에 포함하고 있는 tree/blob 의 list 객체에 대한 접근권한, 파일이름은 여기서 관리한다. |
| blob | 컨텐츠의 용량을 bytes로 표시 | 텍스트, 이미지, 음악 혹은 단순 이진 파일처럼 다양한 형식의 파일이 저장될 수 있다. 파일이름이나 파일형식은 blob에 저장되지 않는다. (포함하는 tree에서 저장) |
| commit | commit content의 용량을 bytes 로 표시 | parent commit, author(date), committer(date), tree hash, commit message 를 포함 |  

<br>
*출처: [git-scm](https://git-scm.com/book/ko/v1/Git%EC%9D%98-%EB%82%B4%EB%B6%80-Git-%EA%B0%9C%EC%B2%B4){:target="_blank"}*  
*참조: [storycompiler](https://storycompiler.tistory.com/7){:target="_blank"}*
