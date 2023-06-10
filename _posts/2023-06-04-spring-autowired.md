---
layout: post
title: 그래서 spring @Autowired란 무엇인가?
categories: [spring]
tags: [spring, autowired, DI, IoC]
---

spring에서는 Bean을 IoC(Inversion of Control) 컨테이너에 의해 관리된다. spring에서 객체는 IoC 컨테이너로부터 의존성을 주입 받을 수 있으며, 다양한 방법으로 의존성을 주입받을 수 있다. 그 중 @Autowired에 대해 알아본다.

# IoC 컨테이너
Ioc는 spring에만 있는 개념은 아니며, 의존 관계를 역전하여 의존성을 주입하는 방식을 뜻한다. 이때 spring에서 IoC를 제공하는 객체가 Ioc 컨테이너이다.

## DI
IoC 컨테이너는 bean의 생명주기를 관리하며, 객체의 생명주기 및 의존성을 관리한다. 의존성은 말 그대로 특정 객체에서 다른 객체 정보를 필요로 하여 서로 의존 관계에 있는 것들 뜻한다. 이를 spring에서는 IoC 컨테이너에 의해 DI(Dependency Injection) 의존성 주입을 받게 된다. spring에서는 그저 DI를 원하는 객체에 `@Autowired`와 같은 표기를 통해서 Ioc 컨테이너에게 의존성 주입 대상임을 알려주기만 하면 된다.
{% highlight java %}
public class StudentService {
    @Autowired
    private StudentRepository studentRepository
    ...

    public void register() {
        ...
    }
}
{% endhighlight %}

# Autowired
@Autowired는 spring에서 가장 일반적으로 사용되는 어노테이션으로, 의존성 주입을 위해 사용된다. @Autowired는 타입(Type)을 기반으로 자동으로 의존성을 주입한다. spring은 클래스의 생성자, 필드, 메서드 등에서 @Autowired이 적용된 대상을 찾고, 해당하는 의존성을 자동으로 주입한다.

## Injections
@Autowired로 의존성을 주입 받는 방법은 생성자, setter, 필드 주입 방식이 있다.

`Constructor Injection`
{% highlight java %}
public  class  ExampleCase {
    private final ChocolateService chocolateService;
    private final DrinkService drinkService;
    
    @Autowired
    public ExampleCase(ChocolateService  chocolateService, DrinkService  drinkService) {
   	    this.chocolateService = chocolateService;
   	    this.drinkService = drinkService;
    }
}
{% endhighlight %}


`Setter Injection`
{% highlight java %}
public  class  ExampleCase{
    private ChocolateService chocolateService;
    private DrinkService drinkService;

    @Autowired
    public void setChocolateService(ChocolateService chocolateService){
        this.chocolateService = chocolateService;
    }

    @Autowired
    public void setDrinkService(DrinkService  drinkService){
        this.drinkService = drinkService;
    }
}
{% endhighlight %}


`Field Injection`
{% highlight java %}
public  class  ExampleCase{
    @Autowired
    private ChocolateService  chocolateService;

    @Autowired
    private DrinkService  drinkService;
}
{% endhighlight %}

일반적으로 `Constructor Injection`의 생성자 주입 방식을 선택한다. 이유는 아래와 같다.

1. SRP(single responsibility principle)
필드 주입 방식은 의존성 주입 방식이 매우 간단하다는 장점이 있다. 이는 양날의 검으로 너무 쉽게 의존성을 주입하다보니, 막무가내로 하나의 객체에 책임을 부여하도록 개발할 수 있는 것이다. 즉, 객체지향 원칙중 단일 책임의 원칙에 어긋나게 개발할 위험에 노출시킬 수 있다. 생성자에 의존관계를 기입하면 한눈에 파악하기 좋을 뿐더러, 너무 많아진 파라미터는 심리상 뭔가 잘못됨(?)을 느낄 수 있게 해준다.

2. DI 컨테이너의 결합성과 테스트 용이성
Field Injection을 사용하면 필요한 의존성을 가진 클래스를 곧바로 인스턴스화 시킬 수 없다. 이유는 final 선언시 필드값을 초기화 해야하기 때문이다. 객체는 생성과 동시에 생성에 필요한 파라미터와 함께 초기화 하여 메모리에 올라가게 되는데, 필드를 초기화 할 수 없는 객체가 되어 단위테스트와 같은 mocking 객체를 만들 수 없다. 또한 final 키워드가 붙지 않기 때문에 객체 생성 이후 불변성을 보장할 수 없게된다.

3. 순환 의존성
Constructor Injection에서 순환 의존성(A객체 <-> B객체 서로간에 참조)을 가질 경우 순환참조 관련된 오류가 발생되어, 순환 의존성을 알 수 있다. 하지만 Field Injection의 경우 해당 객체가 생성되는 시점에 알 수 있기 때문에 런타임간에 오류가 발생될 수 있다.

## Inject, Resource, Quirifire, Primay
@Autowired는 타입(Type)을 기반으로 자동으로 의존성을 주입한다. Spring은 클래스의 생성자, 필드, 메서드 등에서 @Autowired 적용된 대상을 찾고, 해당하는 의존성을 자동으로 주입한다. 그 외 @Inject, @Resource, @Quirifire, @Primay 방법도 간단하게 알아보자.

* `Inject`<br>
@Inject는 JSR-330(Java Dependency Injection specification)에 정의된 어노테이션이다. 즉, spring이 아닌 java 스펙에서 제공하는 DI 어노테이션인 것이다. @Autowired와 기능적으로 동일하다. Spring에서도 @Inject을 지원하며, @Autowired와 동일하게 타입(Type)을 기반으로 자동 의존성 주입을 수행한다.

* `Resource`<br>
@Resource은 @Inject와 마찬가지로 java 스펙에서 제공하는 JSR-250(Java Common Annotations specification)에 정의된 어노테이션으로, 이름(Name)을 기반으로 의존성 주입을 수행한다. @Resource은 주입할 빈을 지정할 때 이름을 사용하며, 기본적으로 필드 이름과 동일한 이름의 빈을 찾아서 주입한다. 또한 name 속성을 사용하여 다른 이름의 빈을 명시적으로 지정할 수도 있다.

* `@Qualifier`<br>
@Qualifier은 같은 타입(Type)의 여러 빈 중에서 특정 빈을 선택할 때 사용한다. spring에서 같은 타입의 여러 빈을 찾았을 때, 어떤 빈을 주입해야 할지 명확하게 판단할 수 없는 경우 @Qualifier을 함께 사용하면 된다. @Qualifier은 주입할 빈을 식별하는 데 사용되는 추가적인 정보를 제공한다. 예를 들어, @Qualifier("myBean")과 같이 사용하여 myBean이라는 특정 빈을 주입할 수 있다.

* `@Primary`<br>
@Primary은 같은 타입(Type)의 여러 빈 중에서 기본 빈(Default Bean)을 지정할 때 사용한다. Spring은 @Primary이 적용된 빈을 우선적으로 선택하여 주입하며, 여러 개의 빈이 있을 때 기본 빈을 명시적으로 지정하고자 할 때 사용한다.


# 그래서?
실무에서 개발을 하더라도 bean의 타입과 이름을 겹칠 정도의 개발을 할 일은 거의 없다고 봐도 무관하다. 새로운 공통 컴포넌트나 프레임워크를 만들지 않는 이상 말이다.
> Spring에서 앵간하면 이상한 주입 방식 쓰지말고, 장점 많은 Constructor(생성자) Injection 방식의 @Autowired로 의존성을 주입받자.


{% include ref.html %}
* https://velog.io/@aidenshin/%ED%95%84%EB%93%9C%EC%A3%BC%EC%9E%85-%EC%83%9D%EC%84%B1%EC%9E%90-%EC%A3%BC%EC%9E%85%EB%B0%A9%EC%8B%9D%EC%9C%BC%EB%A1%9C-%EB%B3%80%EA%B2%BD
* 