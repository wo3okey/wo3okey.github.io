---
layout: post
title: 그래서 kotlin iline 함수가 뭔데?
categories: kotlin
tags: [kotlin, java, iline]
---

![kotlin inline]({{site.url}}/assets/images/kotlin-inline-01.png )

kotlin 함수에 붙는 `inline` 키워드는 말 그대로 호출되는 특정 코드 line 사이에 특정 inline 키워드가 붙은 함수의 코드를 넣을(in) 수 있도록 지원하는 키워드이다.

## inline function

간단하게 아래 코드로 살펴보자. 아주 단순한 역할을 해주는 키워드 그 잡채이다.


`Kotlin`
{% highlight kotlin %}

fun main() {
    println("print1")
    print2and3()
    println("print4")
    print5()
}

fun print2and3() {
    println("print2")
    println("print3")
}

inline fun print5() {
    println("print5")
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

kotlin 코드에는 2개의 print는 직접 처리하며, 2개의 print는 함수를 호출해서 출력되는 것으로 보인다. 하지만 이를 java 코드로 변환한 결과를 보면 다소 충격적이다. 실제로 `print5` 함수는 선언만 되어 있을뿐 호출하지 않는다. 내가 만든 함수가 호출되지 않다니! 하지만 해당 함수에 있는 모든 코드가 main문 안에 그대로 들어가 있는 모습이 보인다. 이것이 `inline` 키워드의 역할이다. 그렇다면 이렇게 코드를 2배로 만들게 하는 키워드가 왜 필요할까?

<hr>
`ref.`

* <https://amitshekhar.me/blog/inline-function-in-kotlin>