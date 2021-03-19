---
layout: post
title: 단축 url 서비스 개발
date: 2021-03-19 19:52:00 +0900
categories: [Project]
tags: [url-shortener, golang]
toc: true
comments: true
---

# introduction

- TMI: 예전에 kakaopay 개발자 채용때 단축 url 제작 과제를 받았다가 떨어진 적이 있다.
> 솔직히 탈락이 당연했다.

그때 부끄럽게 구현했던 생각도 나고, 2020년 연말에 갑자기 무료하기도 하고, 사내 시스템으로 제공하면 괜찮다는 생각으로 개발을 시작하게 되었다. 

이 글은, 시스템 구조에 대한 자세한 설명 보다는 단축url 구현에 사용했던 알고리즘을 정리하는 의도가 더 크다.

[link](https://github.com/fransoaardi/url-shortener) 에서 실제 구현을 확인할 수 있다.


# progress

## architecture

사내에 구축 할때는, aws 의 eks 를 이용, ingress-nginx, route53 을 사용해서 도메인을 할당받았고, cert manager 를 이용해서 letsencrypt 의 인증서를 자동으로 갱신하게끔 구성했다. 

aws elasticache(redis) 를 db 로 이용했다. redis 가 제공하는 expiry 기능을 사용해서 데이터가 1달 간격으로 사라지게 했다. 또한 redis 는 빠른 key 조회와, 자체 expire 기능이 있고 저장하려는 데이터 value 가 적어서, 가볍게 사용할 의도로는 가장 적합했다.

## algorithm

다른 서비스들을 보니, 일단 62 자리(`[0-9a-zA-Z]`) 기반으로 개발하는게 대세였고, 자릿수는 변동이 있었다.

- 변환과정: 단축하려는 url(:`url`) -> (hash) -> 단축된 형태로 변경(:`urlKey`)
- 저장과정: `urlKey` 를 redis 의 key 로 하고, value 에 `url` 을 저장하는 방식

hash 는 간단하게 `sha256` 을 사용했다.

```go
sum := sha256.Sum256([]byte(url))
```

`sha256.Sum256` 은 length 32 의 byte slice 를 return 하는데, 1 byte 를 2자리의 16진수로 나타내면 64 자리의 hash 를 얻어낼 수 있다. 이 중 앞의 10자리를 이용해서 62 진법으로 하는 7자리의 숫자로 나타내려고 한다.

```python
>>> 16 ** 9  # 10자리의 16진수 맨 앞 자리를 10진수로 변환한값
68719476736
>>> 62 ** 6  # 7자리의 62진수 맨 앞 자리를 62진수로 변환한값
56800235584
```

16 진법 변환의 10 자리는 맨 앞의 숫자가 0 인 경우에는 혹시나 다음 숫자가 작은 경우에 6자리로 축소될 가능성이 있다. 따라서 안정적으로, 7자리의 62 진법으로 나타내기 위해서 맨 앞 숫자가 0 인 경우는 다음 자리수부터 10자리의 숫자를 사용하도록 shifting 하는 메커니즘을 적용했다.

```go
if nArr[base] == 0 {
    return "", base, ErrorTryAgain
}
```

예를들면 `https://google.com` 의 16진법 변환 결과는 아래와 같다. 

```go
sha256.Sum256(32 byte)

[5 4 111 38 200 62 140 136 179 221 171 46 171 99 208 209 98 36 172 30 86 69 53 252 117 205 206 238 71 160 147 141]

sha256.Sum256(converted to length 64 hexa)

[0 5 0 4 6 15 2 6 12 8 3 14 8 12 8 8 11 3 13 13 10 11 2 14 10 11 6 3 13 0 13 1 6 2 2 4 10 12 1 14 5 6 4 5 3 5 15 12 7 5 12 13 12 14 14 14 4 7 10 0 9 3 8 13]
```

16진수로 표현된 64 자리의 맨 앞자리가 0이고 원래는 우측으로 시프팅 해야겠지만, 예시를 위해 진행해보면

```
10 자리의 16진수 : [0, 5, 0, 4, 6, 15, 2, 6, 12, 8] 

10진수 변환값: 21549229768

0 * 16 ** 9
+ 5 * 16 ** 8
+ 0 * 16 ** 7 
+ 4 * 16 ** 6
+ ... + 8 * 16 ** 0 = 21549229768

62진수 변환값: 0nwmn44
21549229768 // (62 ** 6) = 0                    # '0' 으로 변경됨
(21549229768 % (62 ** 6)) // (62 ** 5) = 23     # 'n' 으로 변경됨
...
0nwmn44
```

같은방법으로, 사실 맨 앞자리가 0 이었기 때문에, 우측으로 한칸 시프팅하면 같은 방식으로 계산하면 아래와 같다.

```
10 자리의 16진수 : [5, 0, 4, 6, 15, 2, 6, 12, 8, 3]

10진수 변환값: 344787676291

5 * 16 ** 9
+ 0 * 16 ** 8
+ 4 * 16 ** 7 
+ 6 * 16 ** 6
+ ... + 3 * 16 ** 0 = 344787676291

62진수 변환값: 64lLX35
344787676291 // (62 ** 6) = 6                    # '6' 으로 변경됨
(344787676291 % (62 ** 6)) // (62 ** 5) = 4      # '4' 으로 변경됨
...
64lLX35
```

물론 이대로 잘 동작할것이라고 생각은 했지만, 맨 앞자리 0인 경우 시프팅을 해도, 혹시 이전 데이터와 해시충돌이 일어날 수도 있다.
이 경우, 생성된 7자리의 해시가 이미 존재하는지 체크하고, 없는 경우에만 발급을 한다. 

정말 말도안되는 확률로, 한칸씩 시프팅하며 최대 64-10 회 반복 가능한데, 이 경우 모두 중복이 되었다고 한다면, 해시 충돌을 피할 수 없고, 에러를 반환하도록 하였다. 

## handler

본 서비스는 간단한 2개의 API 만 제공한다. 

- redirect: 제공된 단축url 이 가리키는 곳으로 redirect 하는 API
- generate: 새로운 단축url 을 생성하는 API

처음엔 서비스별로 별도의 스코프를 제공하고, crud 를 관리할 수 있는 서비스를 생각했었지만, 개발 범위를 단순화했다.

### redirect API

입력받은 linkID 값으로 redis 에 등록된 내용이 있는지 확인하고, 307(Temporary Redirect) 처리 하고 그 주소로 http redirect 를 한다. 없으면 400 bad request 처리 한다.

### generate API

위의 알고리즘을 이용해서, 입력받은 url 을 7 자리의 단축url 로 변환해서 return 한다. 위에서 설명한것처럼 말도안되는 확률로 54 연속 해시충돌이 일어났다면 실패를 반환한다. 이미 만들었던 경우(결과가 남아있는 경우)에는 이전 결과를 그대로 반환할 것이다. 위의 알고리즘 부분 설명을 보면, 같은 input 에 대해 같은 결과를 보장함을 알 수 있다.

# results

위에 기술한 대로 구현을 해서 사내 서비스로 제공을 했고, 이 글을 적는 시점까지는 서비스 하고 있다.

서비스를 좀 더 편하게 제공하고 싶은 마음에 chrome-extension 으로 제공했는데, 관련해서는 이후에 따로 정리해보려고 한다.

![url-shortner 생성 화면](/assets/img/posts/2021-03-19-url_shortener/url-shortener-generate.png)

![url-shortner 생성 화면](/assets/img/posts/2021-03-19-url_shortener/url-shortener-result.png)

- curl 결과: 307 과 함께 location 에 `https://www.google.com` 으로 redirect 한다.

```
> GET /fNgvnA6 HTTP/2
> Host: host.sample.abc
> User-Agent: curl/7.64.1
> Accept: */*
>
* Connection state changed (MAX_CONCURRENT_STREAMS == 128)!
< HTTP/2 307
< date: Fri, 19 Mar 2021 10:24:14 GMT
< content-type: text/html; charset=utf-8
< content-length: 59
< location: https://www.google.com/
```