---
layout: default
title: bash commands
parent: etc
grand_parent: 개발
created_at: 2019.05.16
print_title: true
share_enable: true
permalink: develop/etc/bash
---

### contents
1. [basic](#1-basic)  
2. [default value](#2-default-value)  
3. [if 조건문](#3-if-조건문)  
4. [xargs](#4-xargs)  
5. [awk](#5-awk)  
6. [command](#6-command)  
7. [trap](#7-trap)  
8. [pipefail](#8-pipefail)  
9. [exec](#9-exec)  
10. [shift](#10-shift)  

### 1. basic 

- pipeline 
    ```bash
    |, ||, &, &&
    ```

    | 는 파이프라인, 좌측 명령어의 stdout 을 우측 명령어의 argument로 전달
    &는 명령어를 backgroud로 실행, exit code 를 0으로 보냄 (터미널은 붙어있으니 필요시 적절하게 /dev/null 이나 기타 로그파일로 리디렉트해줘야함)
    || 는 or 연산자, 좌측 명령어의 exit code가 0이 아닐경우 우측 명령어 실행  
    && 는 and 연산자, 좌측 명령어의 exit code 가 0일 경우 우측 명령어 실행  

- type

    shell script 를 짜는데 유용한 built-in command 가 몇개 있다. 늘상 쓰는 pwd, export, source, exit 외에 몇 가지 binary인줄 알았는데 bash builtin 인것도 있음을 발견했다. cd, echo, history (!), alias 도 builtin 명령어들이었다. 
    builtin 인지 아닌지 확인하는 builtin command 가 있다. 
    
    ```bash
    :~$ type -f ${cmd}
    ```

### 2. default value  
    
bash 스크립트 중 parameter 인풋이 없으면 default 로 설정하는 명령어

```bash
if [ -z "$1" ] then
    version = "default"
else
    version = $1
fi 
```
대충 이럴텐데 (테스트는 안해봄),

```bash
version = ${@:-default}
```

그냥 이렇게 쓰면 된단다. 나참.... 문법이 으이구 이걸 어떻게 알아보나이거 배쉬놈들

### 3. if 조건문  

bash 문법은 if 조건문을 사용할 때 

```bash
if [ A == B ] 
if [ C != D ]
```

이게 아니라

```bash
if [ A -eq B ]
if [ C -ne D ]
```

이렇게 써야한다. 이런 예시는 좀 빡치지만 (bash 를 잘 안쓰면 쓸 때마다 검색해야하게만든다), 훨씬 더 많은 비교 및 조사를 할 수 있게 되는 것 같다.

```bash
[ -d ] : 파일이 디렉토리면 
[ -e ] : 파일이 있으면 참
[ -L ] : 파일이 심볼릭 링크면 참
[ -r ] : 파일이 읽기 가능하면 참
[ -s ] : 파일의 크기가 0 보다 크면 참
[ -w ] : 파일이 쓰기 가능하면 참
[ -x ] : 파일이 실행 가능하면 참
[ 파일1 -nt 파일2 ]  : 파일1이 파일2보다 최신파일이면 참
[ 파일1 -ot 파일2 ]  : 파일1이 파일2보다 이전파일이면 참
[ 파일1 -ef 파일2 ] : 파일1이 파일2랑 같은 파일이면 참 

// 출처: https://jink1982.tistory.com/48 [돼민이] 
```

등의 수많은 variation 이 가능하다. 이러면 단순히 같다를 ==로 하게 되면 통일성이 떨어지겠지

### 4. xargs 

xargs 를 얼마나 잘 활용하는가 하는게 이사람이 쫌 간지나는 사람이구나를 판별하는 척도로 사용할 수 있겠다.

딱히 bash 랑만 관계있는건 아니지만, bash를 거의 사용하니까...

사실 대부분 $(명령어) 를 이용해서도 가능하긴 하지만, -0 옵션같은 유용한 사용법이 있어 더 안전하다.

```bash
ls -al $(find . -name "*.py") 
```

는 모든 py 스크립트를 찾기에 간편하지만, 파일 이름에 공백이나 다른 특수문자가 있을 경우 명령어 전체에서 에러가 난다. stdout 도 꼬이고 echo $?로 ec를 확인해봐도 1이 나온다. 이를 방지하기 위해

```bash
find . -name "*.py" -print0 | xargs -0 ls -al
```

이런식으로 사용하면 된다. find 의 -print0 옵션은 결과 사이를 개행문자가 아닌 널문자로 구분하도록 stdout 을 내보내는 역할을 하고,  xarg의 -0옵션은 공백이 아닌 널문자로 인풋파라미터를 구분하도록 하는 옵션이다. man xargs 를 보면 알겠지만 -0 옵션 자체가 find 의 print0에 대응되도록 만들어졌다.​

### 5. awk 

아... awk.... 뭔가 웹크롤링할때처럼 text 파일들이나 df, fdisk, lshw 등 명령어의 output을 크롤링할때 많이 써서 보기만해도 기분나쁘다. 그냥... 잘 쓰면 좋은데 그냥 기분나쁘다. 
잘 정리된 [블로그](https://zzsza.github.io/development/2017/12/20/linux-6/)가 있어 참조한다.

### 6. command 

해당 명령어는 -v 옵션과함께 사용시 arg 로 들어온 것이 bash 에서 어떤 행동을 하는지 알려준다. (alias가 되어있으면 alias도 알려줌) 이를 이용해 스크립트 내에서 사용하는 바이너리들의 존재여부를 스크립트 시작전에 파악하도록 구성할 수 있다.

```bash
//내부 스크립트 진행시 sha256sum, go, javac, java 바이너리는 반드시 필요할 경우
//해당 명령어들을 사전에 command -v ${cmd} 을 통해 진행, exitcode가 0이 아닐경우
//stderr로 에러메세지 뿜고 스크립트 종료
for cmd in sha256sum go javac java; do
if ! command -v $cmd &> /dev/null; then  
    echo >&2 "error: $cmd not found" 
    exit 1
fi
done
```

### 7. trap 

trap 명령어는 go 언어의 defer같은 느낌으로다가 사용할수 있는 builtin command이다. 특정 시그널을 받을 때 실행할 명령어를 지정할 수 있다.

```bash
// 스크립트가 종료될 때 다음 명령어를 수행하도록
trap "echo \"ALL IS WELL\"" EXIT

// ERR SIG를 받을 때 다음 명령어를 수행한다.
trap "echo \"SOMETHING\'S WRONG\"" ERR
```

### 8. pipefail 

스크립트 내부에서 명령어가 실패할경우 bash는 스크립트를 끝내지않고 다음 명령어를 수행한다. 

```bash
set -eouE pipefail
```

`-e` 옵션은 exit code가 0 이 아닌 커맨드를 만났을 때 script를 진행하지 않고 끝내라는 셋팅이다. (`$cmd || true` 등의 명령어를 통해 실패해도 script 종료가 나지 않게 지정할 수 있다.) 다만 이 경우 pipeline 을 통한 명령어가 실행될 때는 pipeline 의 좌측 명령어가 exit code 를 0이 아니게 내뿜어도 우측 명령어가 0이면 에러가 아니게 된다. 즉,
    
```bash
ls /not/existing/dir | grep something
```

같은 명령어의 경우 좌측이 exit code 2로 나와도 파이프라인을 받는 명령어가 exit code를 0을 내밀기에 해당 스크립트는 여전히 진행된다. 이를 막기위해서 -o pipefail 옵션도 주면 된다. `-E` 옵션을 주지 않으면 위의 trap이 지정한 ERR SIG를 캐치하지 못한다.

`-u` 옵션은 setting 되지 않은 환경변수를 호출했을 때 empty를 return하지 않고 script를 종료시킨다.

setting 되지 않은 환경변수 하니까 나온건데 지난번 `${@:-default}` 는 `${a:-b}` 문법의 확장판이다. a라는 변수가 있으면 a로 세팅하고 없으면 b로 하라는 말이다. -u 옵션은 이 문법에서 a가 세팅되어있지않더라도 에러를뿜거나 하지는 않는다.

추가로, 디버깅을 위해 명령어가 실행되기 전 명령어를 인쇄하도록 하고싶으면 `-x` 옵션을 사용하면 된다.
​

### 9. exec 

명령어를 실행하되 실행한 bash 의 하위 프로세스로 돌리는게 아니고 실행한 프로세스의 pid를 물려받아 실행하도록 한다. 허접은 잘 쓸수가 없다.

### 10. shift 

$1, $2 ... 등으로 사용하는 input args 를 왼쪽으로 한칸씩 이동할 때 쓴다. 왜있는건지 도통 모르겠다. 고수들은 잘쓰려나? 약간 java iterable 같은거에서 cursor 옮기는 느낌이려나