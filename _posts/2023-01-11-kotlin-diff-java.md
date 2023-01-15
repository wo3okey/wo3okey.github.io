---
layout: post
title: "[kotlin] 내가 kotlin을 선택한 이유"
categories: kotlin
tags: [kotlin, java]
---

![kotlin vs java]({{site.url}}/assets/images/kotlin-vs-java-01.png )

코틀린으로 프로젝트를 진행한지 어느덧 1년 정도가 되었다. 기본적인 문법이나 동작에 대해서는 개발함에 있어 큰 문제가 없을 정도로 익숙해졌다. 하지만 한번도 자바와 비교하여 코틀린의 장점을 깊이 생각해본 적은 없는 것 같다. 그래서 코틀린과 자바를 비교하며 코틀린이 자바에 비교하여 어떠한 장점을 가진 언어인지 파헤쳐보고자 한다. 참고로 두 언어의 기본적인 문법은 다루지 않는다.

## 1. Kotlin을 좋아하는 이유 

|항목|Java|Kotlin|
|------|---|---|------|
|Null Safe|X|O // 그저 감사하고 행복|
|Extention|X|O // 사기급 기능|
|Checked Exception|O|X // trade off지만 과감한 선택|
|Coroutines|X|O // 사기급 편리함|
|Smart Casts|X|O // 좋아|

### Null Safe
필요성에 대해서는 두말하면 잔소리다. 존재만으로도 너무 행복하다. 

java는 null에 취약하며, 언제 어느순간에 NullPointerException이 발생할지 예측할 수 없다. 그래서 늘 null과의 싸움을 하게된다. 떄로는 비즈니스 로직보다 null 체크 로직이 더 많을때도 있다. 물론 java8 이후에는 Optional을 활용하여 null safe하게 개발 할 수 있지만 불필요한 코드량과 가독성 측면에서는 여전히 아쉬운게 많다.

kotlin은 기본적으로 모든 변수가 null을 허용하지 않는다. non-null 변수에 null을 할당하려하면 컴파일 단계에서 실패한다. 필요에 따라 null이 필요할땐 `?` 키워드로 nullable을 표현할 수 있다. 또한 타입 추론을 지원하기 때문에 별도의 타입 정의 없이 nullable 변수를  컴파일 시점에서 체크할 수 있다. 추가적으로 ``` if null else ... ``` 을 한번에 표현할 수 있는 `let` 스코프 함수와 null이 아닐때를 표현하는 `?:` 엘비스 연산자(엘비스 프레슬리 헤어스타일을 닮아서...)를 함께 활용하면 더욱 간결하고 null safe하게 개발이 가능하다.

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

kotlin에서는 extention 확장함수를 지원한다. 어떤 클래스에 함수를 추가하는 기능이며, extention을 붙여놓은 객체에 `.` 찍으면 내가 만든 메소드를 사용할 수 있기에 마치 라이브러리를 만든 느낌을 받을 수 있다. json를 다루거나, 자주쓰이는 String 기능을 만들거나 등 불필요한 코드 또는 공통의 기능을 만들때 사용하면 좋다. 좋은 기능이나 자유도가 높기에 무차별하게 사용하면 욕먹기 딱 좋을 수 있다. 특정 클래스에서만 사용하거나 특정 컬렉션에서만 사용하는 등 개인적인 이유로 사용하기에는 일반 비즈니스 함수로 명확하게 개발하는것을 추천한다.

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

kotlin에서는 특정 Object 하위의 모든 클래스에는 extention이 적용가능하다. 예시로는 Int타입 List의 첫번째 element에 파라미터로 받은 num값을 더해서 반환하도록 작성했으며, 마치 Collection에서 지원하는 메소드인것 처럼 보여지고 있다. 코드를 읽는 입장에서 심신이 편-안하다.

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

kotlin에서의 extention 표현법은 알았고, 이를 java코드로 변환하면 위와 같다. extention은 결국 static 메소드를 생성한다. static 메소드는 GC의 대상이 되지 못하고 어플리케이션이 기동되는 동안에는 메모리에 남아있다. 결국 무분별하게 extention을 사용할 경우 결국 메모리 낭비를 초래할 것이다. 역시 뭐든 적절하게 필요에 따라 사용하는 것이 건강에 좋다.

### Checked Exception
왜지? 라고 의문을 들 수 있는 kotlin의 특징이 있다. checked exception을 지원하지 않는 것이다.

java에서는 Thread를 핸들링하거나 Databse, File, Stream 등 IO에 관련된 영역이나 그외 다방면으로 checked exception을 컴파일 단계에서 처리하도록 강제화한다. 하지만 kotlin에서는 이를 과감하게 포기했다.

`Kotlin`
{% highlight kotlin %}

Thread.sleep(1000)

{% endhighlight %}

`Java`
{% highlight java %}

try {
    Thread.sleep(1000);
} catch (InterruptedException e) {
    throw new RuntimeException(e);
}

{% endhighlight %}

java에서는 `Thread.sleep(1000);` 까지만 입력하면 sleep에 redline과 함께 컴파일 에러가 발생한다. 하지만 kotlin에서는 묻지도 따지지도 않고 실행된다. 이토록 처리해둔 이유는 아래와 같다.
* checked exception은 비즈니스 로직을 매우 지저분하게 만든다.
* exception을 강제화 해도 대부분의 개발자는 유의미한 exeption 핸들링을 하지않는다.

이러한 이유로 kotlin에서는 과감하게 checked exception을 버린것이다. 어찌보면 발생할 수 있는 exception 위험에 열려있는것 같아 보이지만, 무의미하게 throw 처리를 해놓을바엔 한대 뚜들겨 맞고 필요에따라 적절히 `try catch` 문으로 유의미하게 처리하는 것도 좋아보인다. 다만 java를 사용해보지 않고 kotlin로 입문한 개발자는 어떠한 구문에서 checked exception이 발생될 수 있는지 조차 모를 수 있겠다. 같이 협업하면 조금 난감할지도...

### Coroutines
corutine은 kotlin에만 등장하는 개념이며 java에서는 Future, CompletableFuture로 표현할 수 있다.


## 2. Kotlin 성능 비교
kotlin과 java의 성능을 비교해보고자 한다. 벤치마크 스펙은 아래와 같다.

* pc: MacBook Pro (13-inch, 2020)
* os: macOs Monterey
* cpu: 2 GHz 쿼드 코어 Intel Core i5
* memory: 16GB 3733 MHz LPDDR4X

### 일반적인 성능

### 비동기처리 성능

### Stream 성능

## 3. Kotlin 선택의 최종 이유
### 일급함수

본 포스트에서 언급한 이유들 외에도 개인적으로 kotlin을 좋아하는 이유는 정말 많다. 학부생 시절을 포함하면 거의 10년을 java로만 개발해왔지만, 이제는 kotlin 없이는 개발하기 싫을정도로 너무 편리하고 덕분에 생산성과 가독성이 상당히 높아졌다는 생각을 한다.


`ref.`
* https://www.makeuseof.com/kotlin-vs-java/#code-volume-amp-speed-of-coding