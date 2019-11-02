---
layout: post
title: "포트는 프로세스를 특정하기 위한 것이 아니다"
description: "파이썬 Gunicorn WAS 프로세스들이 같은 포트를 사용할 수 있는 이유"
date: 2019-10-07
tags: [network, gunicorn]
comments: true
share: true
---

<h3 id="overview" style="border-bottom: 1px solid #eee;">목차</h3>

- [개요](#overview)
- [SO_REUSEADDR, SO_REUSEPORT](#SO_REUSEADDR-SO_REUSEPORT)
- [연결용 소켓과 통신용 소켓](#socket-type)
- [소켓은 파일이다](#socket-is-file) 
- [Gunicorn](#gunicorn)
- [결론](#conclusion)
- [참조](#reference)

<h3 id="overview" style="border-bottom: 1px solid #eee;">개요</h3>

파이썬 웹 개발자라면 당연히 알고 있을 법한 내용이 두 가지 있습니다.    

- 파이썬은 GIL 때문에 병렬 실행을 하기 위해선 멀티 프로세싱이 필요하다.  
- 포트는 프로세스를 구분한다.  

이 논리를 조합하면 모순이 생깁니다.  
파이썬 WAS 는 병렬과 동시성 처리를 위해 멀티 프로세스를 사용하는데 이 프로세스들이 하나의 포트를 사용하기 때문입니다.  
어떻게 파이썬 WAS 프로세스들이 같은 포트를 사용하는지 Gunicorn 기준으로 살펴보겠습니다.  

<h3 id="SO_REUSEADDR-SO_REUSEPORT" style="border-bottom: 1px solid #eee;">SO_REUSEADDR, SO_REUSEPORT</h3>

구글에 멀티 프로세스가 같은 포트를 사용하는 법을 검색하면 가장 먼저 나오는 게 소켓 옵션 중 하나인 SO_REUSEADDR 과 SO_REUSEPORT 입니다.  
이 두 옵션 중 하나를 사용하면 다른 프로세스가 같은 포트틀 사용할 수 있습니다.  
하지만 두 옵션이 완전히 똑같다면 굳이 두 개로 분리하지 않갔겠죠?  

#### SO_REUSEADDR

아시다시피 HTTP 는 TCP 프로토콜 위에서 동작합니다. 그리고 TCP 는 데이터 재전송을 지원하는 등 신뢰성을 가지고 있는 프로토콜입니다.  
그렇기 때문에 우리가 소켓을 클로즈해도 일정 시간 동안 소켓이 종료되지 않는데 이 시간을 Time Wait 이라고 합니다.  
WAS 프로세스를 죽이고 바로 다시 실행하면 `port is already in use` 에러가 발생하는 경우가 간혹 있는데 Time Wait 때문입니다.  
SO_REUSEADDR 은 이 Time Wait 인 상태에서 다른 프로세스가 (정확히는 다른 소켓 객체가) 같은 포트를 사용할 수 있게 해줍니다.  

#### SO_REUSEPORT 

SO_REUSEPORT 은 다른 프로세스들이 같은 포트를 사용 가능하게 해주는 옵션입니다.  
아래의 코드로 두 개의 파이썬 프로세스를 실행시키면 `port reused` 에러가 발생하지 않습니다.   

```python
import socket

addr = ('0.0.0.0', 8000)

server_socket = socket.socket()
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
server_socket.bind(addr)
server_socket.listen(5)
client = server_socket.accept()
```

하지만 SO_REUSEPORT 는 Linux 3.9 부터 지원되는 기능입니다.  
그럼 파이썬 WAS 라이브러리의 문서에는 Linux 3.9 이상부터 지원된다는 명세가 있어야 하는데 적어도 Gunicorn 문서에는 그런 내용이 없습니다. 
뭔가 다른 방법을 사용하는 것 같습니다.  

<h3 id="socket-type" style="border-bottom: 1px solid #eee;">연결용 소켓과 통신용 소켓</h3>

멀티 쓰레드에서 어떻게 요청을 동시에 처리하는지 살펴보면 파이썬 WAS 에서 멀티 프로세스가 하나의 포트를 사용하는 방법을 이해하는 데 도움이 됩니다.  

우선 서버 소켓이 클라이언트에게 요청을 받기 위해선 Listen 함수를 호출해 연결용 소켓을 만들어야 합니다.  
연결용 소켓이 만들어지만 OS 는 해당 포트로 요청이 왔을 때 3 way handshake 작업을 실행합니다.  
그리고 이 작업이 완료되면 backlog queue (accept queue) 에 요청이 쌓입니다.  

accept 함수를 통해 backlog queue 에 쌓인 요청을 빼 오면 새로운 소켓인 통신용 소켓을 반환받습니다.  
이 소켓은 \<프로토콜, 클라 주소, 클라 포트, 서버 주소, 서버 포트\> 조합으로 식별됩니다.  
네트워크 기기는 동시성을 지원하기 때문에 여러 연결용 소켓들에게 알맞는 정보를 전달할 수 있습니다.  

멀티 쓰레드는 메모리를 공유하기 때문에 동일한 연결용 소켓으로 여러개의 통신용 소켓을 만들어 낼 수 있습니다.  
Gunicorn 에서도 같은 연결용 소켓을 공유해서 멀티 프로세스에서 동일한 포트를 사용합니다.  
Gunicorn 은 어떻게 같은 연결용 소켓을 가질까요?  
IPC?, 공유 메모리?, PIPE? 같이 한 번 알아봅시다.  

<h3 id="socket-is-file" style="border-bottom: 1px solid #eee;">소켓은 파일이다</h3>

소켓은 파일입니다. 파일이 어떻게 관리되는지 알면 어떻게 같은 연결용 소켓을 공유하는지 알 수 있습니다.  
새로운 파일을 오픈하면 디스크립터 테이블에 행이 추가되고 이 행의 인덱스를 반환받습니다.  
그리고 디스크립터 테이블은 OS 레벨에서 전역으로 관리되는 파일 테이블을 가리킵니다.  
그렇습니다. 디스크립터 테이블은 프로세스 각각에 할당되지만 디스크립터 테이블이 가리키는 파일 테이블이 전역이기 때문에 다른 프로세스에서도 같은 파일 테이블을 가리킬 수 있습니다.  

<h3 id="gunicorn" style="border-bottom: 1px solid #eee;">Gunicorn</h3>

Gunicorn 은 멀티 프로세싱을 위해 os.fork 를 사용합니다.  
이 명령은 실행 중인 프로세스와 동일한 내용의 프로세스를 하나 더 만드는 명령입니다.  
이 때 Gunciron 은 os.fork 를 호출하기 전에 연결용 소켓을 미리 만들어둡니다.  
그럼 부모 프로세스와 자식 프로세스의 내용은 동일하며 독립적인 디스크립터 테이블이 만들어집니다.  
하지만 이 디스크립터 테이블이 가리키는 파일 테이블은 동일합니다.  
그래서 멀티 프로세스에서도 멀티 쓰레드처럼 같은 연결용 소켓을 가질 수 있습니다.  

<h3 id="conclusion" style="border-bottom: 1px solid #eee;">결론</h3>

그래서 저는 포트가 프로세스를 식별하기 위한 용도라는 설명은 조금 틀렸다고 생각합니다.  
포트는 엄밀히 말해 소켓을 식별하기 위한 용도입니다. 그리고 이 소켓은 다른 프로세스에서도 사용할 수 있죠.  
처음에 Gunicorn 코드를 볼 때는 같은 포트를 사용하기 위한 별다른 코딩이 없는 것 같아서 띠용 했는데 파일이 저렇게 관리된다는 점이 흥미로웠습니다.  

<h3 id="reference" style="border-bottom: 1px solid #eee;">참조</h3>

- https://stackoverflow.com/questions/14388706/how-do-so-reuseaddr-and-so-reuseport-differ
- https://www.usna.edu/Users/cs/aviv/classes/ic221/s16/lec/21/lec.html
- https://unix.stackexchange.com/questions/28384/how-can-same-fd-in-different-processes-point-to-the-same-file
- https://stackoverflow.com/questions/670891/is-there-a-way-for-multiple-processes-to-share-a-listening-socket
- https://stackoverflow.com/questions/1694144/can-two-applications-listen-to-the-same-port
