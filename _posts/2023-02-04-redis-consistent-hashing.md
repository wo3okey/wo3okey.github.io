---
layout: post
title: 그래서 redis cluster hash slot은 무엇인가?
categories: [redis]
tags: [redis, redis cluster, hash slot, consistent hashing]
---

redis cluster의 node간에 key 분산 방식으로 사용되는 hash slot에 대해 알아보며, nosql의 분산 key 파티셔닝 기법의 시초가 되는 consistent hashing도 함께 알아본다.

<hr>

## 1. redis cluster
redis cluster은 redis 3.0 버전 이상부터 추가되었으며 데이터 동기화, 복제, fail-over 등을 지원할 수 있는 기능이 추가된 향상된 redis라고 생각하면 된다. 특히 fail-over 관점에서 node의 장애 또는 통신 문제 등 master-slave 및 replica 기능을 통해 수준높은 가용성을 제공할 수 있다.

nosql은 각 node에 key를 할당할 때 특정 node에 집중되지 않고 분산처리를 위해 `consistent hashing` 이라고 하는 파티셔닝 방법을 대게 사용한다. consistent hashing은 redis 3.0 이전에는 다양한 버전에서 채택했다. 이후 redis cluster가 자리잡으면서 `hash slot` 이라는 방법을 선택했다. 

## 2. hashing
분산처리 key 분배 시 기본적으로 hasing에 의해 값을 결정하고 목적에 따라 일부 알고리즘을 변경해서 사용한다. 일단 redis cluster hash slot 설명에 앞서 key 분산처리를 위한 일반적인 hashing 및 consistent hashing을 먼저 알아보자.

### general hashing
먼저 일반적인 hashing 방법에 의해 key를 분산하면 어떻게 될까? 간단하게 알아보자.

![elasticsearch shard replica]({{site.url}}/assets/images/posts/redis-consistent-hashing-01.png)
* node의 수: 3(nodeA~C)
* 유입 key의 수: 9(key1~9)

key가 유입되면 특정 hash(key) 함수의 결과에 의해 특정 node로 결정된다고 가정한다. 3개의 node에 9개 key를 분산 시켰을 때 모습을 보자. 일단 아름답게 잘 분산되어 있도록 가정했다. 편-안

![elasticsearch shard replica]({{site.url}}/assets/images/posts/redis-consistent-hashing-02.png)

만약 nodeC가 장애 또는 서버통신 문제로 제기능을 할 수 없는 상태가 되었다고 하자. 그렇다면 nodeC에 분산된 key는 어떻게 될 것인가? 마음같아선 nodeA, B에 동일하게 하나씩 나눠주고 싶겠지만 결국 최초 정해진 분산 방법에 따라 전체 재해싱을 진행할 수 밖에 없다. 재해싱 후 nodeB, B의 평화에 곧 엄청난 불균형이 찾아왔다. 이는 곧 재해싱 만으로도 server에 부하를 가져오지만, 각 node 간의 key 불균형도 피할 수 없다. 그래서 고안된 방법 중 하나인 `consistent hashing`([논문링크](https://en.wikipedia.org/wiki/Consistent_hashing#History))을 소개한다.

### consistent hashing
consistent hashing은 node의 수에 의존하지 않고 추가/삭제된 node가 있을때 재분산 해야하는 key의 수를 최소화 하기 위한 방법이다. hash ring이라 불리는 추상적인 원 또는 링 형태에 CRC-32(2^32)라고 하는 hash 연산의 결과에 따라 node를 할당하고, 인입되는 key에 대해서 동일한 hash 연산을 통해 어떤 node에 배치할 지 결정해주는 key 분산 처리 방법이다. 이를 코드 및 시나리오를 함께보며 이해해보자.

`kotlin`
{% highlight kotlin %}

data class Node(val id: String)

class ConsistentHash(private val numberOfReplicas: Int, private val nodes: Set<Node>) {
    private val circle = TreeMap<BigInteger, Node>()

    init {
        for (node in nodes) {
            add(node)
        }
    }

    fun add(node: Node) {
        for (i in 0 until numberOfReplicas) {
            val digest = MessageDigest.getInstance("MD5").digest((node.toString() + i).toByteArray())
            val hash = BigInteger(1, digest).mod(BigInteger.valueOf(2).pow(32))
            circle[hash] = node
        }
    }

    fun remove(node: Node) {
        for (i in 0 until numberOfReplicas) {
            val digest = MessageDigest.getInstance("MD5").digest((node.toString() + i).toByteArray())
            val hash = BigInteger(1, digest).mod(BigInteger.valueOf(2).pow(32))
            circle.remove(hash)
        }
    }

    fun getNode(key: String): Node {
        if (circle.isEmpty()) return throw BadRequestException("circle is empty.")

        val digest = MessageDigest.getInstance("MD5").digest(key.toByteArray())
        val hash = BigInteger(1, digest).mod(BigInteger.valueOf(2).pow(32))

        val nodes = circle.keys.sorted()
        var node = nodes.firstOrNull { it >= hash }
        if (node == null) node = nodes.first()

        return circle[node]!!
    }
}

fun main() {
    val ch = ConsistentHash(
        numberOfReplicas = 1,
        nodes = setOf(
            Node("A"),
            Node("B"),
            Node("C"),
        )
    )
    val nodeOfKey = mutableMapOf<Node, MutableSet<String>>()

    (1..9).forEach {
        val key = "key$it"
        val node = ch.getNode(key)
        if (nodeOfKey.containsKey(node)) {
            nodeOfKey[node]!!.add(key)
        } else {
            nodeOfKey[node] = mutableSetOf(key)
        }
    }

    val sortedMap = nodeOfKey.toSortedMap(compareBy { it.id })
    sortedMap.forEach {
        println("${it.key.id}: ${it.value}")
    }
}
{% endhighlight %}

* node의 수: 3(nodeA~C)
* replica 수: 1(여기서 1은 별도의 복제 없이 node의 수와 동일함을 뜻함)
* 유입 key의 수: 9(key1~9)

![elasticsearch shard replica]({{site.url}}/assets/images/posts/redis-consistent-hashing-03.png)

```
nodeA: [key4, key7]
nodeB: [key1, key2, key5, key8, key9]
nodeC: [key3, key6]
```
코드의 실행결과는 print 값은 위와 같으며, 이를 그림으로 표현한 모습이다. 원리는 간단하다. 코드에 표현되어 있듯, 입력된 각 key에 대해서 `position = hash (key) mod (2 ^ 32)` 연산에 의해 위치가 결정되며, 이때의 위치는 시계방향 기준으로 가까운 node를 따라가도록 되어있다. 그리고 hash 값이 가장큰 nodeC의 값을 넘을 경우 첫번째 nodeA를 바라보도록 되어있기 때문에 결국 원형 또는 링의 모습을 띄도록 그릴 수 있는 것이다. 그렇다면 이번에도 nodeC가 장애 또는 네트워크 이슈로 인해 key의 재분산이 필요하다면 어떠할까?

![elasticsearch shard replica]({{site.url}}/assets/images/posts/redis-consistent-hashing-04.png)

```
nodeA: [key4, key7]
nodeB: [key1, key2, key3, key5, key6, key8, key9]
```

결과는 어느정도 예상해볼 수 있다. 시계방향으로 위치할 node가 결정나므로 key3, 6은 자연스럽게 nodeB에 재분산 된 모습이다. 하지만 결과론적으로 안타깝지만 일반적인 hashing과 같이 nodeA, B는 또다시 불균형을 피하지 못했다. 래도 전체 재해싱 및 재분산이 아닌 서버의 추가/장애시 일정 node의 key만 재분산 하면 되므로 특정 키들만 고생하면 된다. 어느정도의 부하는 줄일 수 있었지만 여전히 불균형이 존재한다. 이를 위해 고안된 방법이 바로 가상의 node(vnode)를 배치하는 방법이다.




## 3. hash slot



## 4. 그래서?

Redis의 Hash Slot 과 Consistent Hashing의 차이는 아래와 같습니다.

Redis의 Hash Slot

Redis의 Hash Slot은 매핑 된 키-값 쌍을 각각의 노드에 분배하기 위해 사용됩니다. 기본적으로 전체 키 슬롯의 범위는 0 ~ 16383 입니다. 노드는 슬롯의 구간을 할당 받아 관리합니다. 이는 노드가 늘어날 때 슬롯의 범위를 분배하는데 도움을 줍니다.

Consistent Hashing

Consistent Hashing은 노드가 늘어나거나 줄었을 때 데이터가 노드간 고르게 균등하게 분배되어 있도록 하기 위해 만들어진 해시 알고리즘입니다. 각 노드는 해시 값의 범위를 할당 받고, 모든 키는 해시 함수를 통해 연관된 노드로 라우팅됩니다.


{% include ref.html %}
* <https://severalnines.com/blog/hash-slot-vs-consistent-hashing-redis/>
* <https://en.wikipedia.org/wiki/Consistent_hashing#History>
