---
layout: post
title: 그래서 elasticsearch shard, replica 값은 어떻게 설정할까? 
categories: elasticsearch
tags: [elasticsearch, shard, replica]
---

![elasticsearch shard replica]({{site.url}}/assets/images/posts/elasticsearch-shard-replica-01.png )

elasticsearch를 운영하기 위해서는 shard와 replica 갯수를 적절히 설정해야한다. 또한 index에 설정된 shard 수는 한번 설정하면 변경이 불가하기 때문에 운영중에 index를 변경하는 번거로움을 덜기 위해서는 적절하게 갯수를 선정하는 것이 중요하다. 그렇다고 한번 설정한 shard는 정답이 아니다. 시스템이 커지고 변경됨에 따라 언제든 변경의 여지를 두고 지켜봐야할 대상이다. 관련해서는 다음 포스팅에서 더 자세히 다룬다.

## shard
![elasticsearch shard replica]({{site.url}}/assets/images/posts/test2.png )
shard는 mysql과 같은 RDB를 기준으로 partition과 같은 의미로, 데이터를 저장할 때 나누어진 조각 단위라고 생각하면 된다. 즉 shard의 데이터는 복사본이 아닌 저장한 데이터 그 자체이다. elasticsearch에서는 충분히 크기가 큰 데이터를 가진 index의 데이터를 특정한 파티션 단위로 나누며, 이를 shard라고 부른다. 각 document 문서는 해당 문서의 hash 값을 통해 계산된 shard로 라우팅 된다. 그리고 일반적으로 `shard`라고 하는 것이 primary shard를 뜻한다. 복사되지 않은 CRUD의 주체가 되는 실제 원본 데이터를 뜻한다. 그렇다면 replica shard라고 불리는 replica는 어떠한 역할을 하는 것일까?

### shard 값 설정

## replica
![elasticsearch shard replica]({{site.url}}/assets/images/posts/shard, replica-003 (2).png )
replica를 한 마디로 정의하면 `shard의 복제된 갯수` 로 정의 할 수 있다. 만약 primary shard가 3일때, replica를 2로 설정하면 replica shard의 갯수는 6개로, 총 9개의 shard가 존재한다는 뜻이다. 

replica shard는 각 node에 분산된 primary shard의 복제 역할을 함으로써 두가지 장점의 기능을 갖게 된다. 특정 node로부터 온 요청사항을 다른 node에 찾는 번거로움 없이 primary shard와 함께 온전히 지원할 수 있음으로써 검색성능 향상(search performance)을 기대할 수 있다. 그리고 node 자체에 장애가 발생하더라도 서비스에는 문제 없도록 장애복구(fail-over) 역할을 할 수 있다. 위 그림에도 node 1~3중 하나가 문제 생기더라도 primary, replica shard 관계없이 각각의 node의 shard 0~2번까지 모두 서비스 될 수 있기 때문이다.

### replica 값 설정
replica shard는 중요한 특성을 가지고 있다.
> replica shard는 절대 primary shard와 동일한 데이터를 갖도록 node 내에서 존재 할 수 없다.



## 다다익선?

## 그래서?

{% include ref.html %}
* https://aws.amazon.com/ko/blogs/database/get-started-with-amazon-elasticsearch-service-how-many-shards-do-i-need/
* https://brunch.co.kr/@alden/39
* https://fdv.gitbooks.io/elasticsearch-cluster-design-the-definitive-guide/content/a-few-things-you-need-to-know-about-lucene.html
* https://esbook.kimjmin.net/03-cluster/3.2-index-and-shards