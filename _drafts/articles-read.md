---
layout: post
title: articles read
date: 2020-12-11 01:20:00 +0900
categories: [Wiki]
tags: [grpc, golang]
toc: true
---

## go memory ballast

link: [https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap-26c2462549a2/](https://blog.twitch.tv/en/2019/04/10/go-memory-ballast-how-i-learnt-to-stop-worrying-and-love-the-heap-26c2462549a2/)

- go 에서 GC 가 heap memory 할당의 2배(GOGC)가 되었을때 수행되는데, ballast(무게추) 역할을 하는 미사용할 메모리 할당을 미리 크게 해놓고, GC 를 적게 수행시키도록 한다. 
- ballast 를 크게 잡더라도, virtual memory 로 잡히고, 실제 해당 []byte 에 access 하기 전에는 real memory 할당이 적게 일어나서 걱정할 필요는 없다.
- 그 결과, GC 가 덜 일어나고, GC 가 일어났을때 발생하는 GC assist 가 덜 수행되어 (글에서는 GC 수행보다 GC assist 수행이 더 오버헤드가 큰것으로 되어있다) peak time 의 API latency 가 비약적으로 좋아졌고, GC ratio 도 많이 줄었다고 한다.

## go profiling by Dave Chenney

link: [https://www.youtube.com/watch?v=nok0aYiGiYA&ab_channel=GopherAcademy](https://www.youtube.com/watch?v=nok0aYiGiYA&ab_channel=GopherAcademy)

- Dave Chenney 의 발표영상
- go CpuProfile, MemProfile, Trace 를 사용한 fine-tune. 

## go 1.16 release 

link: [https://golang.org/doc/go1.16](https://golang.org/doc/go1.16)

이번에 1.15 -> 1.16 버전으로 변경되어, 문서를 읽어봤는데 개인적으로 주목할 부분은, ioutil 패키지가 비효율적인 부분이 많아서, io 와 os 로 나뉘어 들어갔다는 사실과, 신규 패키지인 embed, io/fs (filesystem) 패키지 이다.

## Don’t just check errors, handle them gracefully

[https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully](https://dave.cheney.net/2016/04/27/dont-just-check-errors-handle-them-gracefully)

David Cheney 의 error handle 에 관한 글. 이 글은 2016년의 글로 현재 go2 proposal 중에 error 개편이 예정되어 있어서, 공식으로 인정받진 못한 pkg(`pkg/errors`) 에 대한 이야기지만, 읽어볼 가치가 충분한 좋은 글이다.

필자는 Error handle 하는 단일 방식이 있었으면 했지만, 3 가지 핵심 전략으로 분류될 수 있다고 한다.

1. Sentinel error: `io.EOF` 와 같이 특정한 에러를 선언해놓고, 에러 처리 과정에서 직접 비교하는 방법이다.

다만, 필자는 비교하는 과정에서 import 가 생기고, 두 패키지 사이의 dependency 가 생길것이라 지양하라고 한다.

2. Error types

error 는 interface 인데, concrete type 을 만들어서 구현한다면, type assertion 을 통해서 error 를 판단해낼 수 있게 된다. 더 많은 컨텍스트를 전달하기 위해 에러 wrapping 이 가능하다는게 개선점이다. `os.PathError` 를 예로 들 수 있다.

그런데 결국 이 error 도 public 으로 export 되어야 하고, 결국 package 간 의존성을 만들기 때문에, 강한 커플링이 생겨 좋지 않다. 결국 필자는 또 쓰지 말라고 한다.

3. Opaque errors

이 방법이, 가장 caller 와의 커플링이 적어서 가장 유동적인 에러 핸들링 전략이라고 말한다.

caller 는, func 가 잘 돌았는지 아닌지만 확인할 수 있고, error 내부를 들여다볼 방법이 없다. 

필자는 재밌는 방법을 제안하는데, 아래와 같이 에러를 받아서 interface 를 구현하는지 체크하는데, 이는 type assertion 과는 다르게 다른 패키지와의 커플링을 만들지 않는다.
```golang
type temporary interface {
        Temporary() bool
}
 
// IsTemporary returns true if err is temporary.
func IsTemporary(err error) bool {
        te, ok := err.(temporary)
        return ok && te.Temporary()
}
```

```golang
if err != nil {
    return err
}
```
위와 같은 코드는 결국 상위 스코프까지 계속 err 를 전파해서, 나중에는 불충분한 내용을 전달하는 에러일 수 있다. 에러가 어디서 생겼는징에 대한 맥락이 너무 없어서, 코드 inspect 하는데 많은 시간이 걸린다.

도노반과 커니헨의 GOPL 책에서 `fmt.Errorf` 를 이용해서 맥락을 추가하라는 얘기를 하지만, error 를 string 으로 변환하고, 그걸 다시 error 로 만드는 방식은 sentinel 방식과 type assertion 이용하는 방식으로는 사용이 불가능하다. 

그래서 필자는 `pkg/errors` 를 만들어서 `Wrap` 과 `Cause` 2개의 (물론 다른 func 들도 많이 구현해놨다) func 를 제공하며 error 에 context 를 추가하는 방식으로 `fmt.Errorf` 와 비슷하지만, error wrapping 을 제안하고, `Cause` 를 이용해서 error context 중에 맨 처음 error(underlying cause) 가 밝혀질 것이다. 코드가 흥미로운데, `Cause()` 를 구현하지 않은 error 를 찾을때까지 for 를 반복해서 마지막 err 를 찾아낸다.

> https://github.com/pkg/errors/blob/614d223910a179a466c1767a985424175c39b465/errors.go#L155

```golang
func Cause(err error) error {
	type causer interface {
		Cause() error
	}

	for err != nil {
		cause, ok := err.(causer)
		if !ok {
			break
		}
		err = cause.Cause()
	}
	return err
}
```

## inlining optimisations in go

go inlining 이 어떻게 동작하는지에 대한 예시이다.

[https://dave.cheney.net/2020/04/25/inlining-optimisations-in-go](https://dave.cheney.net/2020/04/25/inlining-optimisations-in-go)

## go hidden pragmas

`//go:noinline` 과 같이, go compiler 에게 지시를 내리는 pragmas 에 대한 내용이다.
코드 작성중에 직접 사용할 일은 잘 없겠지만, 흥미로운 내용이다.

[https://dave.cheney.net/2018/01/08/gos-hidden-pragmas](https://dave.cheney.net/2018/01/08/gos-hidden-pragmas)

## If a map isn’t a reference variable, what is it?

channel, map 은 runtime 에 정의된 structure 의 pointer 이고, slice 는 reference 이다.

> A map value is a pointer to a runtime.hmap structure.

[https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)


## pointers in go

[https://dave.cheney.net/2014/03/17/pointers-in-go](https://dave.cheney.net/2014/03/17/pointers-in-go)



[https://dave.cheney.net/2016/03/19/should-methods-be-declared-on-t-or-t](https://dave.cheney.net/2016/03/19/should-methods-be-declared-on-t-or-t)