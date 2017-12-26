---
layout: post
title:  "멈춰있는 Python 프로세스 원인 찾기"
date:   2017-12-21 10:50:00 +0900
categories: dev
---

데이터 추출,변환,적재(ETL)를 위하여 Python프로세스들을 Apache NiFi에 등록하여 사용하고 있습니다. Apache NiFi는 각 프로세스의 실행 및 프로세스간 데이터 흐름을 관리합니다.

데이터 수집이 잘 되고 있는지 확인하기 위하여 NiFi의 Summary메뉴로 들어가보니 AirekoreaFetcher가 멈춰있는 것이 발견되었습니다. AirkoreaFetcher는 환경 공단에서 제공하는 API를 사용하여 전국의 대기 환경 데이터를 가져오는 프로세스로 한 시간에 한 번 실행 됩니다. 프로세스의 데이터 처리량을 확인할 수 있는 Status History의 시계열 차트를 보니 하루 종일 아무 것도 없어 뭔가 문제가 있다는 것을 확인할 수 있었습니다.

![](/assets/img/NiFi_Summary.png)
*프로세스 멈춤 문제 발견 : Summary -> Status History -> 데이터 처리량 확인*

어떤 이유로 이런 문제가 발생했는지 그 원인을 찾는 과정을 설명하겠습니다. NiFi를 사용하지 않는 경우에도 Python 프로세스가 멈추는 문제 발생시 동일한 방법을 적용할 수 있습니다.

<br/>

### 프로세스 실행 정보 확인

![](/assets/img/airkorea_fetcher_command.png)
*AirkoreaFetcher의 Python 실행 명령문*

문제 발생 원인을 찾기 위하여 해당 프로세스의 실행 정보를 알아냅니다.

{% highlight shell %}
[root@es04 nifi-1.2.0]# ps -ef | grep fetcher

root     25405 10961  0 12월18 ?      00:00:00 python /home/deploy/rems/rems/airkorea_fetcher.py /home/deploy/rems/conf/collectors.cfg
root     26607 25405  0 12월18 ?      00:00:00 python /home/deploy/rems/rems/airkorea_fetcher.py /home/deploy/rems/conf/collectors.cfg
{% endhighlight %}

관련 프로세스가 2개 실행 중인데 실행 시간이 3일 전인 12월 18일로 되어있어 문제가 생겨 멈춰있는 것 처럼 보입니다.

<br/>

#### 좀 더 디테일한 시간 찾기

{% highlight shell %}
[root@es04 nifi-1.2.0]# ls -ld /proc/25405
dr-xr-xr-x. 9 root root 0 12월 18 03:25 /proc/25405

[root@es04 nifi-1.2.0]# ls -ld /proc/26607
dr-xr-xr-x. 9 root root 0 12월 18 03:29 /proc/26607
{% endhighlight %}

PID를 이용하여 날짜 뿐만이 아닌 시간,분 까지 찾아내 로그를 분석할 때 사용합니다. 실행 시점에 실제로 어떤 일이 발생했는지 확인할 수 있습니다.

<br/>

### 무엇 때문에 멈췄는지 추적

#### strace

`strace`를 사용하여 실행 중인 프로세스 안을 들여다 볼 수 있습니다. 프로세스가 호출하는 `system call`과 받은 `signal`을 확인 가능하며 각 `system call`의 이름, 인자, 리턴 값을 출력 합니다. (`system call`이란 프로세스와 Linux 커널 사이의 인터페이스로 커널을 통해 어떤 일을 처리할 때 사용합니다.)

{% highlight shell %}
[root@es04 nifi-1.2.0]# strace -p 26607
Process 26607 attached
recvfrom(7, ^CProcess 26607 detached
 <detached ...>

[root@es04 nifi-1.2.0]# strace -p 25405
Process 25405 attached
wait4(26607, ^CProcess 25405 detached
 <detached ...>
{% endhighlight %}

프로세스 실행 중엔 복잡한 내용들이 정신 없이 출력 되는 것을 볼 수 있지만 지금은 프로세스가 멈춘 상태로 마지막으로 호출된 `system call` 정보만 나타납니다.

#### GDB

`strace`를 통해선 `system call` 이름과 첫 번째 인자만 확인할 수 있었는데 보다 자세한 정보를 보기 위해 `GDB`를 사용합니다. `GDB`는 Linux에서 사용하는 디버거로 `strace`와 마찬 가지로 프로세스 안을 들여다 볼 수 있습니다.

{% highlight shell %}
[root@es04 nifi-1.2.0]# gdb

GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-94.el7
Copyright (C) 2013 Free Software Foundation, Inc.

...

(gdb) attach 26607

Attaching to process 26607
Reading symbols from /usr/bin/python2.7...Reading symbols from /usr/lib/debug/usr/bin/python2.7.debug...done.
done.
Reading symbols from /lib64/libpython2.7.so.1.0...Reading symbols from /usr/lib/debug/usr/lib64/libpython2.7.so.1.0.debug...done.
done.
Loaded symbols for /lib64/libpython2.7.so.1.0
Reading symbols from /lib64/libpthread.so.0...Reading symbols from /usr/lib/debug/usr/lib64/libpthread-2.17.so.debug...done.
done.
[Thread debugging using libthread_db enabled]
Using host libthread_db library "/lib64/libthread_db.so.1".
Loaded symbols for /lib64/libpthread.so.0
Reading symbols from /lib64/libdl.so.2...Reading symbols from /usr/lib/debug/usr/lib64/libdl-2.17.so.debug...done.
done.

...
...

Loaded symbols for /lib64/libnss_files.so.2
Reading symbols from /lib64/libnss_dns.so.2...Reading symbols from /usr/lib/debug/usr/lib64/libnss_dns-2.17.so.debug...done.
done.
Loaded symbols for /lib64/libnss_dns.so.2
0x00007fae5db8182b in __libc_recv (fd=7, buf=buf@entry=0x24ca224, n=n@entry=8192, flags=flags@entry=0) at ../sysdeps/unix/sysv/linux/x86_64/recv.c:33
33	  ssize_t result = INLINE_SYSCALL (recvfrom, 6, fd, buf, n, flags, NULL, NULL);

(gdb) quit
{% endhighlight %}

`strace`와 `GDB`를 통해 PID `26607` 프로세스가 `recvfrom`함수에서 멈춰 있는 것을 확인했습니다.

#### recvform

인터넷에 `recvfrom`을 검색하여 소켓 통신에서 데이터를 받아올 때 사용하는 함수라는 것을 알게되었습니다.  
관련 링크 : <https://msdn.microsoft.com/en-us/library/windows/desktop/ms740120(v=vs.85).aspx>

<br/>

### 소스 코드 확인

#### multiprocessing

도대체 어느 부분이 문제인지 소스 코드 및 로그를 확인 합니다.

{% highlight python%}
lock = multiprocessing.RLock()
fetchers = [
    AirkoreaFetcher(station_id, station_name, dt_str, lock, config)
    for station_id, station_name in station_info]

for w in fetchers:
    w.start()
    time.sleep(2)

for w in fetchers:
    w.join()
{% endhighlight %}

`0` 부터 `209` 까지의 station_id 별로 프로세스를 생성(multiprocessing 사용)하여 동시에 실행 하는 코드 인데 로그를 살펴보니 station_id가 `144`일 때 시작(start_time, end_time 출력)은 있는데 끝(데이터 길이 출력)이 없는 것이 보입니다. 전체 로그를 보면 `144`를 제외한 모든 station_id의 시작과 끝이 쌍을 이루고 있습니다.

{% highlight shell %}
...
...
2017-12-18 03:29:43 [__main__] INFO: station_id: 0142, start_time: 2017-12-18 03:00:00, end_time: 2017-12-18 03:29:43.232133
2017-12-18 03:29:45 [__main__] INFO: station_id: 0143, start_time: 2017-12-18 03:00:00, end_time: 2017-12-18 03:29:45.233463
2017-12-18 03:29:47 [__main__] INFO: station_id: 0144, start_time: 2017-12-18 03:00:00, end_time: 2017-12-18 03:29:47.235439
2017-12-18 03:29:49 [__main__] INFO: station_id: 0145, start_time: 2017-12-18 03:00:00, end_time: 2017-12-18 03:29:49.237130
2017-12-18 03:29:51 [__main__] INFO: station_id: 0143, len(data_list): 1
2017-12-18 03:29:53 [__main__] INFO: station_id: 0147, start_time: 2017-12-18 03:00:00, end_time: 2017-12-18 03:29:53.242072
2017-12-18 03:29:53 [__main__] INFO: station_id: 0142, len(data_list): 0
2017-12-18 03:29:54 [__main__] INFO: station_id: 0145, len(data_list): 1
2017-12-18 03:29:58 [__main__] INFO: station_id: 0147, len(data_list): 1
...
...
{% endhighlight %}

로그 상의 문제 발생 시점이 12월 18일 3시 29분 47초 인데 PID `26607` 프로세스의 시작 시간(3시 29분)과 정확히 일치함을 볼 수 있습니다. `25405` 프로세스가 부모이고 `26607` 프로세스가 자식인데 `26607` 프로세스는 소켓 통신 문제로 멈춰있고(시작만 있고 끝이 없음) 부모인 `25405`는 자식을 기다리기 때문에 멈춰있다는 것을 유추할 수 있습니다.

#### requests

{% highlight python %}
def fetch(stationName):
    request_str = "http://openapi.airkorea.or.kr/openapi/services/rest/"\
        "ArpltnInforInqireSvc/getMsrstnAcctoRltmMesureDnsty?"\
        "stationName={0}&dataTerm=daily&pageNo=1&numOfRows=1&"\
        "ServiceKey={1}&ver=1.3".format(stationName, SERVICE_KEY)

    response_str = requests.get(request_str).content
    return response_str
{% endhighlight %}

어떤 부분이 소켓 통신을 사용하는지 소스 코드를 확인해보니 `requests.get`을 사용하는 부분이 있습니다. 왜 멈췄을까 고민하던 중 인터넷을 찾아보니 `requests.get`의 timeout 옵션의 기본 값이 None이라는 것과 이 경우 문제가 발생하면 하염 없이 영원히 기다리게 된다는 것을 알게되었습니다.

관련 링크 : <https://stackoverflow.com/questions/17782142/why-doesnt-requests-get-return-what-is-the-default-timeout-that-requests-get>

airkorea_fetcher의 경우 timeout을 지정하지 않았기 때문에 문제가 발생했는데도 멈춰있는 상태가 되었습니다. 확인해보니 비슷한 역할을 하는 다른 프로그램에도 같은 문제가 있어 `requests.get`을 사용하는 모든 부분에 timeout을 지정해주기로 했습니다.

<br/>

### 마치며

멈춰 있는 Python 프로세스의 문제 원인 분석 및 해결 방안 도출 과정을 정리해보았습니다. `ps -ef`명령을 사용하여 PID, 실행 시간 등의 프로세스 실행 정보를 알아낸 뒤 `strace`와 `GDB`를 이용하여 멈춤 현상의 원인이 되는 `system call`을 확인했습니다. 알아낸 정보들과 로그를 대조하여 소스 코드의 어떤 부분이 문제가 되는지 유추하였고 그에 대한 해결책을 도출하였습니다.

Python 뿐만 아니라 Linux 운영체제 상에서 실행되는 모든 프로세스의 멈춤 원인 파악에 유용하게 쓰일 수 있는 방법으로 비슷한 문제를 겪고 있는 분들께 도움이 되길 바랍니다.
