---
layout: post
title: "포트는 프로세스를 특정하기 위한 것이 아니다"
description: "파이썬 Gunicorn WAS 프로세스들이 같은 프로세스를 사용할 수 있는 이유"
date: 2019-10-07
tags: [network, gunicorn]
comments: true
share: true
---

### 목차 

- [개요](#overview)
- [SO_REUSEADDR, SO_REUSEPORT](#SO_REUSEADDR-SO_REUSEPORT)
- [연결용 소켓과 통신용 소켓](#socket-type)
- [소켓은 파일이다](#socket-is-file) 
- [Gunicorn](#gunicorn)
- [결론](#conclusion)
- [참조](#reference)

<h3 id="overview">개요</h3>

위키피디아에선 포트를 다음과 같이 설명합니다.  

```
At the software level, within an operating system, a port is a logical construct that identifies a specific process or a type of network service.
```

다들 잘 아시겠지만 포트가 프로세스를 구분한다는 설명입니다.  
우리가 또 잘 아는 사실이 있는데 파이썬은 GIL 때문에 병렬 실행을 하기 위해선 멀티 프로세싱을 해야 한다는 것입니다.      
그래서 파이썬 WAS 는 멀티 프로세스로 요청을 처리합니다. 즉 멀티 프로세스가 하나의 포트를 사용한다는 것이죠.  
모순이 발생합니다. 어떻게 파이썬 WAS 는 같은 포트를 사용할까요?  

<h3 id="SO_REUSEADDR-SO_REUSEPORT">SO_REUSEADDR, SO_REUSEPORT</h3>

가장 먼저 검색된 결과는 소켓 옵션 중 하나인 SO_REUSEADDR 과 SO_REUSEPORT 입니다.  
이 두 옵션 중 하나를 사용하면 다른 프로세스가 같은 포르틀 사용할 수 있습니다.  

두 옵션이 완전히 똑같다면 두 개로 분리하지 않았겠죠? SO_REUSEADDR 은 우리가 생각하는 것과 조금 다릅니다.  
아시다시피 HTTP 는 TCP 위에서 동작합니다. 그리고 TCP 는 데이터 재전송을 지원하는 등 신뢰성 있는 프로토콜입니다.    
이 특성 때문에 우리가 소켓을 닫아도 일정 시간 동안 소켓이 닫히지 않는데 이를 Time Wait 이라고 합니다.  
아마 프로세스를 죽이고 바로 다시 실행했을 때 `port is already in use` 에러를 본 경험이 있을 텐데 Time Wait 때문입니다.  
SO_REUSEADDR 은 이 Time Wait 인 상태에서 다른 프로세스가 (정확히는 다른 소켓 객체가) 같은 포트를 사용할 수 있게 해줍니다.  

SO_REUSEPORT 가 우리가 생각하는 동작입니다. 이 옵션으로 소켓을 실행하면 다른 프로세스에서도 같은 포트를 사용할 수 있습니다.  

```python
import socket

addr = ('0.0.0.0', 8000)

server_socket = socket.socket()
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEPORT, 1)
server_socket.bind(addr)
server_socket.listen(5)
client = server_socket.accept()
```

해당 코드로 두 개의 파이썬 프로세스를 실행시켜면 `port reused` 에러가 발생하지 않습니다.  
하지만 SO_REUSEPORT 는 Linux 3.9 부터 지원되는 기능입니다.  
그럼 파이썬 WAS 라이브러리의 문서에는 Linux 3.9 이상부터 지원된다는 명세가 있어야하는데 적어도 Gunicorn 문서에는 그런 내용이 없습니다.    
뭔가 다른 방법을 사용하는 것 같습니다.  

<h3 id="socket-type">연결용 소켓과 통신용 소켓</h3>
  
서버 소켓이 클라이언트로에게 요청을 받기 위해선 `listen` 를 통해 연결용 소켓을 만들어야합니다.  
연결용 소켓이 만들어지면 흔히들 말하는 3 way handshake 가 가능해지는데 이 작업이 완료되면 `backlog queue` (`accept queue`) 에 요청이 쌓입니다.    
연결용 소켓으로 `accept` 를 호출하면 통신용 소켓이 반환되고 이제 이 소켓으로 통신을 할 수 있게됩니다.  

네트워크 기기는 동시성을 지원하기 때문에 하나의 연결용 소켓으로 여러개의 통신용 소켓을 만들수있습니다.  
이 때 OS 는 `프로토콜, 클라이언트 주소, 클라이언트 포트, 서버 주소, 서버 포트` 의 조합으로 적절한 소켓에 요청 받은 데이터를 넘겨줍니다.  
GIL 이 없는 언어에선 쓰레드로 동시 요청을 처리할테고 쓰레드는 같은 메모리를 공유하기 때문에 동일한 연결용 소켓을 사용할 수 있습니다.  

<h3 id="socket-is-file">소켓은 파일이다</h3>

아시겠지만 소켓은 파일입니다. 파일이 어떻게 관리되는지 알아야 파이썬 WAS 멀티 프로세스가 어떻게 같은 포트를 공유하는지 알 수 있습니다.  
새로운 파일을 오픈하면 디스크립터 테이블에 행이 하나 더 생기고 이 행의 인덱스를 반환받습니다.  
그리고 디스크립터 테이블은 OS 레벨에서 전역으로 관리되는 파일 테이블을 가리킵니다.  
디스크립터 테이블은 프로세스 각각에 할당되지만 디스크립터 테이블이 가리키는 파일 테이블이 전역이기 때문에 다른 프로세스도 같은 파일을 가리킬 수도 있습니다.  

<h3 id="gunicorn">Gunicorn</h3>

Gunicorn 은 멀티 프로세싱을 위해 `os.fork` 를 사용합니다.   
`os.fork` 는 해당 프로세스와 동일한 내용의 프로세스를 하나 더 만드는 명령입니다.  
Gunicorn 은 연결용 소켓을 만든 다음 `os.fork` 를 호출합니다.  

부모 프로세스와 자식 프로세스는 내용은 동일하지만 독립정인 디스크립터 파일 테이블을 가질 것입니다.  
헌데 `os.fork` 를 통해 만들어진 자식 프로세스는 부모 프로세스가 가리키던 파일 테이블을 가리킵니다.  
즉, 쓰레드를 이용하는 것 처럼 동일한 연결용 소켓을 가지게 되는 것입니다.    
이것이 Gunicorn 에서 WAS 멀티 프로세스들이 같은 포트를 가질 수 있는 이유입니다.  

<h3 id="conclusion">결론</h3>

엄밀히 말해서 포트는 프로세스를 식별하기 위한 용도가 아니고 소켓 객체를 식별하기 위한 용도입니다.  
한 프로세스가 하나의 소켓을 가지는 경우 맞는 설명이지만 파이썬 WAS 처럼 여러 프로세스에서 동일한 소켓을 가질 수 있기 때문입니다.  

<h3 id="reference">참조</h3>

- https://stackoverflow.com/questions/14388706/how-do-so-reuseaddr-and-so-reuseport-differ
- https://www.usna.edu/Users/cs/aviv/classes/ic221/s16/lec/21/lec.html
- https://unix.stackexchange.com/questions/28384/how-can-same-fd-in-different-processes-point-to-the-same-file
- https://stackoverflow.com/questions/670891/is-there-a-way-for-multiple-processes-to-share-a-listening-socket
- https://stackoverflow.com/questions/1694144/can-two-applications-listen-to-the-same-port
