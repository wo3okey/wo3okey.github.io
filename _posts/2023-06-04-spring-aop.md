---
layout: post
title: 그래서 spring AOP proxy란 무엇인가?
categories: [spring]
tags: [spring, transactional, AOP]
---
Spring에서 AOP(Aspect Oriented Programming)는 횡단 관심사를 분리할 수 있는 핵심적인 기술이다. @Transactional의 경우 어노테이션 하나만으로 트랜잭션 구현 기능을 사용할 수 있는데, 이는 Spring에서 AOP Proxy에 의해 제공되기 때문이다.

# AOP
AOP(Aspect Oriented Programming)는 컴퓨터 패러다임의 일종으로, 관점 지향 프로그래밍이라 불린다. 이러한 AOP의 메커니즘은 프로그램을 관심사(Concerns) 기준으로 크게 핵심 관심사와 횡단 관심사로 분류한다.

![spring-aop]({{site.url}}/assets/images/posts/spring-aop/spring-aop-01.png)

핵심 관심사는 해당 프로그램의 핵심적인 기능, 즉 비즈니스 로직을 뜻한다. 그에 빈해 횡단 관심사는 보안, 프로파일링, 로그, 트랜잭션은 비즈니스 기능은 아니지만, 요구 상황에 따라서 다수의 비즈니스 기능에 포함되며, 비즈니스 로직과는 다른 관심의 영역을 뜻한다. 여담으로 핵심 관심사를 모듈화 한것을 OOP, 부가적인 횡단 관심사를 모듈화 한것들 AOP라고 표현할 수 있을것 같다.

## 관련 용어
* Target: 어떤 대상에 부가 기능을 부여할 것인지
* Advice: 어떤 부가 기능을 부여할 것인지
* Join Point: 어디에 적용할 것인지
* Point Cut: 실제 Advice가 적용될 시점을 의미. Spring AOP에서는 advice가 적용될 메서드를 선정.

## @Transactional
트랜잭션 처리를 위해 아래와같은 트랜잭션 처리 동작 코드를 작성할 수 있다. 

{% highlight java %}
public class TransactionProxy {
    private final TransactonManager manager = TransactionManager.getInstance();
    ...

    public void transaction() {
        try {
            manager.begin();

            target.logic(); // 비즈니스 로직(target)

            manager.commit();
        } catch (Exception e) {
            manager.rollback();
        }
    }
}
{% endhighlight %}

하지만 모든 트랜잭션 처리 코드에 이와 같은 로직을 작성하게 되면, 비즈니스 로직에 집중이 되지 않고 중복적인 코드를 발생시키게 된다. 그래서 spring에서는 트랜잭션 처리 로직을 횡단으로 적용할 수 있도록 `@Transactional` AOP Proxy를 지원한다.

# AOP Proxy
## JDK Proxy
## CGLIB


# 그래서?

{% include ref.html %}
* 
* 