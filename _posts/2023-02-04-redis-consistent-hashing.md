---
layout: post
title: 그래서 redis cluster hash slot은 무엇인가?
categories: [redis]
tags: [redis, redis cluster, hash slot, consistent hashing]
---

redis cluster의 node간에 key 분산 방식으로 사용되는 hash slot에 대해 알아보며, nosql의 분산 key 파티셔닝 기법의 시초가 되는 consistent hashing도 함께 알아본다.

<hr>

## 1. redis cluster
redis cluster은 redis 3.0 버전 이상부터 추가되었으며 데이터 동기화, 복제, fail-over 등을 지원할 수 있는 기능이 추가된 향상된 redis라고 생각하면 된다. 특히 fail-over 관점에서 node의 장애 또는 통신 문제 등 master-slave 및 replica 기능을 통해 수준높은 가용성을 제공할 수 있다.

nosql은 각 node에 key를 할당할 때 특정 node에 집중되지 않고 분산처리를 위해 `consistent hashing` 이라고 하는 파티셔닝 방법을 대게 사용한다. consistent hashing은 redis 3.0 이전에는 다양한 버전에서 채택했다. 이후 redis cluster가 자리잡으면서 `hash slot` 이라는 방법을 선택했다. 그렇다면 먼저 consistent hashing은 무엇인지 알아보자.

## 2. consistent hashing
consistent hashing은 node의 수에 의존하지 않고 추가/삭제된 node가 있을때 재분산 해야하는 key의 수를 최소화 하기 위한 방법이다. hash ring이라 불리는 추상적인 원 또는 링 형태에 CRC-32라고 하는 2의32 거듭 제곱에 따른 hash연산의 결과에 따라 node를 할당하고, 인입되는 key에 대해서 동일한 hash 연산을 통해 어떤 node에 배치할 지 결정해주는 key 분산 처리 방법이다.

### 일반 hashing
그렇다면 일반적인 hashing 방법에 의해 key를 분산하면 어떻게 되는걸까?

### 시나리오 및 코드
글 설명만으로는 절대 이해하기 쉽지 않을것이다. 아래와 같은 시나리오를 그림으로 살펴보자.
* node의 수: 3
* 유입 key의 수: 100


## 3. hash slot







## 4. 그래서?

Redis의 Hash Slot 과 Consistent Hashing의 차이는 아래와 같습니다.

Redis의 Hash Slot

Redis의 Hash Slot은 매핑 된 키-값 쌍을 각각의 노드에 분배하기 위해 사용됩니다. 기본적으로 전체 키 슬롯의 범위는 0 ~ 16383 입니다. 노드는 슬롯의 구간을 할당 받아 관리합니다. 이는 노드가 늘어날 때 슬롯의 범위를 분배하는데 도움을 줍니다.

Consistent Hashing

Consistent Hashing은 노드가 늘어나거나 줄었을 때 데이터가 노드간 고르게 균등하게 분배되어 있도록 하기 위해 만들어진 해시 알고리즘입니다. 각 노드는 해시 값의 범위를 할당 받고, 모든 키는 해시 함수를 통해 연관된 노드로 라우팅됩니다.


{% include ref.html %}
* <https://severalnines.com/blog/hash-slot-vs-consistent-hashing-redis/>
* <https://en.wikipedia.org/wiki/Consistent_hashing#History>
