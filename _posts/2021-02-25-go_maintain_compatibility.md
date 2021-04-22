---
layout: post
title: go api 호환성(compatibility)을 유지하기 위한 방법들
date: 2021-02-25 15:38:00 +0900
categories: [Wiki]
tags: [golang]
toc: true
comments: true
---

# introduction 

golang 은 backward compatibility 를 상당히 신경을 많이쓴다. 덕분에 버전업이 되어도 새로운 버전을 걱정없이 사용할 수 있다. 하지만, 시간이 흐르며 api 에는 더 많은 기능들이 들어갈 것이고, struct 혹은 func 가 확장되는건 당연하다. 물론 과감한 refactor 를 할 수도 있겠다. 

설계 단계부터 api compatibility 를 잘 고려하는 유용하고, 생각할만한 내용들을 다룬 좋은 [글들](#references) 을 읽고, 관련 내용과 생각을 정리하고, 앞으로 개발할때 다시 꺼내보고 참고하려는 목적을 가진 글이다.

# 접근방식

## 1. func signature 를 바꾸지 않고, 다른 함수를 추가한다

보통, func 에 기능이 추가된다면 아래와 같이, param 을 추가할거고, 이를 빈값이 와도 문제없도록 ... 를 이용하게 될텐데, func 의 signature 이 바뀌었기 때문에, 이는 좋은 방향이 아니다.

```go
func Run(name string)

func Run(name string, size ...int)

package mypkg
var runner func(string) = yourpkg.Run
```

golang 1.7 부터 `context` pkg 가 제공되기부터, api 의 첫번째 param 은 `ctx context.Context` 를 전달하는게 패턴처럼 되어버렸다. Api 의 cancel 관리를 context 를 이용하면 간편하기 때문인데, 대다수의 api 들과는 다르게, 자주 쓰는 패키지인 `net/http` 에서는 ctx 를 `http.Request` 를 통해서 간접 전달하고있는게 특이했다. 

알고보니 go 에서는 필요한 func 에 context 를 직접 넘기는것을 권장하여, api 사용자가 직관적으로 이해할 수 있기를 바라고, 이에 반하는 struct 에 context 를 포함시키는것은 확실한 경우가 아니면 지양하라는 입장이다. 예를들면, context 의 전달 시점이 struct 의 New 시점인지 명시하는건 복잡한 일이기도 하고, 각기 다른 method 에서 같은 context 의 영향을 받는걸 원하지 않을수도 있다.

> Contexts should not be stored inside a struct type, but instead passed to each function that needs it.

위에서 말했던 `http.Request` 는 이전에 api 들의 backward compatibility 를 보장하기 위해서 어쩔수없이 취한 방법이라는 것인데, request 를 생성하는 `http.NewRequest` func 를 보면 확실하게 알 수 있다. 언젠가부터(아마도 1.7 부터일것이라고 예상한다), `http.NewRequest` 는 `http.NewRequestWithContext` 의 helper func 처럼 동작하고 있었다. 습관처럼 `http.NewRequest` 로 생성한 request 의 context 를 deadline 을 설정한 context 로 `req.WithContext` 를 이용해서 적용하고 있었는데, 앞으로는 `http.NewRequestWithContext` 를 쓰면 될것이다. 이 부분은 사용자들에게 backward compatibility 를 보장하는 동시에, 각자의 스케쥴에 맞게 api 변경에 대응할 수 있게하는 좋은 방식이다.

```go
// net/http/request.go
// NewRequest wraps NewRequestWithContext using the background context.
func NewRequest(method, url string, body io.Reader) (*Request, error) {
	return NewRequestWithContext(context.Background(), method, url, body)
}
```

## 2. self referential 

struct 를 선언할때, config 를 전달해야 된다면, 라이브러리 들에서 config 를 struct 로 정의해서 그대로 넘기는것 보다는, gRPC 의 예처럼, self-referential 방식이 많이 보인다. config 를 정의해서 넘기는것도 직관적이지만, 정의해서 쓰는것보다, 안하는게 훨씬 많아, zero value 들이 넘처나는 모습이 보기 안좋기도 하다.

```go
grpc.Dial("some-target",
  grpc.WithAuthority("some-authority"),
  grpc.WithMaxDelay(time.Second),
  grpc.WithBlock())
```

gRPC 를 사용하면, 아래와 같이 Dial 을 생성하는것이라면, `DialOption` 라는 interface 를 선언해놓고, 이 interface 는 `apply` 만 갖고있는데, 특이한건 `*dialOptions` 는 `dialOption` 에 대한 모든 정보를 갖고있는 struct 이다. 예를들어 `WithBlock()` 을 보면 block 값을 true 로 바꾸는 func 인데, `newFuncDialOption` 은 결국 apply 로 변경 해서 주는, adpater 이다.

```go
// code from grpc/server.go

// DialOption configures how we set up the connection.
type DialOption interface {
	apply(*dialOptions)
}

// WithBlock returns a DialOption which makes caller of Dial blocks until the
// underlying connection is up. Without this, Dial returns immediately and
// connecting the server happens in background.
func WithBlock() DialOption {
	return newFuncDialOption(func(o *dialOptions) {
		o.block = true
	})
}

// funcDialOption wraps a function that modifies dialOptions into an
// implementation of the DialOption interface.
type funcDialOption struct {
	f func(*dialOptions)
}

func (fdo *funcDialOption) apply(do *dialOptions) {
	fdo.f(do)
}

func newFuncDialOption(f func(*dialOptions)) *funcDialOption {
	return &funcDialOption{
		f: f,
	}
}
```

이는 `http` 의 `HandlerFunc` 로 생각하면 될 것 같다. `ServeHTTP` 를 구현해야 `http.Handler` 가 될 수 있기 때문에, 원형이 같은 func 를 정의하면 자연스럽게 `ServeHTTP` 를 정의해서 연결해주는 모습이다.

```go
// code from net/http/server.go

type HandlerFunc func(ResponseWriter, *Request)

// ServeHTTP calls f(w, r).
func (f HandlerFunc) ServeHTTP(w ResponseWriter, r *Request) {
	f(w, r)
}
```

## 3. interface 를 추가 정의한다.

interface 사용하던 부분을 그대로 수정하기에는, 영향도가 클 수 있기 때문에, 그 interface 을 포함하는, 추가 기능이 정의된 interface 로 assertion 해서 변경된 로직을 적용하는 방법이 있다고 한다. 아래와 같이 `Reader` 중, `io.Seeker` 를 추가로 구현하는 경우에 더 효율적인 알고리즘을 택할 수 있도록 되어있고, 혹시 `Seeker` 를 구현하지 않았다고 해도, 기존의 로직을 타면 되기 때문에 문제 생길것이 없다. 

```go
// code from archive/tar/reader.go 
package tar

type Reader struct {
  r io.Reader
}

func NewReader(r io.Reader) *Reader {
  return &Reader{r: r}
}

func (r *Reader) Read(b []byte) (int, error) {
  if rs, ok := r.r.(io.Seeker); ok {
    // Use more efficient rs.Seek.
  }
  // Use less efficient r.r.Read.
}
```

`net/http` 의 `Transport` 의 `RoundTrip` 에서도 이와 비슷하게, interface 를 이용한건 아니지만, backward compatibility 를 유지하기 위해, `alternateRoundTripper` 인지 아닌지 미리 체크하고, 맞는 경우에 신규 로직으로, 지원하지 않는 `RoundTripper` 인 경우에는 기존 로직을 타도록 한 부분이 있다.

```go
// code from net/http/transport.go
if altRT := t.alternateRoundTripper(req); altRT != nil {
    if resp, err := altRT.RoundTrip(req); err != ErrSkipAltProtocol {
        return resp, err
    }
    var err error
    req, err = rewindBody(req)
    if err != nil {
        return nil, err
    }
}
```

## 4. 흥미로웠던 부분 (compatibility 유지를 위해 api 에 제약을 미리 걸어두는 방법)

개발하는데 interface 가 필요하지만, 사용자들이 마음대로 인터페이스를 정의하지 못하도록, 인터페이스 내부에 unexported func 를 끼워넣는 방법이 있다. 실제로, `testing` pkg 에서 사용하고 있다고 한다.

```go
type TB interface {
    Error(args ...interface{})
    Errorf(format string, args ...interface{})
    // ...

    // A private method to prevent users implementing the
    // interface and so future additions to it will not
    // violate Go 1 compatibility.
    private()
}
```

추가로, struct 를 비교하지못하고, pointer 끼리만 비교되게 하려면 아래와 같이, struct 에 비교불가능한 필드를 넣는 방법이 있다. 아래의 `doNotCompare` 는 unexported 이기 때문에, 사용자가 지정도 불가능하고, 공간도 차지하지 않는다. 대신, map 의 key 로 이용할 수 있는건 compare 가능한것 뿐이니, `Point` 는 map 의 key 로는 쓰지 못한다.

```go
type doNotCompare [0]func()

type Point struct {
        doNotCompare
        X int
        Y int
}
```

# references

> module compatibility 를 유지하는 방법들을 소개하는 글, 본 글에서 가장 많이 참고한 글이다.

- https://blog.golang.org/module-compatibility

> context 는 struct 내에 포함하지 말고, func 에 직접 전달하는것이 좋다는 내용의 글

- https://blog.golang.org/context-and-structs

> struct 생성에 self-referential func 를 이용한 확장성 높은 설계 by Rob Pike

- https://commandcenter.blogspot.com/2014/01/self-referential-functions-and-design.html