---
layout: post
title: 그래서 AWS kinesis는 무엇인가?
categories: [aws]
tags: [aws, kinesis]
---

AWS에서는 실시간 데이터 스트리밍을 처리할 수 있는 kinesis를 서비스한다. kafka와 마찬가지로 로그 수집 파이프라인, 이벤트 메세지 큐, 스트리밍 서비스 등에 사용되고 있다. 간략하게 알아보자.

## kinesis 란
`kinesis`는 대규모의 실시간 데이터 스트리밍 처리를 지원하는 pub/sub 모델을 구현한 AWS 관리형 서비스이다.

### data strems
kinesis data streams의 아키텍처 구조이다. 용어를 정리하면서 간단히 알아보자.

![AWS-kinesis]({{site.url}}/assets/images/posts/AWS-kinesis-02.png)

| 구성 | 설명 |
| --- | --- |
| producer | - data stream에 데이터를 보내는 주체의 서버 또는 애플리케이션 |
| consumer | - data stream에서 데이터를 읽어들이고 처리하는 주체의 서버 또는 애플리케이션 |
| stream | - 데이터 스트리밍을 처리하는 기본 단위이며, data recode로 구성 |
| shard | - stream 내에서 data recode의 처리를 병렬화 하는 방법이며, stream을 구성하는 단위 <br> - stream은 하나 이상의 shard로 구성 |
| data recode | - stream을 통해 전송되는 데이터 가장 작은 데이터 단위 <br> - sequence number, partition key, Blob로 구성됨 <br> - 일반적으로 json으로 작성하며, 최대 1MB까지의 크기를 가지고 shard를 통해 분산 처리 |
| partition key | - stream 내에서 shard별로 데이터를 그룹화 하는데 사용 <br> - 동일한 partition key를 가진 data recode는 동일한 shard로 라우팅 <br> - MD5 hash 함수를 통해 data recode를 shard 매핑 |
| sequence number | - 각 data recode의 유일한 식별자이며, 모든 데이터 레코드에는 시스템에서 자동으로 생성된 고유 번호 할당 <br> - sequence number는 각 shard에서 recode 순서를 유지하는데 사용 |

### KPL/KCL
AWS에서는 KPL, KCL을 지원하여 kinesis data stream을 쉽고 빠르게 구축 할 수 있도록 지원한다.

* `KPL(Kinesis Producer Library)`은 kinesis data stream에 데이터를 추가할 수 있는 애플레케이션을 쉽게 구축 할 수 있도록 지원하는 라이브러리
* `KCL(Kinesis Client Library)`은 kinesis data stream의 데이터를 읽고 처리하는 애플리케이션을 쉽게 구축할 수 있도록 지원하는 라이브러리


## 용량 모드
kinesis data stream을 생성할 때에는 두가지 비용의 용량모드를 선택할 수 있다.

![AWS-kinesis]({{site.url}}/assets/images/posts/AWS-kinesis-03.png)

* 온디맨드(on-demand)
    * stream 처리량이 불규칙하거나 예측할 수 없는 경우에 사용을 권장
    * 데이터 처리량에 따라 shard의 수를 조정해주며, 처리한 데이터량에 따라 비용이 결정
    * shard의 수를 직접 늘릴 수 없음
* 프로비저닝(provisioned)
    * stream 처리량을 어느정도 예측할 수 있는 경우에 사용을 권장
    * shard 수를 직접 설정하여 stream을 생성하여, 미리 할당한 shard의 갯수에 따라 비용이 결정
    * shard의 수를 직접 늘릴 수 있음

요약하자면, 온디맨드 모드는 비교적 예측할 수 없는 데이터 stream 처리량에 적합하며, 프로비저닝 모드는 처리량을 예측할 수 있는 경우에 적합하다. 어떤 모드를 선택하든, 처리량에 따라 샤드당 시간당 요금이 발생하므로 처리할 데이터 양과 예상되는 비용을 고려하여 적절한 모드를 선택해야 한다.

### 모드 변경
kinesis data stream은 필요에 따라 라이브 서비스 중에도 즉각적으로 용량 모드를 변경할 수 있다. 아래의 예시는 AWS 블로그에서 예시 상황을 설명한 글이다.

![AWS-kinesis]({{site.url}}/assets/images/posts/AWS-kinesis-04.png)
먼저 5개의 shard로 프로비저닝 모드로 생성된 stream이 있다고 한다. 처음 3분 동안 4MB/s의 로드를 전송하고 있고, 정상적으로 처리 하고 있다. 이후 타임스탬프 21:19에 로드를 12MB/s로 늘리며, stream이 더이상 로드를 처리할 수 없게되며 빨간색 선의 제한된 요청에 대한 수치가 높아지고 있는 모습이 보인다.
이에 따라 타임스탬프 21:23에서 프로비저닝 용량모드에서 온디맨드 모드로 변경한다. stream에 영향을 주지 않고 즉시 적용할 수 있다.

![AWS-kinesis]({{site.url}}/assets/images/posts/AWS-kinesis-05.png)
타임스탬프 21:24 시점부터 stream이 확장되면서 제한이 떨어지기 시작한다. 타임스탬프 21:26 시점에 stream 용량은 최초5개의 shard의 2배인 10개의 shard로 확장되고, 각 shard에서 로드가 0.5MB/s 미만의 수준으로 처리가능 할 때까지 stream은 계속 확장된다. 

![AWS-kinesis]({{site.url}}/assets/images/posts/AWS-kinesis-06.png)
이후 빨간색의 제한된 처리에 대한 수치가 사라지고, 안정적으로 서비스되고 있는 모습을 볼 수 있다. 이처럼 stream이 갑자기 대량의 로드를 받게되면 온디맨드 모드에서는 이를 처리할 수 있는 shard 용량을 확보할 수 있다.

온디맨드 모드와 프로비저닝 모드 간을 하루에 두 번 전환할 수 있다. 프로비저닝된 모드에서 온디맨드 모드로 전환할 때 데이터 스트림의 shard 수는 동일하게 유지되며 그 반대의 경우도 마찬가지이다. 반대의 경우에는 kinesis data streams는 데이터 트래픽을 모니터링하고 트래픽 증가 또는 감소에 따라 온디맨드 data stream의 shard 수를 늘리거나 줄인다.

## shard
`shard`는 간단하게 stream을 구성하는 단위라고 위에서 설명했다. 좀 더 자세히 알아보자.

### 처리량
shard의 갯수는 곧 stream에서 단위시간에 처리할 수 있는 data recode의 갯수를 뜻한다. kinesis data stream에서 제공하는 shard 관련 처리량 스펙은 아래와 같다.
* 단일 stream 기준 최대 설정 가능 shard 수는 200개(50개 이상부터는 AWS에 문의해야함)
* 단일 shard 기준 읽기 처리량은 초당 5 transaction, 최대 2MB까지 지원
* 단일 shard 기준 쓰기 처리량은 초당 1000개의 data recode 또는 초당 1MB 까지 지원

### shard 갯수 설정
용량모드 중 `프로비저닝` 모드에서는 최초 stream 생성시 shard의 수를 지정해줘야 한다. 그렇다면 적절한 shard의 수는 어떻게 알 수 있을까?

* average_data_size_in_KB: 스트림에 쓰여지는 평균 데이터 레코드 수(KB), 1KB에 가깝게 반올림한 데이터 레코드 수
* records_per_second: 스트림에서 초당 쓰고 읽는 데이터 레코드 수
* incoming_write_bandwidth_in_KB: average_data_size_in_KB에 records_per_second를 곱한 값
* outgoing_read_bandwidth_in_KB: incoming_write_bandwidth_in_KB에 number_of_consumers를 곱한 값

> number_of_shards = max(incoming_write_bandwidth_in_KiB/1024, outgoing_read_bandwidth_in_KiB/2048)

AWS에서 가이드 하는 개수 설정 방법은 조금 복잡해 보일 수 있다. 로그 데이터 파이프라인 서비스나 대량의 스트리밍 서비스 등의 작업이 아닌 이상 단일 shard 정도로도 꽤나 준수한 처리량을 제공받을 수 있다.

## 그래서?
AWS kinesis는 운영 설정이나 관리 포인트를 대폭 감소시키고 각종 라이브러리까지 지원하여 쉽고 빠르게 대규모 스트리밍 서비스를 개발할 수 있도록 지원하고 있다. 
특히 AWS 관리형 서비스 답게 별도의 복잡한 설정이나 시스템 구축에 대한 허들 없이도 가능한 것이 매우 장점으로 보인다. 다만 굳이 [kafka](https://wo3okey.github.io/kafka/2023/02/22/kafka.html)와 비교하자면 kinesis는 shard에 대해 속도나 용량등의 제한적 처리량을 강제화하고 있는것은 사실이자 단점일 수 있다.
> AWS를 사용하는 인프라 시스템에서 쉽고 빠르게 대규모 스트리밍, 이벤트 메세지 시스템 등을 구축하기에 kinesis 서비스는 꽤 좋은 선택지가 될 수 있다.

{% include ref.html %}
* <https://docs.aws.amazon.com/ko_kr/streams/latest/dev/introduction.html>
* <https://docs.aws.amazon.com/ko_kr/streams/latest/dev/how-do-i-size-a-stream.html>
* <https://www.softkraft.co/aws-kinesis-vs-kafka-comparison/>
* <https://aws.amazon.com/ko/blogs/korea/amazon-kinesis-data-streams-on-demand-stream-data-at-scale-without-managing-capacity/>