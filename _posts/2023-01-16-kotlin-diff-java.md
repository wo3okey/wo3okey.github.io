---
layout: post
title: 내가 kotlin을 선택한 이유
categories: kotlin
tags: [kotlin, java]
---

![kotlin vs java]({{site.url}}/assets/images/kotlin-vs-java-01.png )

kotlin으로 프로젝트를 진행한지 어느덧 1년 정도가 되었다. 하지만 한번도 java와 비교하여 kotlin의 장점을 깊이 생각해본 적은 없는 것 같다. java와 비교하여 현재 kotlin을 선택한 이유를 얘기해보고자 한다. 참고로 언어의 기본적인 문법은 다루지 않는다.

## 1. Kotlin 선택의 이유

### Null Safe
필요성에 대해서는 두말하면 잔소리다. null에 대한 지원만으로도 너무 행복하다. 

java는 null에 취약하며, 언제 어느순간에 NullPointerException이 발생할지 예측할 수 없다. 그래서 늘 null과의 싸움을 하게된다. 떄로는 비즈니스 로직보다 null 체크 로직이 더 많을때도 있다. 물론 java8 이후에는 Optional을 활용하여 null safe하게 개발 할 수 있지만 불필요한 코드량과 가독성 측면에서는 여전히 아쉬운게 많다.

kotlin은 기본적으로 모든 변수가 null을 허용하지 않는다. non-null 변수에 null을 할당하려하면 컴파일 단계에서 실패한다. 필요에 따라 null이 필요할땐 `?` 키워드로 nullable을 표현할 수 있다. 또한 타입 추론을 지원하기 때문에 별도의 타입 정의 없이 nullable 변수를  컴파일 시점에서 체크할 수 있다. 추가적으로 ``` if null else ... ``` 을 한번에 표현할 수 있는 `let` 스코프 함수와 null이 아닐때를 표현하는 `?:` 엘비스 연산자(엘비스 프레슬리 헤어스타일을 닮아서...)를 함께 활용하면 더욱 간결하고 null safe하게 개발이 가능하다.

`Kotlin`
{% highlight kotlin %}

fun nonNullReturnFunction(): Int {
    val num = 1
    val numOfNullable = nullReturnFunction()

    return numOfNullable?.let { it + num } ?: 0
}

fun nullReturnFunction(): Int? {
    val list = listOf(1, 2, 3, 4)
    return list.firstOrNull { it < 1 }
}

{% endhighlight %}

### Extention
공통적으로 사용되는 범용성 코드를 잘 만들었을때 뽕맛은 개발자라면 공감할 듯하다.

kotlin에서는 extention 확장함수를 지원한다. 어떤 클래스에 함수를 추가하는 기능이며, extention을 붙여놓은 객체에서 내가 만든 메소드를 사용할 수 있기에 마치 라이브러리를 만든 느낌을 받을 수 있다. json를 다루거나, 자주쓰이는 String 기능을 만들거나 등 불필요한 코드 또는 공통의 기능을 만들때 사용하면 좋다. 좋은 기능이나 자유도가 높기에 무차별하게 사용하면 욕먹기 딱 좋을 수 있다. 특정 클래스에서만 사용하거나 특정 컬렉션에서만 사용하는 등 개인적인 이유로 사용하기에는 일반 비즈니스 함수로 명확하게 개발하는것을 추천한다.

`Java`
{% highlight java %}

public static int firstPlusNum(Collection<Integer> collection, int num) {
    return collection.stream().findFirst().get() + num;
}

public static void main(String[] args) {
    List<Integer> list = Arrays.asList(1, 2, 3, 4);
    int result = firstPlusNum(list, 100);
    System.out.println(result); // 101
}

{% endhighlight %}

아래 kotlin 코드를 java 코드로 변환하면 위와 같다. extention은 결국 static 메소드를 생성한다. static 메소드는 GC의 대상이 되지 못하고 어플리케이션이 기동되는 동안에는 메모리에 남아있다. 결국 무분별하게 extention을 사용할 경우 결국 메모리 낭비를 초래할 것이다. 역시 뭐든 적절하게 필요에 따라 사용하는 것이 건강에 좋다.

`Kotlin`
{% highlight kotlin %}

fun List<Int>.firstPlusNum(num: Int): Int {
    return this.first() + num
}

fun main() {
    val list = listOf(1, 2, 3, 4)
    val result = list.firstPlusNum(100)
    println(result) // 101
}

{% endhighlight %}

kotlin에서는 특정 Object 하위의 모든 클래스에는 extention이 적용가능하다. 예시로는 Int타입 List의 첫번째 element에 파라미터로 받은 num값을 더해서 반환하도록 작성했으며, 마치 Collection에서 지원하는 메소드인것 처럼 보여지고 있다. 코드를 읽는 입장에서 심신이 편-안하다. 물론 실무에서 특정 컬렉션 구현체에 제너럴하지 못한 타입으로 extention을 만들어 사용하는 일은 드물다. 예제 코드 정도로만 생각했으면 좋겠다.


### Checked Exception
왜지? 라고 의문을 들 수 있는 kotlin의 특징이 있다. checked exception을 지원하지 않는 것이다. 하지만 이유가 납득된다면 이를 kotlin을 선택한 이유로 꼽을 수 있다. 

`Java`
{% highlight java %}

try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    throw new RuntimeException(e);
}

{% endhighlight %}

`Kotlin`
{% highlight kotlin %}

Thread.sleep(1000)

{% endhighlight %}

kotlin에서는 Thread에 대한 처리에 대해 묻지도 따지지도 않고 실행 가능하다.


java에서는 `Thread.sleep(1000);` 까지만 입력하면 sleep에 redline과 함께 컴파일 에러가 발생한다. java는 Thread를 핸들링하거나 Databse, File, Stream 등 IO에 관련된 영역이나 그외 다방면으로 checked exception을 컴파일 단계에서 처리하도록 강제화한다. 하지만 코틀린은 이를 과감하게 포기했다. 이유는 아래와 같다.
* checked exception은 비즈니스 로직을 매우 지저분하게 만든다.
* exception을 강제화 해도 대부분의 개발자는 유의미한 exeption 핸들링을 하지않는다.

어찌보면 발생할 수 있는 exception 위험에 열려있는것 같아 보인다. 무의미하게 throw 처리를 해놓을바엔 필요에따라 적절한 exception 처리를 유도한 것이다. 불필요한 checked exeption을 제거함으로써 얻은 가독성 효과는 확실하다. 다만 java를 사용해보지 않고 kotlin로 입문한 개발자는 어떠한 구문에서 checked exception이 발생될 수 있는지 조차 모를 수 있겠다. 같이 협업하면 조금 난감할지도...

### Coroutines
비동기 처리가 제일 쉬웠어요. (위험할 소리..😇)

corutine은 비동기 처리를 굉장히 쉽게 처리 할 수 있도록 지원하는 kotlin 라이브러리다. java에서는 Thread, Callable, Runnable, CompletableFuture 등 다양하게 비동기 개발 방법이 있지만, 예시에서는 CompletableFuture과 비교한다. 비동기 처리시 함께 고민해야할 자원 관리, 동시성 제어, 트랜잭션 처리 등의 내용은 다루지 않는다. just corutine의 편리함만 다룬다.


`Java`

{% highlight java %}

class Pizza {
    final String name;
    final int minute;

    public Pizza(String name, int minute) {
        this.name = name;
        this.minute = minute;
    }

    public Pizza makePizza() {
        try {
            Thread.sleep(this.minute); // make pizza time
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
        System.out.println(this.name + "를 " + this.minute + "분만에 완성했습니다.");

        return this;
    }
}

public class AsyncTest {
    public static void makePizzaAsync(List<Pizza> pizzas) {
        List<CompletableFuture<Pizza>> futures = pizzas.stream()
                .map(pizza -> CompletableFuture.supplyAsync(() -> pizza.makePizza()))
                .collect(Collectors.toList());

        futures.stream()
                .map(CompletableFuture::join)
                .collect(Collectors.toList());
    }

    public static void main(String[] args) {
        final List<Pizza> pizzas = Arrays.asList(
                new Pizza("페퍼로니 피자", 10),
                new Pizza("불고기 피자", 40),
                new Pizza("하와이언 피자", 30),
                new Pizza("콰트로치즈 피자", 20)
        );

        makePizzaAsync(pizzas);
    }
}

{% endhighlight %}


`Kotlin`
{% highlight kotlin %}

fun main() {
    val pizzas = listOf(
        Pizza("페퍼로니 피자", 10),
        Pizza("불고기 피자", 40),
        Pizza("하와이언 피자", 30),
        Pizza("콰트로치즈 피자", 20)
    )

    makePizzaAsync(pizzas)
}

fun makePizzaAsync(pizzas: List<Pizza>) {
    runBlocking(Dispatchers.IO) {
        val defer = pizzas.map {async { it.makePizza() }}
        defer.awaitAll()
    }
}

data class Pizza(
    val name: String,
    val minute: Int
) {
    fun makePizza(): Pizza {
        Thread.sleep(this.minute.toLong())
        println(this.name + "를 " + this.minute + "분만에 완성했습니다.");
        return this
    }
}

{% endhighlight %}

```
페퍼로니 피자를 10분만에 완성했습니다.
콰트로치즈 피자를 20분만에 완성했습니다.
하와이언 피자를 30분만에 완성했습니다.
불고기 피자를 40분만에 완성했습니다.
```

두 언어 모두 결과는 위와같이 로직상 Thread sleep time 순으로 동일하게 프린트된다. main 메소드나 Pizza 객체부를 제외하고 `makePizzaAsync` 메소드를 보면 비동기 처리를 얼마나 간편하게 처리할 수 있는지 알 수 있다.

### Smart Casts

이젠 스마트폰 없이 안되는 세상, 캐스팅도 스마트 캐스팅 시대. 코드의 `ability` 메소드를 보면 바로 직감할 것이다.

`Java`
{% highlight java %}

interface Animal { String getName(); }

class Dog implements Animal {
    private final String name;

    Dog(String name) {this.name = name;}

    @Override
    public String getName() {return this.name;}
    public String running() {return "달려요.";}
}

class Bird implements Animal {
    private final String name;

    Bird(String name) {this.name = name;}

    @Override
    public String getName() {return this.name;}
    public String flying() {return "날아요.";}
}

public class SmartCastTest {
    public static void main(String[] args) {
        Dog dog = new Dog("개");
        Bird bird = new Bird("새");

        List<Animal> animals = Arrays.asList(dog, bird);
        animals.forEach(animal -> {
            System.out.println(animal.getName() + "는 " + ability(animal));
        });
    }

    public static String ability(Animal animal) {
        if (animal instanceof Dog) {
            Dog dog = (Dog)animal;
            return dog.running();
        }

        if (animal instanceof Bird) {
            Bird dog = (Bird)animal;
            return dog.flying();
        }

        throw new RuntimeException("동물이 아닙니다.");
    }
}

{% endhighlight %}

`Kotlin`
{% highlight kotlin %}

interface Animal { val name: String }
class Dog(override val name: String) : Animal { fun running(): String = "달려요." }
class Bird(override val name: String) : Animal { fun flying(): String = "날아요." }

fun ability(animal: Animal): String {
    return when (animal) {
        is Dog -> animal.running()
        is Bird -> animal.flying()
        else -> throw RuntimeException("동물을 입력하세요.")
    }
}

fun main() {
    val dog = Dog("개")
    val bird = Bird("새")

    val animals = listOf(dog, bird)
    animals.forEach {
        println("${it.name}는 ${ability(it)}")
    }
}

{% endhighlight %}

java는 `instanceof` 이후에도 직접 Animal의 구현체로 down casting 해줘야한다. 하지만 kotlin은 `is` 키워드와 함께 캐스팅된 결과를 받고, 이를 바로 사용할 수 있다. 타입 체크와 변환까지 한번에 지원되는 기능이며 `is` 라는 키워드 자체가 가독성 측면에서도 너무 직관적이고 명확하다. 코드길이 차이는 두말할 것 없다.

### First Class

함수형 프로그래밍을 공부하거나 한번쯤 사용해봤다면 `일급객체` `일급함수` 라는 단어를 봤을 것이다. 아니라면 지금부터 알면된다.
일급시민(First-class citizen)이 될 수 있는 객체를 일급객체(First-class object), 함수를 일급함수(First-class funcation)으로 지칭할 수 있다.
일급시민이란 아래 요소를 모두 만족하는 대상을 뜻한다. 아래에는 함수를 예제로 작성한다.

>1. 변수에 할당할 수 있다.
2. 객체의 인자로 넘길 수 있다.
3. 객체의 반환값으로 반환할 수 있다.

`Kotlin`
{% highlight kotlin %}

fun main() {
    // 1. 변수에 할당할 수 있다.
    val sum = { x: Int, y: Int -> x + y }
    println(sum(1, 2))

    val sum2 = sumFun1(sum)
    println(sum2)

    val sum3 = sumFun2(sum2)
    println(sum3.invoke())
}

// 2. 객체의 인자로 넘길 수 있다.
fun sumFun1(firstClass: (x: Int, y: Int) -> Int): Int {
    return firstClass.invoke(3, 4)
}

// 3. 객체의 반환값으로 반환할 수 있다.
fun sumFun2(sum: Int): () -> Int {
    return { sum + 5 }
}

{% endhighlight %}

그렇다면 java는 어떨까? kotlin에서는 기본적으로 지원하는 lambda가 java에서는 java8부터 지원하며, 이를 이용하면 비슷한 느낌으로는 만들 수 있다.

`Java`
{% highlight java %}

@FunctionalInterface
interface Lambda {
    int sum(int a, int b);
}

@FunctionalInterface
interface Lambda2 {
    int justReturn();
}

public class FirstClassJava {
    public static void main(String[] args) {

        Lambda sum1 = (int a, int b) -> a + b;
        System.out.println(sum1.sum(1, 2));

        int sum2 = sumFun1(sum1);
        System.out.println(sum2);

        Lambda2 sum3 = sumFun2(sum2);
        System.out.println(sum3.justReturn());
    }

    public static int sumFun1(Lambda lambdaSum) {
        return lambdaSum.sum(3, 4);
    }

    public static Lambda2 sumFun2(int sum) {
        Lambda2 justReturn = () -> sum + 5;
        return justReturn;
    }
}

{% endhighlight %}

```
3
7
12
```

두 언어 모두 같은 결과를 얻을 수 있다. 하지만 java의 경우 일급시민의 조건을 만족하지 않는다. 하나하나 짚어보자면, `sum1`은 마치 변수에 함수를 할당한 것 처럼 보이지만 이는 java에서 lambda를 할당하기 위해서 interface를 만들도록 강제화 되어있다. `sumFun1`의 파라미터로 `Lambda` 객체를 넘겨야하며, 함수(lambda)를 넘길 순 없다. `sumFun2`의 반환형으로 함수(lambda)를 반환할 수 없으며 `Lambda2`를 반환 해야한다. 즉, java에서 lambda를 이용하여 kotlin과 구조적으로 비슷한 형태로 개발할 수 있지만 java는 일급시민(함수, 객체)이 될 수 없는 언어라는 점이다.

금융회사 재직당시 특정 프로젝트에서 java 1.6을 사용하는 곳을 보았지만, 이런 경우는 어쩔 수 없이 사용할 수 없을것이다. 하지만 java8 이상을 사용하면서도 lambda, stream 등의 함수형 프로그래밍 지식을 습득하지 못하여 거부감을 느껴하는 케이스도 보았다. kotlin에서는 언어 레벨에서 간결하고 쉽게 사용할 수 있도록 지원되어 가장 큰 장점이라 생각되었다.

### Immutable

언어 레벨에서 지원하는 불변성의 효과는 실로 기가막히다.

`Java`
{% highlight java %}
class Person {
    private String name;
    private int age;

    public Person(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getName() {
        return name;
    }
}

public class FirstClassJava {
    public static void main(String[] args) {
        Person person = new Person("wookey", 30);
        System.out.println(changeName1(person).getName());
    }

    public static Person changeName1(Person person) {
        person.setName(person.getName() + "변경1");
        return changeName2(person);
    }

    public static Person changeName2(Person person) {
        person.setName(person.getName() + "변경2");
        return changeName3(person);
    }

    public static Person changeName3(Person person) {
        person.setName(person.getName() + "변경3");
        return person;
    }
}

{% endhighlight %}

실무에서 이렇게 극단적인 코드는 없을것 같다고 생각될 것이다. 하지만 실제로 실무 코드에도 수개의 메소드의 파라미터에 레퍼런스를 받고 이를 변경하고 최종적으로 결과를 받는 형태의 코드는 상당히 많다. 그것도 예제 코드처럼 단순한 변경이 아닌, 수많은 비즈니스 로직과 얽히고 엮여 레퍼런스의 값을 계속 변경, 조작하는 코드들 말이다.

`Kotlin`
{% highlight kotlin %}
data class Person(val name: String, val age: Int)

fun main() {
    val person = Person("wookey", 30)
    println(changeName1(person).name)
}

fun changeName1(person: Person): Person {
    person.name = person.name + "변경1"
    return changeName2(person)
}

fun changeName2(person: Person): Person {
    person.name = person.name + "변경2"
    return changeName3(person)
}

fun changeName3(person: Person): Person {
    person.name = person.name + "변경3"
    return person
}
{% endhighlight %}

예제 java코드를 kotlin으로 옮기면 위와 같다. 정확히 말하자면 느낌만 옮긴것이다. 위 kotlin 코드는 컴파일되지 않는다. 언어레벨에서 main문에 선언한 `Person` 객체에 선언한 `name` 필드를 `val` 이라는 키워드로 변경불가능한 immutable(final) 변수로 만들었기 때문이다. 어찌보면 `name` 필드의 키워드를 `val`에서 `var`로 변경만 하면 컴파일될 뿐 아니라 java 코드와 동일한 결과를 가져 올 수 있다. 하지만 여기서 얻을 수 있는 인사이트가 있다. 왜 kotlin은 기본적으로 언어 레벨에서 final 키워드를 채택한 것일까? 만약 kotlin 코드에서 `var` 키워드로 바꾸지않고 동일한 결과를 가져 오려면 각 메소드마다 아래와 같이 작성해야할 것이다.

{% highlight kotlin %}
fun changeName1(person: Person): Person {
    val newPerson = Person(person.name + "변경1", person.age)
    return newPerson
}
{% endhighlight %}

이는 사실상 결과만 같지 다른 코드이다. 레퍼런스의 값을 바꾸는게 아닌, 새로운 객체를 계속 만들어내는 방법이기 때문이다. 결과적으로 레퍼런스를 조작하는 코드는 결코 좋지못한 코드를 양산해낼 가능성이 높다. 즉, kotlin은 레퍼런스의 변경을 최대한 막고 하나의 메소드는 자신의 역할만 충실히 하도록 개발할 수 있도록 언어레벨에서 지원하는 것이다. 이는 결국 OOP원칙 중 SRP(Single Responsibility Principle)에 굉장히 충실할 수 있다고 생각한다. 결과론적으로 immutable의 지원은 유지보수 좋은 코드, 생산성 있는 코드를 만들 수 있다고 생각한다.


## 2. Kotlin은 단점이 없는가

결론부터 말하면 아니다. 모든 언어에는 특징이 있으며, 자신의 입맛도 중요하지만 시장의 수요도 중요하다. 기술적으로만 보았을때 kotlin은 java 기반으로 만들어졌고 동일하게 JVM 아래에서 돌아간다. java의 불편한 단점들을 보완하기 위해 태어난 언어이므로 기술적인 부분보다는 그 외적 이유로 단점을 꼽아 볼 수 있다.

* 새로운 언어이므로 개발자 커뮤니티가 작을 수 있다. 
* 대한민국은 java 공화국이라는 별명이 있을정도로 java에 대한 애착심이 높다. 그렇다면 취업의 입장에서 수요 차이는 분명할 수 있다.
* kotlin은 java와 100% 호환되는 언어라고 제작사에서 언급한다. `SomeClass::class.java` 와 같은 리플렉션 문법은 `KClass` 라는 kotlin 클래스로 랩핑된다. 이와 같은 사용은 모호한 java 문법과의 호환이 있어 보인다.
* 시간이 해결해줄 것 같지만, 아직까지 완벽 지원되지 않는 라이브러리나 개발툴(IDE)가 있다. 특히 kotlin은 intellij IDE 개발사인 jetbranin에서 만든 언어이므로 다양한 IDE가 나오기 전까진 특정 IDE에 특화된 언어라는 제약이 있다.
* 함수형 언어라는 패러다임 자체를 긍정적으로 이해하지 못하는 집단들에게 선택받지 못할 수 있다.
* 개발 공부 자체를 처음 배우는 입문자에게 추천하기 어려울 것 같다. 아무래도 kotlin의 기본 base는 JVM java이며, 특징을 모른채 개발하게 된다면 기본적이고 중요한 지식을 많이 놓칠 수 있다고 생각된다.


## 3. 결론
기존 java의 사용자라면 충분히 메리트를 느끼고 kotlin에게 매력을 느낄 수 있을 것이다. 나도 학부생 시절 포함하여 개발자 커리어 전부를 java로 개발했다. 하지만 익숙하면서도 많은 변화를 가져다준 kotlin의 릴리즈를 지켜보며 과감하게 주 언어를 변경할 수 있었다. 포스트에 언급하지 않은 kotlin의 장점과 단점은 더 많이 있겠지만 kotlin을 선택한 이유를 요약하면 아래와 같이 정리 할 수 있을것 같다.

* Concise (간결성)
* Safe (안정성)
* Interoperable (상호운용가능성)



<hr>
`ref.`

* <https://www.makeuseof.com/kotlin-vs-java/#code-volume-amp-speed-of-coding>
* <https://kotlinlang.org/docs/comparison-to-java.html#what-kotlin-has-that-java-does-not>