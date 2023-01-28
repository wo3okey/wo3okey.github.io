---
layout: post
title: 그래서 kotlin data class 상속은?
categories: kotlin
tags: [kotlin, data class]
---

![kotlin vs java]({{site.url}}/assets/images/posts/kotlin-vs-java-01.png )

## 1. Data class
java의 lombok도 편하지만 kotlin data class는 기본적인 메소드들을 만들기 진짜 세상 편하다. 하지만 상속을 할때에는 꼭 유의해야하는 사항이 있다. 

### hash code
먼저 data class를 선언 했을때 컴파일러가 만들어주는 hash code 메소드를 살펴보자.

{% highlight kotlin %}
data class Person(
    val name: String,
    val age: Int,
    val phone: String
)
{% endhighlight %}

{% highlight kotlin %}
public int hashCode() {
    String var10000 = this.name;
    int var1 = ((var10000 != null ? var10000.hashCode() : 0) * 31 + Integer.hashCode(this.age)) * 31;
    String var10001 = this.phone;
    return var1 + (var10001 != null ? var10001.hashCode() : 0);
}
{% endhighlight %}


fun main() {
    val a = Person("a", 30, "010-1234-5678")
    val b = Person("a", 30, "010-1234-5678")

    println(a.hashCode() == b.hashCode())
}

> true

위 경우 a, b에 대해 hash code 비교결과가 `true`로 나온다. 이유는 아래 decompile된 Person 객체의 hash code 메소드를 살펴보면 된다.