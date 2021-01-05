---
layout: post
title: 단축url 제작
date: 2020-12-29 11:40:00 +0900
categories: [Project]
tags: [url-shortener, golang]
toc: true
---

# introduction
- TMI: 예전에 kakaopay 개발자 채용때 단축 url 제작 과제를 받았다가 떨어진 적이 있다.
> 솔직히 탈락이 당연했다.

- 외부 공유용은 아니고, 사내용으로 단축 url 을 만들어서 사용하고싶은 생각과, 연말의 여유로움이 개발을 하게 이끌었다.

# progress
## architecture
- 구조는 단순하게 api 서버, DB 로 구성했고, DB 는 redis 를 사용했다.
- redis 를 사용한 이유는, 빠른 key 조회와, 자체 expire 기능이 있기 때문이다. 추가로 저장하려는 데이터 value 가 적어서, 가볍게 사용할 의도이다.

## algorithm
- 다른 서비스들을 보니, 일단 62 자리(`[0-9a-zA-Z]`) 기반으로 개발하는게 대세였고, 자릿수는 변동이 있었다.

- 변환과정: 단축하려는 url(:`url`) -> (hash) -> 단축된 형태로 변경(:`urlKey`)
- 저장과정: `urlKey` 를 redis 의 key 로 하고, value 에 `url` 을 저장하는 방식

- hash 는 간단하게 `sha256` 을 사용했다.
```golang
sum := sha256.Sum256([]byte(url))
```
- sha256