---
layout: post
title: 그래서 spring Transactional은 무엇인가?
categories: [spring]
tags: [spring, transactional]
---

spring에서 각 transaction를 묶어주고 관리해주는 역할의 `@Transactional`를 알아보자.

## 1. transaction
transaction은 DB의 상태 변경을 뜻한다. 코드도 git에 commit 하듯 DB도 변경점에 대한 savepoint를 남기며, 이는 rollback 가능한 지점을 뜻한다.

### ACID
흔히 `애시드`라고 불리는 transaction의 성질 4가지가 있다.

* Atomicity(원자성): 한 transaction 내에서 실행한 작업들은 하나의 단위로 처리
* Consistency(일관성): transaction은 일관성 있는 데이터베이스 상태를 유지
* Isolation(격리성): 동시에 실행되는 transaction들이 서로 영향을 미치지 않도록 격리
* Durability(영속성): transaction을 성공적으로 마치면 결과가 항상 저장

## 2. @Transactional
spring에서는 transaction을 구현할 수 있도록 여러가지 형태로 지원한다. 그 중 가장 쉽게 사용할 수 있고 많이 사용하는 annotation 형태의 선언적 방법인 `@Transactional` 을 알아본다. spring에서 @Transactional을 사용하기 위한 방법, DB, ORM 설정 등의 내용은 다루지 않는다.

### proxy

### options
* propagation: 동작 도중 다른 transaction을 호출할 때, 어떻게 할 것인지 지정하는 옵션(전파 수준)
* isolation: transaction에서 일관성 없는 데이터 허용 수준을 설정(격리 수준)
* noRollbackFor: 특정 예외 발생 시 rollback이 동작하지 않도록 설정
* rollbackFor: 특정 예외 발생 시 rollback이 동작하도록 설정
* timeout: 지정한 시간 내에 메소드 수행이 완료되지 않으면 rollback이 동작하도록 설정
* readOnly: transaction을 읽기 전용으로 설정

`@Transactional`의 다양한 속성 중 `propagation`, `isolation`에 대해서 좀 더 자세히 알아보자.

## 3. propagation
spring에서 제공하는 @Transactional은 6개의 propagation 설정을 제공한다. 각각의 설정에 따른 동작을 간단한 예시 코드와 함께 알아보자. 예시로 진행할 코드는 아래와 같으며 DB는 mysql, ORM은 JPA를 사용한다. 

제과(빵) 정보를 bulk 입력할때 bread 및 log를 단건으로 각각 입력하는 코드이다. 사실 실제 bulk 처리는 이렇게 건바이건 처리 하지 않지만, 부모/자식 transaction 처리를 간단하게 보여주기 위해 일부러 묶었다. 참고로 각각의 propagation 전파 설정을 코드로 설명할 때에는 function 부분만 기재한다.

{% highlight kotlin %}
@Service
class BreadBulkService(
    val breadLogService: BreadLogService,
    val breadService: BreadService
) {
    fun bulkSave(to: Int, from: Int) {
        (to..from).forEach {
            val bread = breadService.saveBread(it)
            breadLogService.saveLog(bread)
        }
    }
}
{% endhighlight %}

{% highlight kotlin %}
@Service
class BreadService(
    val breadRepository: BreadRepository
) {
    fun saveBread(key: Int): Bread {
        return breadRepository.save(Bread(key, "bread$key"))
    }
}
{% endhighlight %}

{% highlight kotlin %}
@Service
class BreadLogService(
    private val breadLogRepository: BreadLogRepository
) {
    fun saveLog(bread: Bread) {
        val log = BreadLog(log = BreadConverter.toResponse(bread).toString())
        breadLogRepository.save(log)
    }
}
{% endhighlight %}

### REQUIRED
부모 transaction이 존재한다면 부모 transaction에 합류시키며, 그렇지 않다면 새로운 transaction을 만든다. 즉 부모 또는 자식에서 exception이 발생된다면 자식과 부모 transaction에 관련된 테이블의 savepoint는 모두 rollback 된다.

{% highlight kotlin %}
@Transactional(propagation = Propagation.REQUIRED)
fun bulkSave() {
    (1..10).forEach {
        val bread = breadService.saveBread(it)
        breadLogService.saveLog(bread)
    }
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional
fun saveBread(key: Int): Bread {
    return breadRepository.save(Bread(key, "bread$key"))
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional(propagation = Propagation.REQUIRED)
fun saveLog(bread: Bread) {
    if (bread.id % 7 == 0) throw RuntimeException("7번째 강제 rollback")

    val log = BreadLog(log = BreadConverter.toResponse(bread).toString())
    breadLogRepository.save(log)
}
{% endhighlight %}

```
java.lang.RuntimeException: 7번째 강제 rollback
```
부모 transaction 기준 자식에 있는 `log` transaction에서 강제로 발생시킨 예외에 의해 transaction 기준으로 부모 하위에 있는 `bread`, `log` 테이블 모두 rollback 되었다. 따로 코드상 언급하지 않지만, 부모 transaction인 `bulckSave` 메소드에서 예외를 발생 시켜도 마찬가지로 두 테이블 모두 rollback 된다. 이는 가장 기본적인 전파 옵션으로, propagation 설정을 따로 하지 않아도 `REQUIRED` 설정값으로 동작한다. 여러 write query가 진행될 때 예외 발생 시 일관성 있게 모든 테이블의 savepoint를 rollback 하고 싶을 때 사용하면 된다.

### REQUIRES_NEW
무조건 새로운 transaction을 생성한다. 즉 부모 또는 자식에서 exception이 발생된다면 각자에 해당되는 transaction만 rollback 한다. 사실상 각 transaction은 독립적인 구조라 생각하면 된다.

{% highlight kotlin %}
@Transactional
fun bulkSave() {
    (1..10).forEach {
        val bread = breadService.saveBread(it)
        breadLogService.saveLog(bread)
    }
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional
fun saveBread(key: Int): Bread {
    return breadRepository.save(Bread(key, "bread$key"))
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional(propagation = Propagation.REQUIRES_NEW)
fun saveLog(bread: Bread) {
    if (bread.id % 7 == 0) throw RuntimeException("7번째 강제 rollback")

    val log = BreadLog(log = BreadConverter.toResponse(bread).toString())
    breadLogRepository.save(log)
}
{% endhighlight %}

```
java.lang.RuntimeException: 7번째 강제 rollback
```
이번에도 자식 `log` transaction에서 강제로 예외를 발생시켰다. 부모 transaction 내에 있는 두개의 자식 transaction 중 `bread` 테이블은 모두 rollback 되었지만, `log` 테이블은 예외가 발생된 7번째 정보 전까지 1~6의 정보는 테이블에 남아있다. 이는 `REQUIRES_NEW` 설정에 의해 반복되는 save는 각각 transaction을 생성하고, 해당 transaction이 끝나면 모든 지점들을 commit 하기 때문이다. 이해관계가 있는 다른 transaction에서 실패가 나더라도 자신은 꼭 입력정보를 남겨야할때 사용하면 좋다. 특히 log 기록과 같은 히스토리성 정보에 적합할 수 있다.

### MANDATORY
무조건 부모 transaction에 합류시킨다. 부모 transaction이 존재하지 않는다면 예외를 발생시킨다.

{% highlight kotlin %}
@Transactional
fun bulkSave() {
    (1..10).forEach {
        val bread = breadService.saveBread(it)
        breadLogService.saveLog(bread)
    }
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional
fun saveBread(key: Int): Bread {
    return breadRepository.save(Bread(key, "bread$key"))
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional(propagation = Propagation.MANDATORY)
fun saveLog(bread: Bread) {
    if (bread.id % 7 == 0) throw RuntimeException("7번째 강제 rollback")

    val log = BreadLog(log = BreadConverter.toResponse(bread).toString())
    breadLogRepository.save(log)
}
{% endhighlight %}

```
org.springframework.transaction.IllegalTransactionStateException: No existing transaction found for transaction marked with propagation 'mandatory'
```

이전과는 결과가 달라졌다. 의무적이라는 뜻 그대로 `MANDATORY`는 부모 trasaction 없이 단독적으로 trasaction을 생성할 경우 이를 강제로 막을 수 있는 설정이다. 만약 부모 trasaction이 있다면 `REQUIRED`와 동일하게 동작한다. 특정 입력 기능을 단독적으로 사용하지 못하도록 interface 처럼 강제화 할 수 있는 효과가 필요할 때 사용하면 좋다.

### NESTED
`NESTED`라는 단어 그대로 중첩이 있는 transaction의 중첩이 있는 경우 사용한다. 

{% highlight kotlin %}
@Transactional
fun bulkSave() {
    (1..10).forEach {
        val bread = breadService.saveBread(it)
        breadLogService.saveLog(bread)
    }
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional
fun saveBread(key: Int): Bread {
    return breadRepository.save(Bread(key, "bread$key"))
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional(propagation = Propagation.NESTED)
fun saveLog(bread: Bread) {
    if (bread.id % 7 == 0) throw RuntimeException("7번째 강제 rollback")

    val log = BreadLog(log = BreadConverter.toResponse(bread).toString())
    breadLogRepository.save(log)
}
{% endhighlight %}

```
JpaDialect does not support savepoints - check your JPA provider's capabilities
```
띠용. 갑자기 JPA 에러는 무엇인가? `NESTED`는 JDBC 3.0 이후부터 적용된다 라는 특징이 있으며, JPA를 사용하는 경우 dirty check와 같은 변경감지를 통해서 update를 최대한 지연해서 발행하는 방식을 사용하기 때문에 중첩된 transaction 경계를 설정할 수 없어 지원하지 못한다고 한다. 관련해서는 부가설명으로 이해를 조금 도울 수 있도록 한다.
* 부모 transaction이 없다면 자식 transaction이 새로운 transaction을 생성함
* 부모 transaction이 있다면 중첩 transaction을 생성함
* 중첩 transaction이 끝나도 모든 commit은 부모 transaction의 끝에서 시행됨
* 중첩 transaction의 내부에서 예외가 발생되어도 부모 transaction까지 rollback을 전파하지 않음
* 부모 transaction의 내부에서 예외가 발생하면 모든 transaction은 rollback함

즉, 먼저 시작된 부모 transaction은 내부 transaction에게 commit, rollback에 영향을 줄 수 있지만 내부 transaction의 예외가 부모에게 영향을 끼치진 않는다는 것이다.

### SUPPORTS
간단하게 말로 설명될 것 같은 설정이다. `SUPPORTS`는 부모 transaction이 있다면 합류하는 의미로, 진행중인 부모 transaction이 없다면 transaction을 생성하지 않는다. 
이와 반대로 `NOT_SUPPORTED`는 부모 transaction이 있던 말던 transaction 없이 진행한다.

### NAVER
말그대로 transaction을 생성하지 않는다. 부모 transaction이 존재한다면  예외를 발생시킨다.

{% highlight kotlin %}
@Transactional
fun bulkSave() {
    (1..10).forEach {
        val bread = breadService.saveBread(it)
        breadLogService.saveLog(bread)
    }
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional
fun saveBread(key: Int): Bread {
    return breadRepository.save(Bread(key, "bread$key"))
}
{% endhighlight %}

{% highlight kotlin %}
@Transactional(propagation = Propagation.NEVER)
fun saveLog(bread: Bread) {
    if (bread.id % 7 == 0) throw RuntimeException("7번째 강제 rollback")

    val log = BreadLog(log = BreadConverter.toResponse(bread).toString())
    breadLogRepository.save(log)
}
{% endhighlight %}

```
org.springframework.transaction.IllegalTransactionStateException: Existing transaction found for transaction marked with propagation 'never'
```
한번도 실무에서 사용해본 적은 없다. 이론상 transaction 처리가 절대 발생하면 안되는 메소드에 실수로 transaction 처리가 되는 것을 방지하기 위함으로 보인다.

## 4. 그래서?


{% include ref.html %}
* <>
* <>
