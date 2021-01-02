---
layout: post
title: "모델 캐시"
description: "모델로 캐시를 관리하는 방법의 장점과 단점"
date: 2020-09-19
tags: [database]
comments: true
share: true
---

## 개요

모델 캐시는 모델의 ID를 키로, 모델을 값으로 저장하는 캐시 방법을 말합니다.
예전에는 모델 캐시를 안 좋게 봤습니다. 그래서 주로 응답 캐시와 쿼리 캐시를 사용해왔습니다.  
지금 회사는 원래부터 모델 기반 캐시를 사용하는 프로젝트가 있습니다.  
실제로 접해보니 장점도 보이게 되어서, 글로 정리합니다.

### ID 조회

어떤 모델의 리스트를 응답하는 API가 있습니다. 모델 캐시의 경우 리스트에 포함될 PK를 알아야 합니다.  
따라서 디비 쿼리가 필요합니다. 다만 PK 리스트만 쿼리하면 됩니다.  
ID만 쿼리하면 Index only scan 하기 쉽습니다. 당연히 전체 필드를 조회하는 것보다 훨씬 빠릅니다.  
심지어는 [DB만 사용할 경우에도 ID를 쿼리하고 그 결과값으로 전체 필드를 조회하는 것이, 처음부터 전체 필드를 조회하는 것보다 빠르다는 의견](https://blog.codinghorror.com/all-abstractions-are-failed-abstractions/)도 있습니다.

Index only scan이 성공하려면 select, where, group by, order by 에서 사용하는 필드들이 인덱스에 있으며, 인덱스 컬럼 순서도 맞아야합니다.
모든 필드를 Index 에 넣는 것도 현명하지 않습니다. 따라서 Index only scan 하기는 쉽지 않거나 불가능합니다.
[late row lookup](https://explainextended.com/2009/10/23/mysql-order-by-limit-performance-late-row-lookups/)을 통해 최대한 Index only scan을 유효하게 할 수는 있습니다.
예를 들어 다음의 쿼리는 late row lookup을 충족합니다.

```sql
select id from (
    select id
    from users
    where index_field1 = 'foo' and index_field2 = "bar"
    order by id
    offset 10000
) o
JOIN o.id = users.id
where users.not_index_field1 is False
limit 500
```

해당 쿼리는 원본 테이블을 히트하는 경우를 최소화합니다.  

### 높은 공유성

응답 캐시는 한 API에 한정됩니다. 쿼리 캐시는 한 쿼리에 한정됩니다.  
반면 모델 캐시는 모델의 PK로 캐싱하기 때문에 모델이 필요한 어떤 곳에서든 공유됩니다.  
따라서 같은 모델이, 여러 API에서 쓰이는 경우 유리할 수 있습니다.  

### 쉬운 캐시 무효화

모델에 수정이 발생하면 캐시를 무효화해야 합니다. (혹은 TTL을 도매인 상황에 맞게 짧게 가져갑니다.)  
모델 캐시는 대응하는 PK의 캐시키를 하나 날리면 됩니다.  
반면 응답 캐시 혹은 쿼리 캐시의 경우 한 모델이 여러 키에 산재 되어 있습니다. 그래서 무효화는 비효율적이거나 불가능에 가깝습니다.
다만 레이스 컨디션을 감안해야합니다. 모델이 삭제되어 캐시를 지웠지만, 엇비슷하게 읽기가 발생해 다시 캐시 설정이 될 수 있습니다.  
따라서 도매인 상황에 맞게 적절하게 사용해야 합니다.    

### 마무리

프로젝트 초반에는 캐시에 의존하지 않고, 효율적인 쿼리를 만들기 위해 노력할 것 같습니다.
쿼리라 함은 비단 RDS만을 말하는게 아니라 NoSQL (클러스터디비)도 합쳐서요.  
그러다가 디비 쓰르풋 이슈가 생기면 짧은 응답 캐시를 붙일 것 같습니다.  
그럼에도 쿼리가 느려지면 앞서 말한 것처럼 Index only scan을 유도하고, 도매인에 따라선 추가로 모델 캐시를 붙일 수도 있을 것 같습니다.  
