---
layout: post
title: The Go Programming Language by 앨런 도노반, 브라이언 커니건
date: 2021-02-03 19:17:00 +0900
categories: [Book]
tags: [golang]
toc: true
comments: true
---

- 너무나 좋은 내용이 많고, 얻은게 많은 책이라 기록을 남기기로 했다.
- 그동안 읽은 golang 책들 중 최고.

# 정리

## defer 170p

- `defer` 는 `func()` 를 우측에 명시해야되는데, closure 처럼 func return 으로 trace 를 간편히 표현할 수 있었다.

```golang
func bigSlowOperation(){
    defer trace("bigSlowOperation")() // () 를 뒤에 붙여, 일단 trace() 실행 하고, 
                                      // return 된 함수를 defer 한다.
    ...
}

func trace(msg string) func() {
    start := time.Now()
    log.Printf("enter %s", msg)
    return func() { log.Printf("exist %s (%s)", msg, time.Since(start)) }
}
```

## pointer receiver 를 implicit 하게 처리함 183p

- ScaleBy 와 Distance 가 각각 *T, T 를 receiver 로 갖는다고 가정한다.

```golang
func (p *Point) ScaleBy(factor float64){
    ...
}

func (p Point) Distance() float64{
    ...
}
```

```golang
하지만 Point.Distance 와 같은 Point 메소드는 주소에서 값을 얻을 수 있으므로 *Point 수신자로 호출할 수 있다.
그냥 수신자가 가리키는 값을 읽으면 된다. 컴파일러가 묵시적으로 *를 삽입해준다. 다음 두 함수의 호출은 같다.

pptr.Distance(q)
(*pptr).Distance(q)
Point{1, 2}.Distance(q) // Point
pptr.ScaleBy(2)         // *Point
p.ScaleBy(2)            // implicit (&p)
pptr.Distance(q)        // implicit (*pptr)

명명된 타입 T 에서 모든 메소드의 수신자가 (*T가 아닌) T 타입이라면, 해당 타입의 인스턴스는 안전하게 복사할 수 있다.
어떤 메소드를 호출할때도 반드시 복사가 일어난다(time.Duration).

하지만 포인터 수신자가 하나라도 있다면 내부 불변성을 깨뜨릴 수 있으므로 T 의 인스턴스 복사는 피해야한다.
예를들면 bytes.Buffer 의 인스턴스를 복사하면 원본과 복사본이 동일한 내부 바이트 배열에 대한 별칭이 된다. 이후의 메소드 호출 결과는 예측할 수 없다.
```

## memoization 을 이용한 cache 구현체 302p

- 책에서 제공하는 cache 구현체 코드가 대단해서 아래와 같이 전체 내용을 복사붙여넣기 하였다.

- 언젠가 아래의 코드 처럼 시스템 설계가 필요할 것 같다는 생각이 들었다.

- `memo.New` 는 `Func` 인 `f` 를 입력받는데, 이 `Func` 는 `memo.requests` 에 `memo.Get`을 통해 요청된 value 를 처리하고, 반복된 요청은 memoization 을 통해 return 하는 비동기 캐시 기능을 구현한 코드이다.

- `memo` 는 member 로 `request` 의 channel 을 갖고있는데, 특이한건, 요청 key 와 response 받을 channel 까지 포함한 struct 이다.

- `cache` 는 `memo` struct 에 포함되지 않고, `server` func 에 map 으로 선언했고, concurrent 하게 access 되지 않을것이라, thread safe 하다. 

- `Get` 으로 받은 요청을 `memo.requests` 에 담고, `request` 에 만들어 보낸 `response` channel 을 listen 하고, 이 response 는 `server` 에서 `entry` 를 처리해서 보내기 때문에, 문제가 생기지 않는다.

- channel close 원칙중에는, channel 생성자가 책임지고 닫는다가 있는데, 명시적이진 않지만, `entry` 가 생성된 `server` func 의 for loop 내부에서 호출한 `e.call` 에서 close 하므로, 중복으로 닫히지 않는다.

- `entry` 의 `call`, `deliver` step 을 구분하고, `request` 에 response 받을 `response` 를 포함시켜 보내는 설계가 훌륭하다. (생각해보니, `http.Request` struct 도 `http.Response` 를 내부에 가지고있다.)

> reference: https://github.com/adonovan/gopl.io/blob/master/ch9/memo5/memo.go

```golang
// Copyright © 2016 Alan A. A. Donovan & Brian W. Kernighan.
// License: https://creativecommons.org/licenses/by-nc-sa/4.0/

// See page 278.

// Package memo provides a concurrency-safe non-blocking memoization
// of a function.  Requests for different keys proceed in parallel.
// Concurrent requests for the same key block until the first completes.
// This implementation uses a monitor goroutine.
package memo

// Func is the type of the function to memoize.
type Func func(key string) (interface{}, error)

// A result is the result of calling a Func.
type result struct {
	value interface{}
	err   error
}

type entry struct {
	res   result
	ready chan struct{} // closed when res is ready
}

// A request is a message requesting that the Func be applied to key.
type request struct {
	key      string
	response chan<- result // the client wants a single result
}

type Memo struct{ requests chan request }

// New returns a memoization of f.  Clients must subsequently call Close.
func New(f Func) *Memo {
	memo := &Memo{requests: make(chan request)}
	go memo.server(f)
	return memo
}

func (memo *Memo) Get(key string) (interface{}, error) {
	response := make(chan result)
	memo.requests <- request{key, response}
	res := <-response
	return res.value, res.err
}

func (memo *Memo) Close() { close(memo.requests) }

func (memo *Memo) server(f Func) {
	cache := make(map[string]*entry)
	for req := range memo.requests {
		e := cache[req.key]
		if e == nil {
			// This is the first request for this key.
			e = &entry{ready: make(chan struct{})}
			cache[req.key] = e
			go e.call(f, req.key) // call f(key)
		}
		go e.deliver(req.response)
	}
}

func (e *entry) call(f Func, key string) {
	// Evaluate the function.
	e.res.value, e.res.err = f(key)
	// Broadcast the ready condition.
	close(e.ready)
}

func (e *entry) deliver(response chan<- result) {
	// Wait for the ready condition.
	<-e.ready
	// Send the result to the client.
	response <- e.res
}
```

## 고루틴과 스레드 304p

- 원문의 내용을 정리하면, 다음과 같다. 하지만, 워낙 좋은 내용이라 원문 그대로 옮겼다. 

### 가변스택 304p

- go 프로그램에선 고루틴을 많이 생성하는데, 고정 크기인 스레드(2MB)를 이렇게 많이 생성하기에는 부담일것이다. 
- 고루틴은 2KB 인 작은 가변스택으로 시작해서, 1GB 까지도 커질 수 있다.

- 각 OS 의 스레드는 고정 크기의 메모리 블록(보통 최대 2MB)을 현재 진행 중인 함수 호출의 지역 변수를 저장하거나 다른 함수가 호출될 때 일시적으로 중지시키기 위한 작업 영역인 스택으로 사용한다. 이러한 고정 크기의 스택은 너무 많으며 동시에 너무 적다. 2MB 스택은 단지 WaitGroup 을 대기하다가 채널을 닫는 것과 같은 작은 고루틴에서는 메모리의 엄청난 낭비다. Go 프로그램에서 한 번에 수백 또는 수천 개의 고루틴을 생성하는 것은 드문 일이 아니지만, 이 정도 크기의 스택으로는 불가능할 것이다. 또한 고정 크기 스택은 그 크기에도 불구하고 대부분의 복잡하고 깊은 재귀 함수에서는 충분치 않다. 고정된 크기를 변경하면 공간 효율성을 개선해 더 많은 스레드를 생성하거나 더 깊은 재귀 함수를 사용할 수 있지만, 고정 크기 스택에서는 두 가지 모두 불가능하다.

- 반면에 고루틴은 일반적으로 2KB 인 작은 스택으로 시작한다. 고루틴의 스택은 OS 스레드 스택과 마찬가지로 활성화된 지역 변수와 일시적으로 중지된 함수 호출을 저장하지만, OS 스레드와는 달리 고루틴의 스택은 고정 크기가 아니다. 즉 필요한 만큼 늘어나고 줄어든다. 고루틴 스택의 최대 크기는 일반적인 고정 크기 스레드 스택보다 수배나 더 큰 1GB 가 될 수도 있지만, 이 정도 크기의 스택을 사용하는 고루틴은 많지 않다.

### 고루틴 스케줄링

- OS 스레드는 OS 커널에 의해 스케줄된다. 매 밀리초 마다 하드웨어 타이머가 프로세서를 인터셉트해 커널 함수 scheduler 가 호출되게 한다. 이 기능은 현재 실행중인 스레드를 일시적으로 중단하고 메모리의 레지스터를 저장한 후 스레드 목록을 조회해 다음에 수행할 스레드를 결정하고, 메모리에서 해당 스레드의 레지스터를 복원한 후 복원된 스레드의 수행을 재개한다.

- OS 스레드는 커널에 의해 스케줄링되므로 한 스레드에서 다른 스레드로 제어를 넘기려면 한 사용자 스레드의 상태를 메모리에 저장하고, 다른 스레드의 상태를 복원한 후 스케줄러의 자료 구조를 갱신하는 전체 컨텍스트 전환이 필요하다. 이 동작은 지역성의 부족과 필요한 메모리 접근 횟수로 인해 느리며, 메모리에 접근하기 위해 필요한 CPU 사이클의 수가 증가함에 따라 점점 더 느려지고 있다.

- Go 런타임은 n 개의 OS 스레드에 있는 m 개의 고루틴을 다중화(또는 스케줄링) 하는 m:n 스케줄 링 기법의 자체 스케줄러를 포함하고 있다. Go 스케줄러의 역할은 커널 스케줄러와 유사하지만 단일 Go 프로그램의 고루틴에 국한된다.

- Go 스케줄러는 운영체제의 스레드 스케줄러와는 다르게 하드웨어 타이머에 의해 주기적으로 호출되지 않고 특정한 Go 언어의 기반에 의해 묵시적으로 호출된다. 예를 들어 고루틴이 time.Sleep 을 호출하거나 채널 또는 뮤텍스 작업을 대기할 때는 스케줄러가 해당 고루틴을 슬립 상태로 만들고, 이후 깨워야 할 때까지 다른 고루틴을 실행한다. 이때는 커널 컨텍스트 전환이 필요하지 않으므로 고루틴의 스케줄 재조정이 스레드 스케줄을 재조정 하는 것보다 훨씬 비용이 적게든다.

### GOMAXPROCS 

- GOMAXPROCS 파라미터로 동시에 얼마나 많은 OS 스레드에서 Go 코드를 수행할지 결정함

- 기본값은 시스템의 CPU 개수(m:n 의 n)

- 슬립 상태이거나 통신을 대기하고 있는 고루틴에서는 스레드가 필요하지 않고, I/O 또는 다른 시스템 호출이나 Go 가 아닌 함수를 호출해 대기하고 있는 고루틴들에는 OS 스레드가 필요하지만 GOMAXPROCS 에 포함할 필요는 없다.

- runtime.GOMAXPROCS 함수를 통해 명시적으로 지정 가능하다.
- goroutine 이 최대 1개인 경우에, 1 출력하는 main 실행, 일정시간 뒤 go 스케줄러가 이 고루틴을 슬립으로 설정, 0을 출력하는 고루틴을 깨워서 OS 스레드에서 실행되게함.
- 2 인 경우에는, 두 고루틴이 동시에 실행되어 같은 비율로 숫자가 출력됨

```golang
for {
    go fmt.Print(0)
    fmt.Print(1)
}

// $ GOMAXPROCS=1 go run main.go
// 11111111110000000000111111111100000000
// $ GOMAXPROCS=2 go run main.go
// 01010101010101
```

## 내부 패키지 321p

- go pkg 중에 `internal` 이라는 명칭을 별 뜻 없이 쓰는줄 알았는데, 내부 패키지를 의미한다.

- 큰 패키지를 관리 가능한 작은 부분들로 분할할 때는 각 부분 간의 인터페이스를 다른 패키지에 공개하고 싶지 않을 수도 있다. 이럴때 내부 패키지를 이용한다.

- 내부 패키지는 `internal` 디렉토리의 상위 디렉토리 안에 있는 다른 패키지에서만 임포트 할 수 있다. 

- `net/http/internal/chunked` 는 `net/http/httputil` 이나 `net/http` 에서는 임포트 할 수 있지만, `net/url` 에서는 임포트 할 수 없다.


## 구조체 크기 관련 379p

- 구조체의 필드 타입들이 서로 다른 크기일 때는 필드를 가능한 한 밀접한 순서로 정의하는 것이 좀 더 공간 효율적일 수 있다.

- 아래의 첫번째 구조체는 다른 두 구조체에 비해 최대 50%의 메모리가 더 필요하다.

```
struct{ bool; float64; int16 } : 64bit(3 words) : 32bit(4 words)
struct{ float64; int16; bool } : 64bit(2 words) : 32bit(3 words)
struct{ bool; int16; float64 } : 64bit(2 words) : 32bit(3 words)
```
- `unsafe.Sizeof`: 구조체의 모든 피연산자를 바이트로 표현한 크기
- `unsafe.Alignof`: 인자 타입에 필요한 정렬방식을 알려줌
- `unsafe.Offsetof`: 바깥쪽 구조체 x의 시작 위치에서 상대적인 필드 f의 오프셋을 계산하며, 홀이 있다면 홀도 표현한다.