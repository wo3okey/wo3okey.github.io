---
layout: post
title: 그래서 HTTP 프로토콜은 어떻게 진화하고 있는가?
categories: [cs]
tags: [cs, HTTP, HTTP 0.9, HTTP 1.1, HTTP 2.0, quic]
---

HTTP 프로토콜은 TCP기반의 HTTP 0.9, 1.1, 2.0 그리고 UDP기반의 QUIC까지 계속 진화하고 있다. 각 버전마다 발생되는 단점을 다음 버전에서 극복하며, 지속적으로 HTTP 통신 속도를 높여가고 있다. HTTP의 발자취를 함께 따라가보자.

## TPC/UDP
먼저 HTTP 관련 정보에 단골로 등장하는 내용인 TPC/UDP를 아주 얕게 설명한다. TPC/UDP의 동작 방식이나 차이점 등은 교수님 강의에서도 필수로 등장한다. 간단하게 그림으로 살펴보자.

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-01.png)

TCP 통신에는 `3 handshake`라고 부르는 과정이 데이터 통신에 선행된다. 이는 서버와 클라이언트 간에 `SYN`와 `ACK`패킷을 통해 서로의 연결에 대한 신뢰성을 확보하는 과정을 포함한다. 결국 TCP가 UDP에 비해 느린 이유는 신뢰성을 보장하기 위한 통신 과정이 포함 되기 때문이다. 이번 포스팅에서는 TPC/UDP에 대해 아래 표 내용으로 얕게 정리한다.

| | TCP | UDP |
| --- | --- | --- |
| 연결 방식 | 연결형 서비스 | 비연결형 서비스 |
| 패킷 교환 | 가상 회선 방식 | 데이터그램 방식 |
| 전송 순서 보장 | 보장함 | 보장하지 않음 |
| 신뢰성 | 높음 | 낮음 |
| 전송 속도 | 느림 | 빠름 |


## HTTP/0.9
초기단계 HTTP 프로토콜은 매우 단순하다. 초기에는 버전 번호가 없었고, HTTP/0.9는 차후 버전과 구별하기 위해 그냥 0.9로 부른 것이다.

```
GET /mypage.html
```
HTTP/0.9의 요청은 단일 라인으로 구성되며 연결 가능한 HTTP method는 `GET`이 유일하다. 지극히 단순하다.

```
<HTML>
A very simple HTML page
</HTML>
```
응답 또한 극도로 단순했으며, 오로지 html 파일 내용 그 자체로 구성된다. HTTP 헤더도 없을뿐 아니라, 오류 코드 또한 없다.

## HTTP/1.0
HTTP/1.0에서는 HTTP/0.9에 비해 브라우저 및 서버에 대해서 모두 확장성 있게 진화했다.
```
GET /mypage.html HTTP/1.0
User-Agent: NCSA_Mosaic/2.0 (Windows 3.1)
```
요청 `header` 개념이 추가되었다. 메타데이터를 전송할 수 있게 된 것이다. 

```
200 OK
Date: Tue, 15 Nov 1994 08:12:31 GMT
Server: CERN/3.0 libwww/2.17
Content-Type: text/html
<HTML>
A page with an image
  <IMG SRC="/myimage.gif">
</HTML>
```
응답 정보에도 마찬가지로 `header` 개념이 추가되었다. 응답값에 상태코드가 붙기 시작했으며, `Content-Type` 도입으로 html 이외의 문서 전송도 가능해졌다.

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-02.png)

HTTP/1.0 에서는 커넥션 하나당 하나의 요청과 응답만 처리 가능했다. 그래서 요청마다 커넥션을 맺고 닫고를 반복하는 구조이다. 즉, 매번 새로운 요청마다 새로운 연결로 인해 성능이 떨어지고 서버에 부하가 생길 수 있다.

## HTTP/1.1
HTTP/1.1은 HTTP의 첫번째 표준 버전이다. 

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-06.png)
HTTP/1.0의 단기 커넥션 방식에서 HTTP/1.1로 진화하면서 persistent connection(영속적인 커넥션), pipelining(파이프라이닝)을 지원하게 되었다. 위 그림은 각 방식에 따른 커넥션 연결 및 요청 방식과 네트워크 타임 감소를 한번에 볼 수 있다.

### persistent connection
![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-03.png)

HTTP/1.1에서는 HTTP/1.0에서의 불필요한 커넥션 문제를 해결 하기위해 persistent connection이 도입 되었다. 지정한 timeout 동안 커넥션을 닫지 않는 방식이다. 영속적인 커넥션은 연결 정보를 열어두고 여러 요청을 재사용 함으로써 새로운 TCP 핸드쉐이크 비용을 아끼고 TCP 성능을 향상 시킬 수 있었다. 물론 연결을 유지하려는 시간이 길어질수록 서버에 부하가 생기기 때문에 연결을 유지하는 시간을 제한하고 있으며, 이를 `Keep-Alive`라고 부른다.

영속 커넥션 방법도 단점이 있다. 커넥션이 유후 상태일때에도 서버 리소스를 소비하며, 과부하 상태에서는 DOS 공격을 받을 수 있다. 이러한 경우에는 커넥션이 유휴 상태가 되자마자 닫히는 비영속 커넥션을 사용하는 것이 더 나은 성능을 보일 수 있다.

### pipelning
![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-04.png)
HTTP은 순차적으로 요청, 응답하는 프로세스를 가졌다. 즉, 현재 요청에 대한 응답을 받고 나서야 다음 요청을 실시 한다. 이는 먼저 오는 요청에 지연이 있다면 다음 요청들은 모두 지연이 걸려, 전체 네트워크 시스템에 상당한 부하와 딜레이를 발생시킬 수 있는 것이다. 파이프라이닝은 영속적인 커넥션으로 연결하여, 응답을 기다리지 않고 요청을 연속적으로 보내는 기능이다. 이것은 커넥션 지연을 회피하는 방법이다. 이론적으로 여러개의 HTTP 요청을 하나의 TCP 메세지 안에 채워서 발송하는 것이다. HTTP의 요청 사이즈가 계속 커지고 있지만, 일반적인 TCP에서는 최대 세그먼트 크기 내에서 몇개의 간단한 요청을 포함해서 보내기에는 여유로웠다.

하지만 파이프라이닝 방법은 여러가지 버그와 구현의 복잡성 등 다양한 문제로 모던 브라우저들에서 기본적으로 활성화 되어 있지 않는 기능이다. 이를 대체하기 위해 HTTP/2.0 에서는 `멀티플렉싱` 방식으로 대체 되었다. 또한 파이프라이닝 방식으로는 `HOLB` 및 `중복헤더` 문제는 해결하지 못했다.

### HOLB 문제
![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-05.png)
HOLB는 `head of line blocking`을 뜻한다. 영속적인 커넥션 내에서 여러 요청을 timeout 시간동안 연속적으로 받을 수 있으나, 특정 요청의 응답이 밀리게 되면 뒤따라오는 모든 요청들의 응답은 밀릴 수 밖에 없는 문제점이 있다. 파이프라이닝 방법을 사용하더라도 하나의 TCP요청 내 여러 요청을 병렬적으로 보내어도, 결국 여러 요청중 하나의 응답이 늦어지면 전체적으로 해당 커넥션 자체의 응답은 늦어질 수 밖에 없는 문제가 있다.

### 중복헤더 문제
HTTP/1.1은 초기 HTTP와 다르게 헤더에 많은 메타 정보가 들어간다. 요청이 연속적으로 이루어질 때 동일한 헤더의 값도 연속적으로 보내며, 이는 곧 통신간에 쓸데없는 데이터를 중복으로 보내게 되는 문제가 있다.

## HTTPS
한편, HTTP의 보안 강화를 위한 HTTPS(HTTP Secure)가 등장한다. 본 포스팅에서는 HTTPS에 대해서 자세히 다루지는 않는다.

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-07.png)

HTTPS 프로토콜은 HTTP 프로토콜의 보안을 강화하기 위해 `SSL(Secure Sockets Layer)` 보안 소켓 계층을 사용하는 프로토콜이다. 즉, HTTPS는 HTTP에 SSL을 추가한 것으로, HTTP 요청에 암호화와 인증 등의 보안 기능을 제공하여 데이터의 안전성을 보장한다. 따라서, HTTPS는 인터넷 상에서 데이터를 안전하게 전송하는 데 매우 중요한 역할을 한다. 

`TLS(Transport Layer Security)` 또한 SSL과 마찬가지로 보안을 위한 암호화 프로토콜이다. 다만 암호화 방식과 보안 강도가 다르기 때문에 보안 수준의 차이는 있다. SSL3.0, TLS1.0, 1.1같은 오래된 버전은 보안 취약점이 발생될 가능성이 높기 때문에 포스트 작성 기준 TLS 1.2, 1.3을 권장한다.

들리는 말로는 구글에서 SEO를 진행할 때, HTTPS로 호스팅된 사이트의 페이지에 대해 SEO 우선순위를 높여준다고 한다. 보안성 높은 페이지를 제공함에 있어 가산점을 주는듯 하다.

## HTTP/2.0
![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-08.png)

네이버의 포털의 메인 페이지다. 오른쪽 메인 페이지의 호출 정보를 보면 Protocol 메뉴에 `h2`라는 것이 보인다. 현재 우리나라에서 가장 많은 접속량의 네이버 메인 포탈에서 HTTP/2.0 프로토콜을 사용한다. HTTP/2.0는 HTTP/1.x의 다음 버전으로, 웹 성능과 안전성을 개선하기 위해 설계 되었다.

### binary protocol
![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-09.png)

HTTP/2.0 에서는 HTTP 메세지의 전송 방식을 변경한다. 기존 HTTP/1.x에서 text 형태로 처리하던 메세지를 HEADERS/DATA 각각을 frame이라는 단위로 분할하고 binary로 인코딩한다. 이는 곧 text가 아닌 binary로 변환 함으로써 컴퓨터 입장에서는 메세지의 파싱 처리가 더욱 빨라진다. 이는 곧, 전송속도 향상 및 오류 발생 가능성을 낮추는 효과를 볼 수 있다.

### multiplexing
HTTP/2.0에는 HTTP/1.x의 메세지 라는 단위 외에 frame, stream 이라는 단위가 추가 되었다.
* frame: HTTP/2.0 통신상 가장 작은 정보의 단위이며, Header or Data
* message: HTTP/1.x 와 마찬가지로 요청 혹은 응답의 단위이며, 다수의 frame로 이루어짐
* stream: 클라이언트와 서버사이 맺어진 연결을 통해 양방향으로 주고받는 하나 혹은 복수의 message
> frame < message < stream

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-10.png)

1개의 stream을 보면 GET요청이므로 header stream 하나를 메세지로 보내고 header, data frame을 각각가진 메세지를 응답 받고 있다. 그리고 아래 N개의 stream을 보면, 복수 요청 및 응답 메세지들이 하나의 stream에서 존재하는 모습을 볼 수 있다. 이처럼 HTTP/2.0에서는 HTTP/1.x와 다르게 요청과 응답은 하나의 메세지만 담당하는 것이 아닌 하나의 stream이 다수의 요청과 응답을 받을 수 있는 구조로 바뀐 것이다. 즉, 이를 `멀티플렉싱(multiplexting)`이라 한다. 

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-12.png)
멀티플렉싱은 TCP 연결에서 여러개 요청과 응답을 병렬로 처리 하는 방식이다. 이를 통해 모든 요청과 응답을 하나의 TCP 연결을 통해 모든 요청과 응답을 처리할 수 있다. 덕분에 HTTP/1.x 에서 발생하던 고질적인 지연 문제인 HOLB도 자연스럽게 해결되었다.

### server push
HTTP/2.0에서는 `서버 푸시(Server Push)`라는 기능을 제공한다. 이 기능은 서버가 클라이언트의 요청에 대한 응답으로 요청한 리소스를 보내는 것뿐만 아니라, 클라이언트가 요청하지 않은 리소스를 미리 보내주는 기능이다.

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-13.png)

클라이언트가 요청한 `/page.html`을 받고, 이후 서버에서는 요청간에 `script.js`와 `style.css`가 필요할 것 이라 예측하고 이를 미리 보내준다. 이러한 서버푸시 기능을 통해 추가적인 통신 단계를 줄여, 불필요한 네트워크 지연을 줄일 수 있다. 이를 통해 웹 페이지의 로딩 속도를 더욱 빠르도록 지원 할 수 있다.


### header compression
HTTP/1.1에서는 요청 및 응답 헤더에 많은 정보가 포함되는데, 이러한 헤더 정보는 매 요청마다 반복적으로 전송되기 때문에 중복이 발생하며, 이로 인해 데이터 전송 속도가 느려지는 문제가 발생한다. HTTP/2.0에서는 이러한 문제를 해결하기 위해, `헤더 압축(Header Compression)` 방법을 통해 데이터를 전송한다. 이를 위해, HPACK이라는 새로운 헤더 압축 방식을 사용한다. 이때 `Huffman Encoding` 알고리즘이 사용된다.

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-14.png)

첫번째 `request1` 요청에 6개 필드를 header frame에 담아 stream1 통해 보내고, 두번째 `reuqest2`에서는 중복되지 않는 path 정보만 header frame에 담아 stream3을 통해 보냈다. 이렇듯 HTTP/2.0에서는 중복헤더 데이터를 전송하지 않음으로써 매 요청마다 오버헤드를 크게 줄일 수 있다. 결국 이러한 헤더 압축 방법을 통해 데이터 전송 속도를 높일 수 있다.

## HTTP/3.0
HTTP/2.0이 나온지 얼마 되지 않아, 세번째 HTTP 메이저 버전인 HTTP/3.0이 구글에 의해 공개 되었다.

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-16.png)

먼저 TCP의 헤더를 보자. TCP는 기본적인 신뢰성을 위한 기능 외에 옵션을 추가할 수 있도록 헤더에 필드를 제공한다. 옵션은 무한정 제공할 수는 없으니 설계상 320bits로 정해놓았다. TCP의 단점을 보안하기 위해 나중에 정의된 MSS(Maximum Segment Size), WSCALE(Window Scale factor), SACK(Selective ACK) 등 많은 옵션들이 필드를 차지하고 있다. 즉, 사실상 더이상 커스텀할 수 있는 옵션 자리가 많지 않은 상황이다.

반면에 UDP의 헤더 모습을 살펴보자. 한눈에 보아도 TCP와 UDP의 복잡함의 차이가 한눈에 보인다. 데이터 전송 자체에만 초점을 맞추고 설계 되었기 때문에 흰 도화지나 다름이 없는 상태이다. 그래서 구글이 HTTP/3.0을 설계할때 TCP가 아닌 UDP를 선택한 것이다. 이와 동시에 `QUIC(Quick UDP Internet Connections)`이라는 UDP기반의 전송 프로토콜을 함께 개발했다.

![HTTP protocol]({{site.url}}/assets/images/posts/http-protocol/http-protocol-17.png)

HTTP/3.0 QUIC는 TCP를 사용하지 않기 때문에 3 way handshake 과정을 거치지 않는다. QUIC는 `0-RTT(Zero-Round Trip Time)`를 지원하여 최초 연결 설정 시간이 매우 짧다. 또한 TLS1.3와 결합하여 암호화된 연결을 제공한다. 즉, QUIC는 TCP에서 가지고 있는 고질적인 레이턴시 문제는 해결하면서 동시에 보안성을 강화시킨 UDP 기반 HTTP 프로토콜이다.

## 그래서?
HTTP는 내가 태어나기도 전부터 인터넷이 발달하는 그 순간에 만들어진 개념이다. TCP, UDP도 마찬가지로 꽤 예전에 만들어진 프로토콜이다. 반세기가 흘러 지금도 이러한 근본적인 HTTP에 대한 개념을 기반으로 확장성 있는 구조로 프로토콜이 발전하고 있는 것이다. 어찌보면 참으로 신기하다. 마치 역사적 가치와 의미를 보존하며 현대 문화에 맞게 모습과 기능을 발전시킨 한옥 같은 느낌이랄까...

> 그래서 초기 HTTP 모습은 이론적으로만 봐왔지만, 계속 발전해나갈 HTTP의 모습을 응원하고 이에 발맞추어 같이 성장하고 싶다.

ps. `HTTP/3.0` 및 `QUIC`에 대한 내용은 다른 포스팅에서 자세히 정리 해야겠다.

{% include ref.html %}
* <https://www.linkedin.com/pulse/3-tcp-udp-there-way-handshake-cyber-security-basics-ertan-kaya>
* <https://evan-moon.github.io/2019/10/08/what-is-HTTP3/>
* <https://developer.mozilla.org/ko/docs/Web/HTTP/Basics_of_HTTP/Evolution_of_HTTP>
* <https://yozm.wishket.com/magazine/detail/1686/>
* <https://heidyhe.github.io/https/>
* <https://http2.tistory.com/13>
* <https://www.popit.kr/%EB%82%98%EB%A7%8C-%EB%AA%A8%EB%A5%B4%EA%B3%A0-%EC%9E%88%EB%8D%98-http2/>
* <https://donggov.tistory.com/188>