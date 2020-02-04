---
layout: post
title: "Django Only는 불안정하다."
description: "모킹을 이용한 Defer 추가 쿼리 발생 확인"
date: 2019-10-19
tags: [python]
comments: true
share: true
---

## 개요

프로덕션에서 디비를 조회할 때 어떤 필드를 가져올지 명시하는 것은 사실상 필수입니다.  
메모리 사용률을 줄이면 그에 따른 성능 향상을 기대할 수 있기 때문입니다.  
따라서 Django도 필드를 명시할 수 있는 방법을 제공합니다.

## 목차
1. [필드 명시](#필드_명시)
2. [Only 방식의 불안정함](#only_방식의_불안정함)
3. [메소드 모킹을 이용한 해결 (결론)](#메소드_모킹을_이용한_해결)

<h3 id="필드_명시" style="border-bottom: 1px solid #eee;">1. 필드 명시</h3>

Django에선 QuerySet의 메소드를 통해 필드를 명시합니다.
메소드는 크게 values, values_list, only가 있습니다.  
셋의 차이점은 반환 값입니다. values는 사전, values_lis는 튜플, only는 모델 인스턴스를 반환합니다.
따라서 저는 모델 메소드를 사용하고 싶은 경우에는 only를 사용합니다.

```python
>>> Person.objects.values("name")
[{"name": "junyoung"}, {"name": "nia"}]
>>> Person.objects.only("name")
[<Person junyoung>, <Person nia>]
```

<h3 id="only_방식의_불안정함" style="border-bottom: 1px solid #eee;">2. Only 방식의 불안정함</h3>

만약 only를 사용한 경우 명시하지 않은 필드에도 접근할 수 있습니다.
하지만 처음에 디비를 조회할 때는 읽어오지 않았기 때문에 당연히 추가 쿼리가 발생합니다.
이러한 필드를 deffered field 라고 부릅니다.
이는 의도한 기능이기 때문에 어떤 경고 메시지도 없습니다.

추가 쿼리 때문에 저는 only를 불안정하다고 생각합니다.
로직이 많아지고 복잡해지면 필요한 필드를 정확하게 명시했는지 확신하기 힘들어집니다.
만약 정확히 필드를 명시하지 못했다면 그만큼 추가 쿼리가 발생해 심각한 성능 손실을 발생시킵니다.
하지만 모델 인스턴스를 포기하고 싶지는 않았습니다. 성능만큼 코드 퀄리티도 중요하다고 생각하기 때문입니다.
메소드를 포기하면 코드가 너무 난잡해지는 상황이었습니다.

분명 저와 같은 이슈를 많은 사람들이 겪을 것이란 생각이 들었습니다.
하지만 적절한 라이브러리는 찾지 못했습니다.
저는 보통 이 경우에 오버라이딩할 방법을 알아봅니다. 하지만 라이브러리의 너무 깊은 곳을 바꾸는 위험성과 깔끔한 코드가 떠오르지 않아 포기했습니다.  
그러던 중 테스트를 통해 나름 만족할 수 있는 대안을 찾았습니다.

<h3 id="메소드_모킹을_이용한_해결" style="border-bottom: 1px solid #eee;">3. 메소드 모킹을 이용한 해결</h3>

추가 쿼리가 발생한다면 무조건 호출되는 메소드가 있을 것입니다.
그렇다면 이 메소드가 호출된 경우 에러를 반환하면 됩니다.
해당 메소드를 찾는 것은 Pycharm 디버거와 함께라면 간단합니다.

결론적으로 추가 쿼리가 발생한다면 `refresh_from_db` 가 호출됩니다.
아래 코드는 `refresh_from_db`를 모킹합니다. 그리고 이 메소드가 호출되면 에러를 띄웁니다.
여러 곳에서 자주 사용될 기능이기 때문에 함수로 만들었습니다. 

```
def raise_exception_for_defer_query():
    """Only 에서 정의하지 못한 필드가 호출 되면 에러를 발생시킨다."""

    def raise_exception(fields):
        raise DeferQueryExecutedError(f'Only 에서 정의하지 못한 {fields} 가 호출 되었습니다.')

    # Only 에서 정의하지 않은 필드가 호출 되면 refresh_from_db 메소드가 호출된다.
    return patch.object(Model, 'refresh_from_db', side_effect=raise_exception)
```

```
... 테스트 코드
@raise_exception_for_defer_query()
def test_ab(*_):
    pass  # 만약 추가 쿼리가 발생하면 테스트는 실패한다.
```
