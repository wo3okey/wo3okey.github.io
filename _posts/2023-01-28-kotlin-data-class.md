---
layout: post
title: 그래서 kotlin data class 상속은?
categories: [kotlin]
tags: [kotlin, data class]
---

java의 lombok을 사용할 필요 없도록 잘 설계된 kotlin의 data class의 활용성은 한번 사용해본 개발자라면 공감할 것이다. data class를 상속의 관점에서 알아본다.

<hr>

## 1. Data class
java의 lombok도 편하지만 kotlin data class는 기본적인 메소드들을 만들기 진짜 세상 편하다. 하지만 상속을 할때에는 꼭 유의해야하는 사항이 있다. 차근차근 알아보자.

### hash code
먼저 data class를 선언 했을때 컴파일러가 만들어주는 hash code 메소드를 살펴보자. 간단한 `Person` 객체를 만들었다. 그리고 java 코드로 decompile하여 `hashCode()`만 가져왔다.

`kotlin`
{% highlight kotlin %}
data class DataClassPerson(
    val name: String,
    val age: Int,
    val phone: String
)
{% endhighlight %}

`java`
{% highlight kotlin %}
public int hashCode() {
    String var10000 = this.name;
    int var1 = ((var10000 != null ? var10000.hashCode() : 0) * 31 + Integer.hashCode(this.age)) * 31;
    String var10001 = this.phone;
    return var1 + (var10001 != null ? var10001.hashCode() : 0);
}
{% endhighlight %}

`hashCode()`를 살펴보면 단순히 정의된 변수의 값으로만 hash값을 만들고 있다. 즉 선언된 객체의 변수값들이 같으면 같은 hashcode를 갖는다는 것이다. 바로 한번 확인해보자.

`kotlin`
{% highlight kotlin %}
fun main() {
    val a = DataClassPerson("A", 30, "010-1234-5678")
    val a1 = DataClassPerson("A", 30, "010-1234-5678")
    val b = DataClassPerson("B", 30, "010-1234-5678")

    println(a.hashCode() == a1.hashCode())
    println(a.hashCode() == b.hashCode())
}
{% endhighlight %}

```
true
false
```

예상대로 위 경우 a, a1에 대해 객체의 값이 모두 같으므로 hash code 비교결과가 `true`로 나온다. 또한 a, b에 대해 name값이 다르므로 hash code 비교 결과가 `false`이다. 단순히 변수값만 비교한 값이므로 당연한 결과이다. 그렇다면 data class가 아닌 일반 class는 어떨까?

`kotlin`
{% highlight kotlin %}
class NormalClassPerson(
    val name: String,
    val age: Int,
    val phone: String
)

fun main() {
    val a = NormalClassPerson("A", 30, "010-1234-5678")
    val a1 = NormalClassPerson("A", 30, "010-1234-5678")

    println(a.hashCode() == a1.hashCode())
}
{% endhighlight %}
```
false
```
a, a1은 hashcode가 같지 않다는 `false` 결과를 바로 확인 할 수 있다.

## 2. Data class 상속
data class의 부모객체(SuperClass)를 하나 설정해보자. 

`kotlin`
{% highlight kotlin %}
data class SuperClass(
    val superData: String = ""
)

data class DataClassPerson(
    val name: String,
    val age: Int,
    val phone: String,
): SuperClass()
{% endhighlight %}

일단 위 코드는 컴파일 되지 않는다. data class끼리 상속하거나 받을순 없다. 이유는 간단하다. 
* decompile java 코드를 보면 data class는 기본적으로 `final` 클래스로 정의 되어있기에 상속을 막아뒀다.
* 만약 억지로 상속한다 해도 data class에서 만들어주는 기본 메소드들을 부모와 자식것 중 어떤걸로 선택해야할지 정의할 수 없을 것이다. 그렇다고 강제로 어느것으로 정의하기에도 좀...

그래서 애초에 막아뒀다고 생각한다. 그렇다면 부모 객체 및 변수에 상속 가능한 `open` 키워드를 붙이고, `hashCode()` 까지 재정의하여 data class에 상속을 해보자.

`kotlin`
{% highlight kotlin %}
open class SuperClass(
    open var superData: String = ""
) {
    override fun hashCode(): Int {
        return 1
    }
}

data class DataClassPerson(
    val name: String,
    val age: Int,
    val phone: String,
): SuperClass()

fun main() {
    val a = DataClassPerson("A", 30, "010-1234-5678")
    val a1 = DataClassPerson("A", 30, "010-1234-5678").apply { superData = "super" }

    println(a.hashCode() == a1.hashCode())
}
{% endhighlight %}
```
true
```
a1에 상속받은 클래스의 `superData` 값을 변경했지만 a, a1의 hashCode는 같다. 이유는 간단하다. kotlin에서는 상속을 받을 때 이미 자식 객체에서 기본 메소드가 정의 되어있다면 이는 재정의 하지 않기 때문이다. 그래서 a, a1은 `DataClassPerson`에 정의된 변수만 가지고 hashcode 값을 정의한다. 그렇다면 상속받은 두 객체 a, a1의 hashcode는 어떻게 구분할 수 있는가? 답은 간단하다. 상속은 상속답게 `override`로 부모 변수를 받아서 재정의 하면 된다.

`kotlin`
{% highlight kotlin %}
data class DataClassPerson(
    val name: String,
    val age: Int,
    val phone: String,
    override var superData: String = ""
): SuperClass()

fun main() {
    val a = DataClassPerson("A", 30, "010-1234-5678")
    val a1 = DataClassPerson("A", 30, "010-1234-5678").apply { superData = "super" }

    println(a.hashCode() == a1.hashCode())
}
{% endhighlight %}
```
false
```

## 3. 그래서?
java, kotlin 할 것 없이 상속은 간단하고 자주 사용되는 기본적인 문법이지만, 항상 주의를 기울여야한다. 그래서 kotlin 상속시 다음 유의 사항들은 살펴보면 좋을것 같다.
> 상속한 객체의 변수는 가능한 override 해서 사용할것.<br>
> 반드시 부모객체의 정보를 받아야는게 아니라면, 상속이 아닌 interface를 활용하여 재정의 할것.