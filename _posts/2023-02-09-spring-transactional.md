---
layout: post
title: 그래서 spring Transactional은 무엇인가?
categories: [spring]
tags: [spring, transactional]
---

spring에서 각 transaction를 묶어주고 관리해주는 역할의 `@Transactional`를 알아보자.

## 1. transaction
transaction은 DB의 상태 변경을 뜻한다. 코드도 git에 commit 하듯 DB도 변경점에 대한 savepoint를 남기며, 이는 rollback 가능한 지점을 뜻한다.

흔히 `ACIS(애시드)`라고 불리는 transaction의 성질 4가지가 있다.

* Atomicity(원자성): 한 transaction 내에서 실행한 작업들은 하나의 단위로 처리
* Consistency(일관성): transaction은 일관성 있는 데이터베이스 상태를 유지
* Isolation(격리성): 동시에 실행되는 transaction들이 서로 영향을 미치지 않도록 격리
* Durability(영속성): transaction을 성공적으로 마치면 결과가 항상 저장

## 2. @Transactional
spring에서는 transaction을 구현할 수 있도록 여러가지 형태로 지원한다. 그 중 가장 쉽게 사용할 수 있고 많이 사용하는 annotation 형태의 선언적 방법인 `@Transactional` 을 알아본다. spring에서 @Transactional을 사용하기 위한 방법, DB, ORM 설정 등의 내용은 다루지 않는다.

### proxy
@Transactional을 적용한 기본적인 예제 코드를 보자. 참고로 @Transactional는 proxy를 구현하기 위해 메소드 override가 필요하다. 즉, private 메소드는 @Transactional 적용이 불가하다.

`@Transactional 적용`
{% highlight kotlin %}
@Service
class BreadBulkService(
    val breadLogService: BreadLogService,
    val breadService: BreadService
) {
    @Transactional
    fun bulkSave() {
        (1..10).forEach {
            val bread = breadService.saveBread(it)
            breadLogService.saveLog(bread)
        }
    }
}
{% endhighlight %}
spring의 @Transactional은 `AOP`와 `Proxy`를 사용하여 transaction을 구현한다. proxy는 객체에 대한 대리자 역할을 하는 객체로, 실제 객체를 감싸서 호출되는 메서드의 호출을 가로채거나 다른 작업을 수행할 수 있다. 그렇다면 @Transactional 없이 proxy에 의해 구현되는 코드를 살펴보자.

`transaction proxy`
{% highlight kotlin %}
@Service
class BreadBulkServiceProxy(
    val breadLogService: BreadLogService,
    val breadService: BreadService,
    val transactionManager: PlatformTransactionManager
) {
    fun bulkSave() {
        val transactionTemplate = TransactionTemplate(transactionManager)
        transactionTemplate.execute { status ->
            try {
                (1..10).forEach {
                    val bread = breadService.saveBread(it)
                    breadLogService.saveLog(bread)
                }
                status.setRollbackOnly()
            } catch (e: Exception) {
                throw RuntimeException("Transaction failed", e)
            }
        }
    }
}
{% endhighlight %}

spring에서는 AOP를 사용하여 @Transactional이 붙은 메서드를 찾아내고 이를 실행할 때 proxy를 생성하며, transaction 시작 및 메서드 실행 완료 후 commit 또는 rollback을 수행할 수 있는 코드를 생성한다. 이때, proxy 객체는 원본 객체와 같은 interface를 구현한다. 그래서 클라이언트 코드에서 @Transactional만으로 다른 별도의 처리 없이 transaction을 구현할 수 있다. 이렇게 AOP를 적용하여 구현된 클래스의 interface를 proxy 객체로 구현하여 코드를 끼워넣는 방식을 `JDK Proxy` 라고 한다. 

논외로, JDK Proxy는 replication을 사용하여 주입하는 방식으로 성능적으로 좋지 않다. 그래서 spring boot의 경우 기본적으로 proxy 객체를 생성할 때 CGLib(code generator library) 방식으로 byte 코드를 조작하여 proxy 객체를 생성하고 주입한다.

### options
spring @Transactional는 다양한 속성 정보를 설정 할 수 있다.

* propagation: 동작 도중 다른 transaction을 호출할 때, 어떻게 할 것인지 지정하는 옵션(전파 수준)
* isolation: transaction에서 일관성 없는 데이터 허용 수준을 설정(격리 수준)
* noRollbackFor: 특정 예외 발생 시 rollback이 동작하지 않도록 설정
* rollbackFor: 특정 예외 발생 시 rollback이 동작하도록 설정
* timeout: 지정한 시간 내에 메소드 수행이 완료되지 않으면 rollback이 동작하도록 설정
* readOnly: transaction을 읽기 전용으로 설정

 여러 속성 중 transaction의 성질인 ACID와 긴밀하게 연관된 `propagation`, `isolation`에 대해서 좀 더 자세히 알아보자.

## 3. propagation
spring에서 제공하는 @Transactional은 6개의 propagation 설정을 제공한다. 각각의 설정에 따른 동작을 위에서 언급한 예시 코드와 함께 알아보자. DB는 mysql, ORM은 JPA를 사용했다.

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
부모 transaction이 존재한다면 부모 transaction에 합류 시키며, 그렇지 않다면 새로운 transaction을 만든다. 즉 부모 또는 자식에서 exception이 발생된다면 자식과 부모 transaction에 관련된 테이블의 savepoint는 모두 rollback 된다.

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

이전과는 결과가 달라졌다. 의무적이라는 뜻 그대로 `MANDATORY`는 부모 trasaction 없이 단독적으로 trasaction을 생성할 경우 예외를 발생시켜, 이를 강제로 막을 수 있는 설정이다. 만약 부모 trasaction이 있다면 `REQUIRED`와 동일하게 동작한다. 특정 입력 기능을 단독적으로 사용하지 못하도록 강제화 할 수 있는 효과가 필요할 때 사용하면 좋다.

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
말그대로 transaction을 생성하지 않는다. 부모 transaction이 존재한다면 예외를 발생시킨다.

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
한번도 실무에서 사용해본 적은 없다. 이론상 transaction 처리가 절대 발생하면 안되는 메소드에 실수로 transaction에 합류 되는 것을 방지하기 위함으로 보인다.

## 4. isolation
isolaction은 먼저 언급한 ACID 속성중 하나이다. 격리성 또는 고립성 이라고 부르는 isolation은 말그대로 transaction끼리 서로에게 얼마나 격리되어 있는지를 나타낸다. 쉽게 얘기하면 transaction이 다른 transaction에서 변경한 데이터를 어느정도 수준에서 볼 수 있도록 할 지 결정하는 요소이다. DB는 ACID의 특징에 따라 각 transaction이 독립적인 수행을 할 수 있도록 `locking`을 통해 transaction의 수행간에 다른 transaction이 관여하지 못하도록 제어한다. locking을 하게되면 동시처리 능력이 떨어지므로 결국 전체적인 성능이 떨어질 수 밖에 없고, 너무 느슨하게 lock의 범위를 줄인다면 잘못된 값을 읽고 쓰는 문제가 발생할 수 있다. 이에따라 ANSI/ISO 표준에는 4가지의 isolation level을 정의했다.

### READ_UNCOMMITTED
level0의 가장 낮은 transaction 격리 수준이며 대부분의 DBMS가 사용하지 않거나 권장하지 않는다.
select가 실행할 때 shared lock이 걸리지 않는다.

`dirty read` 현상이 발생될 수 있다.

>dirty read: transaction의 변경 내용이 commit, rollback에 관계없이 다른 transaction에 적용됨

* A transaction에서 wookey의 성적을 80점 -> 95점으로 변경
* B transaction에서 학점을 정정하기 위해 wookey의 성적을 95점으로 조회하여 처리(dirty read)
* 이때 A transaction에 예외가 발생하여 A transaction이 rollback 됨
* 하지만 여전히 B transaction은 wookey의 성적을 95점으로 생각하고 이후 비즈니스 로직을 계속 수행


### READ_COMMITTED
level1의 transaction 격리 수준이며 대부분의 DBMS, Oracle 등에서 기본적으로 사용한다.
select가 실행할 때 shared lock이 걸린다.
한 transaction의 내용이 commit 되어야만 다른 transaction에서 조회가 가능하여, dirty read는 발생하지 않는다. 
select시 실제 테이블 값이 아니라 undo 영역의 backup recode를 가져온다. 동일한 transaction에서 select 쿼리를 실행했을 때 항상 같은 결과를 보장해야 한다는 `repeatable read` 정합성에 어긋나는 `non repeatable read` 현상이 발생될 수 있다.

> non repeatable read: 같은 transaction 내 동일한 값에 대해 commit이 일어나기 전과 commit 후 값이 달라짐

* A transaction에서 wookey의 성적을 조회하니 80점으로 나옴
* B transaction에서 wookey의 성적을 80점 -> 95점으로 변경(commit 완료)
* A transaction에서 wookey의 성적을 다시 조회하니 90점으로 나옴

얼핏보면 맞는 동작처럼 보일 수 있다. 하지만 한 transaction 내에서 select에 대한 일관된 데이터를 반환하지 않는다는 것은 문제를 발생시킬 수 있다. 하나의 transaction 내에서 동일한 데이터를 여러번 읽고 변경하는 비즈니스 로직이 있다면 데이터 일관성이 깨질 수도 있다.


### REPEATABLE_READ
level2의 transaction 격리 수준이며 mysql DBMS에서 사용한다.
transaction이 시작되기 전 commit 된 정보에 대해서만 select가 가능하여 transaction이 끝날때 까지 select가 실행할 때 shared lock이 걸린다. 그래서 transaction 범위 내에서 조회한 내용이 항상 동일함을 보장함으로 dirty, non repeatable read은 발생하지 않는다. 다만 transaction의 시작 시점의 데이터를 계속 관리하고 일관성을 보장해야하기 때문에 transaction의 시간이 길수록 다중 버전 동시성 제어인 MVCC(multi-version concurrency control)를 관리해야 해야 하므로 성능적 단점이 생긴다. 또한 `phantom read`이 발생될 수 있다.

> phantom read: 한 transaction에서 일정 범위의 레코드를 두번 이상 읽을때 처음에 없던 결과가 추가/삭제 되어 결과가 달라져 보이는 현상

* A transaction에서 성적이 80점 이상인 집단을 조회하니 13명이 나옴(남학생: 6명 / 여학생: 7명)
* B transaction에서 남학생인 wookey의 성적을 80점 -> 75점으로 변경(commit 완료)
* A transaction에서 성적이 80점 이상인 집단의 남학생을 조회하니 5명이 나옴

이 또한 얼핏 맞는 동작처럼 보일 수 있다. 하지만 시작 전 select 동일한 정보에 대해서는 일관성을 유지하지만 B transaction이 commit 되어 A transaction에 조회 범위에 영향을 주어 한 transaction 내에서 마치 결과가 삭제(상황에 따라 추가)된 것 같이 동작되어 보이는 phantom read를 발생 시킬 수 있다.

### SERIALIZABLE
level3의 가장 높은 transaction 격리 수준이며 거의 사용하지 않는다.
select, write 실행에 모두 lock을 선점한다. dirty read, non repeatable read, phantom read와 같은 문제가 발생하지 않으며, 일관성을 유지시킬 수 있는 가장 강력한 방법이다. 그 만큼 lock을 최대한으로 사용하기 때문에 동시 처리능력이 급격하게 떨어지고 결국 성능에 문제가 발생될 수 있다.


## 5. 그래서?
![elasticsearch shard replica]({{site.url}}/assets/images/posts/spring-transactional-01.png)

그래서 `@Transactional`은 spring에서 편리하게 transaction 처리를 적용할 수 있는 매우 유용한 방법이며, DB 관련 로직에서는 떼놓을 수 없는 손님이다.
> @Transactional은 본인이 사용하는 DBMS와 처리 로직에 따라 전파수준과 고립수준을 잘 고려하여 사용하자.

{% include ref.html %}
* <https://akasai.space/db/about_isolation/>
* <http://wiki.gurubee.net/pages/viewpage.action?pageId=21200923>
