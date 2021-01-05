---
layout: post
title: articles read
date: 2020-12-11 01:20:00 +0900
categories: [Wiki]
tags: [grpc, golang]
toc: true
---

> https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap-26c2462549a2/

- go 에서 GC 가 heap memory 할당의 2배(GOGC)가 되었을때 수행되는데, ballast(무게추) 역할을 하는 미사용할 메모리 할당을 미리 크게 해놓고, GC 를 적게 수행시키도록 한다. 
- ballast 를 크게 잡더라도, virtual memory 로 잡히고, 실제 해당 []byte 에 access 하기 전에는 real memory 할당이 적게 일어나서 걱정할 필요는 없다.
- 그 결과, GC 가 덜 일어나고, GC 가 일어났을때 발생하는 GC assist 가 덜 수행되어 (글에서는 GC 수행보다 GC assist 수행이 더 오버헤드가 큰것으로 되어있다) peak time 의 API latency 가 비약적으로 좋아졌고, GC ratio 도 많이 줄었다고 한다.