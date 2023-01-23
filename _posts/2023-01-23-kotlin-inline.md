---
layout: post
title: 그래서 kotlin inline은 무엇인가?
categories: kotlin
tags: [kotlin, java, inline]
---

![kotlin inline]({{site.url}}/assets/images/kotlin-inline-01.png )

## 1. inline
kotlin 함수에 붙는 `inline` 키워드는 말 그대로 호출되는 특정 코드 line 사이에 특정 inline 키워드가 붙은 함수의 코드를 넣을(in) 수 있도록 지원하는 키워드이다.
`inline` 키워드는 객체(클래스)와 함수 레벨에서 사용할 수 있다. 예시는 객체 레벨이 아닌 함수 레벨에서 설명한다.

### inline 함수

간단하게 아래 코드로 살펴보자. 아주 inline이 붙은 코드를 호출한 위치에 넣어주는 단순 역할 그 잡채이다.


`Kotlin`
{% highlight kotlin %}

fun main(args: Array<String>) {
    println("print1")
    print2and3()
    println("print4")
    print5 { "print5" }
}

fun print2and3() {
    println("print2")
    println("print3")
}

fun print5(lambda: () -> String) {
    println(lambda.invoke())
}

{% endhighlight %}

`Java`
{% highlight java %}
public static final void main() {
    String var0 = "print1";
    System.out.println(var0);
    print2and3();
    var0 = "print4";
    System.out.println(var0);
    int $i$f$print5 = false;
    String var1 = "print5";
    System.out.println(var1);
}

public static void main(String[] var0) {
    main();
}

public static final void print2and3() {
    String var0 = "print2";
    System.out.println(var0);
    var0 = "print3";
    System.out.println(var0);
}

public static final void print5() {
    int $i$f$print5 = 0;
    String var1 = "print5";
    System.out.println(var1);
}

{% endhighlight %}

```
print1
print2
print3
print4
print5
```

kotlin 코드에는 2개의 print는 직접 처리하며, 2개의 print는 함수를 호출해서 출력되는 것으로 보인다. 하지만 이를 java 코드로 변환한 결과를 보면 다소 충격적이다. 실제로 `print5` 함수는 선언만 되어 있을뿐 호출하지 않는다. 내가 만든 함수가 호출되지 않다니! 하지만 해당 함수에 있는 모든 코드가 main문 안에 그대로 들어가 있는 모습이 보인다. 이것이 `inline` 키워드의 역할이다. 그렇다면 이렇게 본래 함수와 호출부에 들어간 코드까지, 코드량을 2배로 만들게 하는 `inline` 키워드가 왜 필요할까?

### inline 고차함수

위 예제의 `print5` 함수의 파라미터만 lambda 고차함수로 바꿔보자.

`Kotlin`
{% highlight kotlin %}

fun main(args: ArrayList<String>) {
    println("print1")
    print2and3()
    println("print4")
    print5 { "print5" }
}

fun print2and3() {
    println("print2")
    println("print3")
}

fun print5(lambda: () -> String) {
    println(lambda.invoke())
}

{% endhighlight %}

`Java`
{% highlight java %}
public static final void main(@NotNull ArrayList args) {
    Intrinsics.checkNotNullParameter(args, "args");
    String var1 = "print1";
    System.out.println(var1);
    print2and3();
    var1 = "print4";
    System.out.println(var1);
    print5((Function0)null.INSTANCE);
}

public static final void print2and3() {
    String var0 = "print2";
    System.out.println(var0);
    var0 = "print3";
    System.out.println(var0);
}

public static final void print5(@NotNull Function0 lambda) {
    Intrinsics.checkNotNullParameter(lambda, "lambda");
    Object var1 = lambda.invoke();
    System.out.println(var1);
}

{% endhighlight %}

java의 main 함수에서 `print5` 함수를 호출하는 부분을 보면 `(Function0)null.INSTANCE` 부분이 있다. 새로운 Function 익명 클래스 객체를 생성하는 것이다. 즉, 불필요한 객체생성 및 메모리 낭비를 초래할 수 있다는 것이다. `print5` 함수에 inline 키워드만 붙인 java 코드는 아래와 같다.

`Kotlin`
{% highlight kotlin %}
inline fun print5(lambda: () -> String) {
    println(lambda.invoke())
}
{% endhighlight %}


`Java`
{% highlight java %}

public static final void main(@NotNull ArrayList args) {
    Intrinsics.checkNotNullParameter(args, "args");
    String var1 = "print1";
    System.out.println(var1);
    print2and3();
    var1 = "print4";
    System.out.println(var1);
    int $i$f$print5 = false;
    int var2 = false;
    String var4 = "print5";
    System.out.println(var4);
}

public static final void print2and3() {
    String var0 = "print2";
    System.out.println(var0);
    var0 = "print3";
    System.out.println(var0);
}

public static final void print5(@NotNull Function0 lambda) {
    int $i$f$print5 = 0;
    Intrinsics.checkNotNullParameter(lambda, "lambda");
    Object var2 = lambda.invoke();
    System.out.println(var2);
}
{% endhighlight %}

별도의 익명클래스 생성 없이 `print5` 함수 내 로직들이 main안으로 inline 되어있는 모습이다. 이로써 경우 inline 키워드가 주는 장점을 확인 할 수 있었다.

## 2. noinline
`noinline` 키워드는 `inline` 키워드가 붙어 있는 객체 또는 함수 내에 파라미터 레벨에서 특정 파라미터에 `inline`을 적용하지 않고 싶을때 사용한다. 고차함수를 파라미터로 2개 받는 `print5and6` 함수를 만들었고, 이때 첫번째 파라미터에는 `noinline` 키워드를 붙이고 java 코드로 변경해보았다.

`Kotlin`
{% highlight kotlin %}
fun main(args: Array<String>) {
    println("print1")
    print2and3()
    println("print4")
    print5and6({ "print5" }, { "print6" })
}

fun print2and3() {
    println("print2")
    println("print3")
}

inline fun print5and6(noinline lambda: () -> String, lambda2: () -> String) {
    println(lambda.invoke())
    println(lambda2.invoke())
}
{% endhighlight %}


`Java`
{% highlight java %}

public static final void main(@NotNull String[] args) {
    Intrinsics.checkNotNullParameter(args, "args");
    String var1 = "print1";
    System.out.println(var1);
    print2and3();
    var1 = "print4";
    System.out.println(var1);
    Function0 lambda$iv = (Function0)null.INSTANCE;
    int $i$f$print5and6 = false;
    Object var3 = lambda$iv.invoke();
    System.out.println(var3);
    int var4 = false;
    String var6 = "print6";
    System.out.println(var6);
}

public static final void print2and3() {
    String var0 = "print2";
    System.out.println(var0);
    var0 = "print3";
    System.out.println(var0);
}

public static final void print5and6(@NotNull Function0 lambda, @NotNull Function0 lambda2) {
    int $i$f$print5and6 = 0;
    Intrinsics.checkNotNullParameter(lambda, "lambda");
    Intrinsics.checkNotNullParameter(lambda2, "lambda2");
    Object var3 = lambda.invoke();
    System.out.println(var3);
    var3 = lambda2.invoke();
    System.out.println(var3);
}
{% endhighlight %}

java 코드의 결과를 보면 `noinline`을 적용한 첫번째 고차함수는 익명클래스를 생성하고 있고, 변수에 별도 키워드 없는 두번째 파라미터는 `inline`이 적용된 모습을 볼수 있다.

## 3. 그래서?
그래서 모든 함수는 `inline`으로 처리하는게 좋을까? 당연히 정답은 X이다. `inline` 키워드에 대해 정리해보자.

`inline` 키워드는 고차함수를 파라미터로 받게되어, 불필요한 익명 클래스 생성을 막을 수 있도록 처리할 때 사용하면 좋다는 것을 알았다. 사실 이또한 객체 하나를 생성하는 비용보다 해당 고차함수를 파라미터로 받은 함수 자체의 로직이 크다면 의미가 없다. 왜냐면 `inline` 키워드가 붙은 함수의 코드를 옮겨오기 때문에 코드량이 2배가 되어 처리해야할 byte량이 많아진다. 

결국 아래와 같은 타겟에 `inline` 키워드를 적용 하는게 가장 효과적이라 할 수 있다.
> 고차함수를 파라미터로 받는 적은량의 코드를 가진 함수


<hr>
`ref.`

* <https://amitshekhar.me/blog/inline-function-in-kotlin>