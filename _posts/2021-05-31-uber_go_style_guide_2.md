---
layout: post
title: Uber go style guide (2)
date: 2021-05-31 23:08:00 +0900
categories: [Wiki]
tags: [golang, style-guide]
toc: true
comments: true
---

# introduction

팀 내에서 서로의 코드리뷰 퀄리티와 코드 생산성 향상을 위해 Uber 가 정리한 golang style guide 를 함께 읽어보는 시간을 갖기로 했다. 

아래 내용들은, 읽으면서 몰랐던 부분이나, 정리해두고 싶은 내용들을 다룬다.

아래의 코드와 내용들은 [https://github.com/uber-go/guide/blob/master/style.md](https://github.com/uber-go/guide/blob/master/style.md) 문서에서 발췌한 내용이 대부분임을 미리 밝힌다.

Performance 항목부터의 내용을 다룬다.

[Uber go style guide (1) 바로가기](https://fransoaardi.github.io/posts/uber_go_style_guide_1/)

# contents(Performance)

## Prefer strconv over fmt

`fmt.Sprint` 보다는 `strconv` 를 쓰는것이 낫다고 한다.

## Avoid string-to-byte conversion

아래의 코드는 반복적으로 []byte 를 할당하는가보다, 따라서 for 밖에 미리 할당해놓으라고 한다.
```golang
// Bad
for i := 0; i < b.N; i++ {
  w.Write([]byte("Hello world"))
}

// Good 
data := []byte("Hello world")
for i := 0; i < b.N; i++ {
  w.Write(data)
}
```

## Specifying Map Capacity Hints

이부분은 조금 신기했는데, `make(map[T1]T2, hint)` 가 지정한 capacity allocation 이 일어날거라 생각했는데, 아래 내용을 보면, slice 와는 달리 선점형 할당이 일어날것을 보장하진 않고, 필요한 할당을 근사하는데 사용한다고 한다. 

> Note that, unlike slices, map capacity hints do not guarantee complete, preemptive allocation, but are used to approximate the number of hashmap buckets required. Consequently, allocations may still occur when adding elements to the map, even up to the specified capacity.

## Specifying Slice Capacity

`make([]T, length, capacity)` 는 위의 map 과는 달리 hint 가 아니라 실제 capacity 를 할당한다고 한다. 이전에 [실험](https://fransoaardi.github.io/posts/go_append/)해봤던 적도 있는데, slice 의 capacity 가 1024 까지 2배수씩 늘어나며, 계속해서 copy 가 일어나서 비효율적인 동작을 하는것을 확인한 적이 있다.

# contents(Style)

## Be Consistent

일관성을 가지라는 얘긴데 일관적인 코드는 유지보수가 쉽고, 쉽게 이해할 수 있고, 새로운 컨벤션이나 버그 수정에 있어서 쉽게 변경할 수 있다.
반대로, 여러 충돌하는 코드 스타일을 갖고있는 경우에 유지보수가 힘들어지고, 코드 리뷰 하기 힘들고 등등 안좋은말을 많이 써놨다.

## Group Similar Declarations

비슷한 상수, 변수, 타입선언은 묶어서 선언하라고 한다. 반대로 말하면 상관 없는 변수는 묶어서 선언하지 않는다.

## Import Group Ordering

standard library 와 나머지것들 이렇게 두 그룹으로 나눠서 import 하라고 한다. 나머지 중에서도 사내 깃헙에 있는 패키지들을 또 한번 구분해왔었는데 생각보다 이점이 없는것 같아서 여기서 제안한것처럼 해봐도 괜찮을것같다.

## Package Names

package 이름을 정할때는 
- 모두 소문자를 쓰고, 대문자랑 `_` 를 사용하지 않는다.
- `renamed` 될 필요 없다. (정확히 이해를 하지 못했으나, `db.DB` 이런거 하지 말라는 의미로 이해했다)
- 짧고 집약적인 이름을 쓴다.
- 복수형은 사용하지 않는다
- `common`, `util`, `shared`, `lib` 은 전혀 정보를 주지 않는 안좋은 명명이다.

## Function Names
MixedCaps 방식을 쓰는데, test function 은 예외적으로 `_` 를 포함할 수 있다. 예를들어 `TestMyFunction_WhatIsBeingTested` 와 같은 test function 은 가능하다.

## Import Aliasing
package 명이 import path 의 끝부분과 다르면 무조건 aliasing 을 해야된다. 

예를들면 아래와 같다.
```golang
import (
  "net/http"

  client "example.com/client-go"
  trace "example.com/trace/v2"
)
```

import 한 package 이름이 conflict 나는 경우에만 import alias 를 사용한다. 예를들면 아래와 같다.
```golang
import (
  "fmt"
  "os"
  "runtime/trace"

  nettrace "golang.net/x/trace"
)
```

## Function Grouping and Ordering

- function 들은 호출 순서에 비슷하게 정렬되어야 된다.
- function 들은 receiver 로 묶여야된다.

따라서, `struct`, `const`, `var` 이후에 functions 들을 배치한다.

`New` 는 type 정의 이후에, receiver function 들보다는 먼저 등장한다. 

평범한 유틸성 함수들은 파일 끝부분에 위치한다.

아래의 예시를 잘 봐두는게 좋을것같다. 

```golang
// Bad
func (s *something) Cost() {
  return calcCost(s.weights)
}

type something struct{ ... }

func calcCost(n []int) int {...}

func (s *something) Stop() {...}

func newSomething() *something {
    return &something{}
}

// Good
type something struct{ ... }

func newSomething() *something {
    return &something{}
}

func (s *something) Cost() {
  return calcCost(s.weights)
}

func (s *something) Stop() {...}

func calcCost(n []int) int {...}
```

## Reduce Nesting

error 나 특별한 조건을 먼저 handle 해서 빠르게 return 하는 방식으로 nesting 을 줄인다. 

## Unnecessary Else

굳이 불필요한 else 쓰지 말고, if 한번만 쓰라고 한다.
```golang
a := 10
if b {
  a = 100
}
```

## Top-level Variable Declarations

- `var` 에 굳이 type 을 명시할 필요는 없다
```golang
var _s = F()
// Since F already states that it returns a string, we don't need to specify the type again.

func F() string { return "A" }
```

- 원하던 type 과 일치하지 않는 경우에만 type 을 명시하라
아래 예제의 내용이 신기했던건, `myError` 를 return 하는 function 의 결과를 error 로 정의한 것을 볼 수 있다.

```golang
type myError struct{}
func (myError) Error() string { return "error" }
func F() myError { return myError{} }

var _e error = F()
// F returns an object of type myError but we want error.
```

## Prefix Unexported Globals with _

unexport 된 top-level var, const 는 `_` 로 시작하게 하여 global 인것을 명확하게 하라고 하는데, 이건 다른 언어들에서 시작했던 컨벤션으로 알고있는데, 논란의 여지가 있어보이긴 한다.

예외적으로 unexported error 는 `err` 로 시작하라고 한다.

## Embedding in Structs

mutex 와 같은 embedded type 은 struct 의 맨 위에 두고 라인피드로 일반 필드들과 구분한다.

embedding 을 쓸거면 정말 가시적인 이점이 있을때만 쓸것을 권고하고 있는데, 위에서는 embedding 하더라도 직접 function 구현하라는 식으로 얘기했었다.

Embedding 이 주의해야될것들은 아래와 같다
- 미적이거나 편하자고 쓰는건 안되고
- 바깥타입의 생성/사용을 더 힘들게 해선 안되고
- 바깥타입이 의미있는 zero value 를 갖고있을때, 임베딩 이후에도 의미있는 바깥타입을 갖고있어야 한다.
- embedding 하면서 바깥 타입의 불필요한 함수나 필드들을 노출하게 되면 안된다.
- 바깥 타입의 copy 에 영향주면 안된다.
- 외부 타입의 API 나 semantics 를 변경하면 안된다.
- 정식적이지 않은 내부 타입을 임베드 하면 안된다.
- 바깥 타입의 구현 상세를 노출하면 안된다.
- 사용자가 내부 타입을 확인하거나 컨트롤 할 수있게 하면 안된다.
- 사용자가 예상하지 못한 방식으로 임베딩 하는것

제안하는 예제들을 아래에 첨부한다.
아래 예제는 WriteCloser 를 그대로 활용하며 count 만 추가하여, embed 해도 된다고 한다.
```golang
type countingWriteCloser struct {
    // Good: Write() is provided at this
    //       outer layer for a specific
    //       purpose, and delegates work
    //       to the inner type's Write().
    io.WriteCloser

    count int
}

func (w *countingWriteCloser) Write(bs []byte) (int, error) {
    w.count += len(bs)
    return w.WriteCloser.Write(bs)
}
```

만약에 `bytes.Buffer` 가 아닌 `io.ReadWriter` 를 embed 하고 있었으면, zero value 에 대해 `Read`, `Write` 등이 nil reference 로 panic 이 발생할 수 있으므로, zero value 를 잘 감안하라고 한다.

```golang
type Book struct {
    // Good: has useful zero value
    bytes.Buffer
    // Bad: pointer changes zero value usefulness
    // io.ReadWriter
}

// later

var b Book
b.Read(...)  // ok
b.String()   // ok
b.Write(...) // ok
```

## Local Variable Declarations

명시적으로 값을 할당하는 경우에 `var s = "foo"` 대신 `s := "foo"` 를 사용한다.

empty slice 를 선언할때는 `s := []int{}` 보다 `var s []int` 가 낫다.

## nil is a valid slice

`nil` 은 length 가 0인 유효한 slice 이므로 아래와 같이 nil 을 return 하는게 맞다.

```golang
// Bad
if x == "" {
  return []int{}
}
// Good
if x == "" {
  return nil
}
```

빈 slice 인지 체크하기 위해서는 `s==nil` 이 아닌 len 을 이용해서 비교하라. 

```golang
// Bad
func isEmpty(s []string) bool {
  return s == nil
}

// Good
func isEmpty(s []string) bool {
  return len(s) == 0
}
```

`var nums []int` 와 같이 선언해도 바로 append 가 가능하다.

`nil` 은 valid slice 이지만, length 0 인 할당된 slice 와는 다르게 동작한다. 

테스트를 좀 해봤는데 솔직히 `var x []int` 선언 후 `x = nil` 이후에 append 가 동작할것이라고 생각하진 않았는데, 곰곰이 생각해보니 type 이 할당된 이후에 nil 할당은 이상한 일이 아니다. 

```golang
package main

import (
	"fmt"
)

func main() {
	var x []int
	x = nil
	x = append(x, 1)
	x = nil
	x = append(x, 2)
	fmt.Println(x)
}
```

## Reduce Scope of Variables
err 만 return 받는 경우에 if scope 안으로 넣는 방법을 얘기하는데, 그때그때 적절하게 하면 될것같다.


## Avoid Naked Parameters

가독성을 위해서 c-style 주석으로 parameter name 을 전달해주라고 하는데, IDE 에서 이미 제공하고 있는 기능이라, 따로 주석까지 달아줘야되는지 잘 이해가 되진 않는다. 

boolean 을 그대로 사용하기 보다는, custom type 을 선언해서 가독성을 높고 type safe 한 코드를 작성한다. 이후에 true/false 외의 추가 상태를 정의하는것도 가능해서 좋다.

```golang
type Region int

const (
  UnknownRegion Region = iota
  Local
)

type Status int

const (
  StatusReady Status = iota + 1
  StatusDone
  // Maybe we will have a StatusInProgress in the future.
)

func printInfo(name string, region Region, status Status)
```

## Use Raw String Literals to Avoid Escaping
hand escape 된 string 은 가독성이 안좋으니 raw string literal 을 사용하라.

## Initializing Structs

`go vet` 은 강제하고 있는 내용이긴 한데, struct 생성할때 field name 을 명시하라. 단, 3개 이하의 필드 갯수를 가지면 생략될 수 있다.

3개면 적은 숫자는 맞지만, 굳이 예외를 둘 필요는 없어보인다.

## Omit Zero Value Fields in Structs

struct 초기화 할때, 유의미한게 아닌 경우 zero value 는 생략한다. 의미있는 값만 세팅하도록 한다.

## Initializing Struct References

`new(T)` 대신 `&T{}` 를 사용하여 일관성을 유지한다.

## Initializing Maps

`map` 은 make 로 만들어서 이후에 hint 를 추가하기에도 용이하다.
map 이 fixed 된 경우에는 map literal 로 초기화한다.

## Format Strings outside Printf

`Printf` 스타일의 function 내에 format string 을 선언할때 const 로 하라

```golang
const msg = "unexpected values %v, %v\n"
fmt.Printf(msg, 1, 2)
```

# contents(Patterns)

## Test Tables

반복하지 않고, test table 을 구현해서 loop 로 돌린다.

input 은 `give`, output 은 `want` prefix 를 사용한다.

test table 은 `tests` 로 선언하고, range 할때는 `tt` 로 받는다.

## Functional Options

내부 struct 의 값을 전달하는 방식 대신 Option 을 전달하는데, 확장될 가능성이 있는 public API 에 사용하라.

`gRPC` 에서도 사용되고 있는 패턴인데, 예시를 기록해두는 차원에서 아래에 옮겨둔다.

설명을 좀 하면, `options` 라는 unexported struct 를 만들고 `Option` 은 `apply(*options)` 를 구현하는 interface 이다. `Option` 은 결국 `*options` 의 값을 변화시키는 역할을 한다.

```golang
type options struct {
  cache  bool
  logger *zap.Logger
}

type Option interface {
  apply(*options)
}

type cacheOption bool

func (c cacheOption) apply(opts *options) {
  opts.cache = bool(c)
}

func WithCache(c bool) Option {
  return cacheOption(c)
}

type loggerOption struct {
  Log *zap.Logger
}

func (l loggerOption) apply(opts *options) {
  opts.logger = l.Log
}

func WithLogger(log *zap.Logger) Option {
  return loggerOption{Log: log}
}

// Open creates a connection.
func Open(addr string,  opts ...Option) (*Connection, error) {
  options := options{
    cache:  defaultCache,
    logger: zap.NewNop(),
  }

  for _, o := range opts {
    o.apply(&options)
  }

  // ...
}
```

관련 내용은 [go api 호환성을 유지하기 위한 방법들](https://fransoaardi.github.io/posts/go_maintain_compatibility/) 이라는 포스팅에 정리한적 있다.

# contents(Linting)

`errcheck`, `goimports`, `golint`, `govet`, `staticcheck` 기능은 사용하라고 한다.

## Lint Runners

`golangci-lint` 를 사용하면 좋다고 한다.
팀에서 함께 설정하고 필요에 맞게 설정하면 된다고 하는데 설정을 한번 해봐야겠다.