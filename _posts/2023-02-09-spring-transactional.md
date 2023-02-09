---
layout: post
title: 그래서 spring @Transactional은 무엇인가?
categories: [spring]
tags: [spring, transactional]
---

spring에서 각 transaction를 묶어주고 관리해주는 역할의 `@Transactional`를 알아보자.
<hr>

## 1. transaction
transaction은 DB의 상태 변경을 뜻한다. 코드도 git에 commit 하듯 DB도 변경점에 대한 commit을 남기며, 이들을 각각 transaction이라 부를 수 있다.

### ACID
흔히 `애시드`라고 불리는 transaction의 성질 4가지가 있다.

* Atomicity(원자성): 한 트랜잭션 내에서 실행한 작업들은 하나의 단위로 처리
* Consistency(일관성): 트랜잭션은 일관성 있는 데이터베이스 상태를 유지
* Isolation(격리성): 동시에 실행되는 트랜잭션들이 서로 영향을 미치지 않도록 격리
* Durability(영속성): 트랜잭션을 성공적으로 마치면 결과가 항상 저장

## 2. @Transactional
spring에서는 transaction을 구현할 수 있도록 여러가지 형태로 지원한다. 그 중 가장 쉽게 사용할 수 있고 많이 사용하는 annotation 형태의 선언적 방법인 `@Transactional` 을 알아본다.

### proxy
### options

## 4. 그래서?


{% include ref.html %}
* <>
* <>
