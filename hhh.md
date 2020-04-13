
# [STEP-BY-STEP] 타겟에서 수행되는 gtest 성공 여부 판별의 원리

☞ gtest는 결과를 stdout에 출력하기 때문에 journal을 들여다보고 결과를 판별해야한다.

[STEP 1 - 타겟의 journal을 화면에 띄워줌과 동시에 임시파일에 저장하는 monitoring process를 백그라운드에 띄우기](#step-1---타겟의-journal을-화면에-띄워줌과-동시에-임시파일에-저장하는-monitoring-process를-백그라운드에-띄우기)

[STEP 2 - 타겟에 테스트 프로그램 띄워주기](#step-2---타겟에-테스트-프로그램-띄워주기)

[STEP 3 - 타겟에서 테스트 프로그램이 종료될 때까지 blocking하기](#step-3---타겟에서-테스트-프로그램이-종료될-때까지-blocking하기)

[STEP 4 - 백그라운드에 띄워둔 monitoring process 죽이기](#step-4---백그라운드에-띄워둔-monitoring-process-죽이기)

[STEP 5 - 저장된 output에서 패턴을 찾아 성공/실패를 판별](#step-5---저장된-output에서-패턴을-찾아-성공실패를-판별)

## STEP 1 - 타겟의 journal을 화면에 띄워줌과 동시에 임시파일에 저장하는 monitoring process를 백그라운드에 띄우기

```
bash -c "sdb shell 'journalctl -f' | stdbuf -i0 -o0 -e0 grep storeclienttest | stdbuf -i0 -o0 -e0 sed -E 's/[A-Za-z]* [0-9]* ([0-9]*:[0-9]*:[0-9]*) [A-Za-z]* storeclienttest\[[0-9]*\]: /\1 /' | tee ~test_result.tmp | stdbuf -i0 -o0 -e0 grep -E -i --color '\[  FAILED  \]|$'" &
```
이 커맨드는 어떻게 만들어진 것일까?

### STEP 1-1

디바이스의 저널 로그를 모니터링하도록 -f (follow) 옵션을 줘서 journalctl을 실행시키고, 'storeclienttest'가 포함된 행만 보여주도록 grep한다:
```
sdb shell 'journalctl -f' | stdbuf -i0 -o0 -e0 grep storeclienttest
```
※ 실시간성을 보장하기 위해 stdbuf 명령으로 standard input/output/error stream의 buffer 크기를 모두 0으로 만들어주었다.

다른 shell에서 test를 실행시킨다:
```
sdb shell 'aul_test launch org.tizen.storeclient.test'
```

결과:
```
Apr 09 14:00:43 localhost storeclienttest[10625]: [==========] Running 57 tests from 25 test cases.
Apr 09 14:00:43 localhost storeclienttest[10625]: [----------] Global test environment set-up.
Apr 09 14:00:43 localhost storeclienttest[10625]: [----------] 1 test from AppItem_install_uninstall
...
```

### STEP 1-2

Output의 각 행이 너무 길어서, 앞쪽 반복적이고 의미 없는 부분은 제거해주고 싶다:
```
sdb shell 'journalctl -f' | stdbuf -i0 -o0 -e0 grep storeclienttest | stdbuf -i0 -o0 -e0 sed -E 's/[A-Za-z]* [0-9]* ([0-9]*:[0-9]*:[0-9]*) [A-Za-z]* storeclienttest\[[0-9]*\]: /\1 /'
```
**Apr 09 14:00:43 localhost storeclienttest[10625]:** 이 부분을 패턴으로 찾아서, **14:00:43** 이 부분(괄호친 부분)만 취하는 sed 명령어를 추가해주었다.

이 step 역시 실시간성을 위해 stdbuf로 buffer를 0으로 만들어주었다.

다른 shell에서 test를 실행시킨다:
```
sdb shell 'aul_test launch org.tizen.storeclient.test'
```

결과:
```
14:18:36 [==========] Running 57 tests from 25 test cases.
14:18:36 [----------] Global test environment set-up.
14:18:36 [----------] 1 test from AppItem_install_uninstall
...
```
결과가 한결 보기 좋게 간단해졌다.

### STEP 1-3

테스트의 전체적인 성공 여부를 판별하려면, 이 output에서 테스트 종료 시 나오는 특정 패턴을 찾아야한다.

그러려면 이 output을 grep에 먹여줘야 하는데, 그러면 화면에선 전체적인 과정을 볼 수 없게 된다.

명령어 tee를 사용하면 output을 파일로 저장하면서 화면으로도 보여줄 수 있다.

이 경우, 테스트 실행을 script화하려면, 지금처럼 journal 로그 monitoring process를 한 shell에서 실행시켜놓고 manual하게 다른 shell 창을 열어서 테스트 프로그램을 실행시켜주는 것이 아니라, monitoring process를 백그라운드에 띄워놓은 후, 테스트 프로그램을 실행시키고, 테스트 프로그램이 종료되면 journal 결과를 체크해야 하기 때문에, journal 로그를 화면에 뿌려줌과 동시에 특정 tmp file에도 저장되도록 하면 된다:
```
sdb shell 'journalctl -f' | stdbuf -i0 -o0 -e0 grep storeclienttest | stdbuf -i0 -o0 -e0 sed -E 's/[A-Za-z]* [0-9]* ([0-9]*:[0-9]*:[0-9]*) [A-Za-z]* storeclienttest\[[0-9]*\]: /\1 /' | tee ~test_result.tmp
```

화면에 나오는 결과가 ~test_result.tmp에도 저장됨을 확인할 수 있다.

### STEP 1-4

테스트 실패가 좀 더 눈에 잘 띠도록 빨갛게 칠해준다:
```
sdb shell 'journalctl -f' | stdbuf -i0 -o0 -e0 grep storeclienttest | stdbuf -i0 -o0 -e0 sed -E 's/[A-Za-z]* [0-9]* ([0-9]*:[0-9]*:[0-9]*) [A-Za-z]* storeclienttest\[[0-9]*\]: /\1 /' | tee ~test_result.tmp | stdbuf -i0 -o0 -e0 grep -E -i --color '\[  FAILED  \]|$'
```

### STEP 1-5 (완성)

이 모든 게 백그라운드에서 실행되도록 해준다:
```
bash -c "sdb shell 'journalctl -f' | stdbuf -i0 -o0 -e0 grep storeclienttest | stdbuf -i0 -o0 -e0 sed -E 's/[A-Za-z]* [0-9]* ([0-9]*:[0-9]*:[0-9]*) [A-Za-z]* storeclienttest\[[0-9]*\]: /\1 /' | tee ~test_result.tmp | stdbuf -i0 -o0 -e0 grep -E -i --color '\[  FAILED  \]|$'" &
```
※ Sdb command는 & 연산자가 제대로 동작하지 않아서, 저 명령어를 실행시켜주는 bash를 따로 만들어서 그걸 통째로 백그라운드에 띄우는 것이다.

## STEP 2 - 타겟에 테스트 프로그램 띄워주기

이제 output을 받아내는 monitoring process가 백그라운드에서 동작하고 있는 상태가 됐으니, 테스트 프로그램을 실행시켜준다:
```
sdb root on
if [[ -z $FILTER ]]; then
    sdb shell 'aul_test launch org.tizen.storeclient.test'
else
    sdb shell 'aul_test launch org.tizen.storeclient.test TC ' $FILTER
fi
```
스크립트가 받은 TC 필터를 그대로 넘겨주도록 되어있다.

## STEP 3 - 타겟에서 테스트 프로그램이 종료될 때까지 blocking하기

```
sdb shell 'tail --pid=`ps aux | grep '"'"'org.tizen.storeclient.test'"'"' | grep -v grep | awk '"'"'{print $2}'"'"'` -f /dev/null'
```
이것의 원리는?

### STEP 3-1

디바이스에 sdb shell로 접속하여 다음과 같이 프로세스 테이블에서 org.tizen.storeclient.test가 포함된 행에서 두번째 열(pid)를 취할 수 있다:
```
ps aux | grep 'org.tizen.storeclient.test' | grep -v grep | awk '{print $2}'
```
※ grep했을 때 grep 프로세스 자체도 저 결과물에 포함되기 때문에 그것을 제외해주기 위해서 ```| grep -v grep```를 해주는 것이다.

### STEP 3-2

Tail명령어를 -f 옵션을 이용하면, 해당 프로세스가 종료될 때까지 blocking시킬 수 있다:
```
tail --pid=`ps aux | grep 'org.tizen.storeclient.test' | grep -v grep | awk '{print $2}'` -f /dev/null
```

### STEP 3-3 (완성)

위 명령을 script에서 실행해주기 위해 sdb shell 명령어로 만든다:
```
sdb shell 'tail --pid=`ps aux | grep '"'"'org.tizen.storeclient.test'"'"' | grep -v grep | awk '"'"'{print $2}'"'"'` -f /dev/null'
```
※ 외따옴표(```'```)를 외따옴표 안에 넣어주기 위해 ```'"'"'```로 교체해주었다. (중간에 외따옴표 잠깐 닫아주고, ```"'"``` 이걸 껴넣어(concat)주고, 다시 외따옴표 열어주는 개념)

## STEP 4 - 백그라운드에 띄워둔 monitoring process 죽이기

```
ps aux | grep "journalctl -f" | grep -v grep | awk '{print $2}' | xargs -i kill {}
```
sdb shell과, 그걸 감싸고 있는 bash 프로세스 등이 떠있을텐데, 이렇게 하면 몽땅 다 죽여줌

## STEP 5 - 저장된 output에서 패턴을 찾아 성공/실패를 판별

```
if [ `cat ~test_result.tmp | egrep -c '\[  FAILED  \]'` -ne 0 -o `cat ~test_result.tmp | egrep -c '\[  PASSED  \]'` -eq 0 ]; then
```
```[ FAILED ]```라는 패턴이 있(!= 0)거나, ```[ PASSED ]```라는 패턴이 없(==0)으면 실패, 그게 아니면 성공 - 이렇게 해주는 이유는, 중간에 비정상 종료되거나 하면 ```[ FAILED ]```가 없을 수도 있기 때문
