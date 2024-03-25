---
layout: post
title: Uber go style guide (1)
date: 2021-05-26 00:30:00 +0900
categories: [Wiki]
tags: [golang, style-guide]
toc: true
comments: true
---

# introduction

팀 내에서 서로의 코드리뷰 퀄리티와 코드 생산성 향상을 위해 Uber 가 정리한 golang style guide 를 함께 읽어보는 시간을 갖기로 했다. 

아래 내용들은, 읽으면서 몰랐던 부분이나, 정리해두고 싶은 내용들을 다룬다.

아래의 코드와 내용들은 [https://github.com/uber-go/guide/blob/master/style.md](https://github.com/uber-go/guide/blob/master/style.md) 문서에서 발췌한 내용이 대부분임을 미리 밝힌다.

# contents

## Verify Interface Compliance

정의한 interface 가 적절한지 compile time 에 확인하기 위해 아래와 같이 unnamed variable 로 선언해보는 방식이다. 

pointer type receiver function 인 경우는 아래와 같이 nil 을 세팅하고
```golang
var _ http.Handler = (*Handler)(nil)

func (h *Handler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  // ...
}
```

struct type receiver function 인 경우는 zerovalue 를 세팅한다.

```golang
var _ http.Handler = LogHandler{}

func (h LogHandler) ServeHTTP(w http.ResponseWriter, r *http.Request) {
  // ...
}
```

## Zero-value Mutexes are Valid

`smap` 과 같이 unexported type 은 mutex 를 embed 해도 괜찮고, `SMap` 처럼 exported type 은 mutex 를 unexported type 으로 정의해서 사용한다.

```golang
type smap struct {
  sync.Mutex // only for unexported types

  data map[string]string
}

type SMap struct {
  mu sync.Mutex

  data map[string]string
}
```

## Copy Slices and Maps at Boundaries

slice, map 은 pointer 를 가지고 있으므로, 복사하거나 return 하는 경우에 주의해야된다.

```golang
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = trips   // 이 경우 trips 의 slice ptr 가 넘어가서, trips 를 바꾸면 d.trips 가 영향받는다 
}

trips := ...
d1.SetTrips(trips)

// Did you mean to modify d1.trips?
trips[0] = ...
```

따라서 아래와 같이 새로운 할당을 한 뒤에 copy 호출하는것이 바람직하다.
```golang
func (d *Driver) SetTrips(trips []Trip) {
  d.trips = make([]Trip, len(trips))
  copy(d.trips, trips)
}
```

map, slice 를 return 하는 경우에 원본을 내려준것과 동일하기 때문에 아래의 예시에서 기껏 `Stats` 에 mutex 를 적용해서 map access 에 대한 race condition 을 막기 위한 노력이 헛수고가 되어버린다. 물론 `Stats` 의  `counters` 의 data access 하는 부분의 코드는 아래에 제공되진 않았다.

```golang
type Stats struct {
  mu sync.Mutex
  counters map[string]int
}

// Snapshot returns the current stats.
func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  return s.counters
}

// snapshot is no longer protected by the mutex, so any
// access to the snapshot is subject to data races.
snapshot := stats.Snapshot()
```

따라서 아래와 같이 map 을 새로 선언해서 복사해서 내려주어서 이 map 에 대한 race condition 을 방지하는 access 는 `Stats` 의 mutex 가 아닌 다른 방식으로 직접 구현하도록 한다.

```golang
func (s *Stats) Snapshot() map[string]int {
  s.mu.Lock()
  defer s.mu.Unlock()

  result := make(map[string]int, len(s.counters))
  for k, v := range s.counters {
    result[k] = v
  }
  return result
}
```

## Defer to Clean Up

구절이 마음에 들었다. 작성한 함수가 `defer` 만큼 빠른게 아니면, `defer` 를 써서 코드 가독성을 높이라는 의미다.


> Defer has an extremely small overhead and should be avoided only if you can prove that your function execution time is in the order of nanoseconds. 

> The readability win of using defers is worth the miniscule cost of using them

## Channel Size is One or None

채널을 선언할때 unbuffered 로 선언하든, buffered 로 할거면 size 는 1만 잡으라는 이야기인데, 잘 이해가 되진 않는다. 막연히 늘여놓고 제대로 알지 못한 상태로 쓰진 말라는 의미로 이해했다.

## Start Enums at One

zerovalue 가 0인 경우가 있을 수 있으니, 아래 처럼 enum 할때 값을 0 이 아닌 1부터 시작하게끔 하라는 의미이다.

```golang
type Operation int

const (
  Add Operation = iota + 1
  Subtract
  Multiply
)
```

물론, zerovalue 가 그 자체로 유의미한 경우가 있을 수 있으니, 그런 경우는 허용한다. 아래의 예제에서는 stdout 으로 log output 기본설정 하는걸 의도하는 코드이다.

```golang
type LogOutput int

const (
  LogToStdout LogOutput = iota
  LogToFile
  LogToRemote
)

// LogToStdout=0, LogToFile=1, LogToRemote=2
```

## Use "time" to handle time

int type 으로 의미상 '초' 에 해당하는 값을 받지 말고, param 자체를 `time.Time` 혹은 `time.Duration` 으로 직접 받으라는 의미이다. 

> When it is not possible to use time.Duration in these interactions, use int or float64 and include the unit in the name of the field.

또한, 외부 시스템과 시간을 주고받을때는 time.Duration 혹은 time.Time 을 이용하지만, 그럴 수 없는 경우 `int` 나 `float64` 를 이용하고 변수명에 단위를 포함시킨다. 아래와 같다.

```golang
// {"intervalMillis": 2000}
type Config struct {
  IntervalMillis int `json:"intervalMillis"`
}
```

## Error Types

- 추가정보 필요없는 단순 에러라면 `errors.New` 쓰면 된다.
- client 가 이 에러를 감지하고 처리해야된다면 custom type 으로 정의하고 `Error()` 를 구현한다.
- 에러를 전파하고싶으면 `error wrapping` 하라
- 그 외 경우는 `fmt.Errorf` 이면 된다.

아래와 같이, error 변수로 선언하고, `errors.Is` 를 이용해서 비교한다.
```golang
var ErrCouldNotOpen = errors.New("could not open")
```

- error.Error() 호출해서 error message string 비교 하지 말고, type assertion 을 하고, expose 되지 않은 type 이라면, matcher function 을 expose 하는게 더 낫다.
```golang
func IsNotFoundError(err error) bool {
  _, ok := err.(errNotFound)
  return ok
}
```

error 에 context 를 추가할때, `failed to` 와 같은 표현은 피하고, 메세지를 간결하게 유지한다.
나는 보통 `store.New() failed` 라고 구현했을것 같다.
```golang
s, err := store.New()
if err != nil {
    return fmt.Errorf("new store: %v", err)
}
// Output: x: y: new store: the error
```

type assertion 은 잘못될 경우 panic 일어날 것이니,  `t, ok` 구문을 항상 사용한다.

function 은 항상 caller 가 어떻게 error 를 대응할것인지를 선택할 수 있도록, function 안에서 panic 하지 않는다.

panic/recover 는 error 대응 전략이 아니다. nil dereference 처럼 정말로 복구 불가능한 일이 발생했을때만 panic 해야 되는데, 프로그램 init 과정에서 잘못된 시작으로 프로그램을 멈출 것 같은 경우에 panic 을 일으켜도 된다.

테스트 코드를 작성할때도 panic 을 직접 일을키지 말고, `t.Fatal`, `t.FailNow` 을 사용해서 test 가 fail 됐음을 표기하도록 한다.

## `go.uber.org/atomic` 을 사용하라

`sync/atomic` 은 `int32`, `int64` 와 같은 raw type 에 대해서만 동작한다. 

`go.uber.org/atomic` 은 Bool type 도 지원해서 좋다.
```golang
type foo struct {
  running atomic.Bool
}

func (f *foo) start() {
  if f.running.Swap(true) {
     // already running…
     return
  }
  // start the Foo
}

func (f *foo) isRunning() bool {
  return f.running.Load()
}
```

## Avoid Mutable Globals
전역 변수를 변경하지 말고 종속성 주입을 선택하세요. 다른 종류의 값뿐만 아니라 함수 포인터에도 적용됩니다.

global var 로 선언했던 부분을 변경하면, 이를 참조하는 다른 코드들이 영향 받을 수 있으니, type 선언으로 의존성 주입하는게 낫다고 함.

```golang
func TestSigner(t *testing.T) {
  s := newSigner()
  s.now = func() time.Time {
    return someFixedTime
  }

  assert.Equal(t, want, s.Sign(give))
}
```

## Avoid Embedding Types in Public Structs

아래와 같이 embed 시키지 말고, 왠만하면 사용하는 기능들을 직접 구현하라는 의미이다. 
```golang
type AbstractList interface {
  Add(Entity)
  Remove(Entity)
}
// Bad
type ConcreteList struct {
  AbstractList
}
// Good
type ConcreteList struct {
  list AbstractList
}

// Add adds an entity to the list.
func (l *ConcreteList) Add(e Entity) {
  l.list.Add(e)
}

// Remove removes an entity from the list.
func (l *ConcreteList) Remove(e Entity) {
  l.list.Remove(e)
}
```

이유는, AbstractList 의 변경(기능 추가, 삭제)이 필요할때 영향도를 줄이기 위함인데, `조금만 노력을 들이면` implementation detail 을 감출 수 있고, 변경에 대한 기회를 얻을 수 있다고 한다.

## Avoid Using Built-In Names
헷갈릴 수 있고, built-in 이름을 shadow 할 수 있으니, 쓰지 않는다.

## Avoid init()

- 프로그램 환경에 상관없이 완벽하게 결정적이도록 코드를 작성한다.
- 다른 `init()` 코드들 간의 순서와 사이드이펙트에 의존하는걸 피한다. 코드가 에러날 확률도 늘어나고, 코드가 취약해질 수 있다.
- init 에서 global/환경 변수(환경변수, working directory, args 등) 를 조작하거나 접근하거나 하는것을 피한다
- init 에서 filesystem, network, system call 과 같은 i/o 를 피한다.

init 쓰지 말고, 아래와 같이 
```golang
// or, better, for testability:
var _defaultFoo = defaultFoo()

func defaultFoo() Foo {
    return Foo{
        // ...
    }
}
```

아래와 같은 경우는 `init` 이 preferable 하다고 한다.

```
- Complex expressions that cannot be represented as single assignments.

- Pluggable hooks, such as database/sql dialects, encoding type registries, etc.

- Optimizations to Google Cloud Functions and other forms of deterministic precomputation.
```

## Exit in Main

- `log.Fatal` 은 `os.Exit(1)` 을 수반한다.

여러 function 들에서 exit 이 있는 경우 몇가지 이슈가 있다.

1. control flow 에 대해 이해하기 힘들다.
2. 테스트 하기 힘들다, go test 도중에 os.Exit 하면 테스트가 멈추고, 다른 테스트가 진행되지 않는다.
3. cleanup 이 스킵된다, 새로 알게된 재밌는 사실인데, `os.Exit` 은 defer 가 실행되지 않는다. 단, panic 인 경우에는 defer 가 실행이 된다.

```golang
package main

import (
	"fmt"
	"os"
)

func inner() {
	fmt.Println("inner")
	defer fmt.Println("inner defer")
	os.Exit(1)
	panic("panic inner!")
}

func main() {
	fmt.Println("main")
	defer fmt.Println("main defer")
	inner()
	os.Exit(1)
	panic("panic main")
}

// Output: 
// main
// inner
//
// Program exited: status 1.
```

- `os.Exit` 혹은 `log.Fatal` 은 main 에서 최대 한번까지만 호출하도록 한다. 
- 프로그램을 종료시키는 다양한 에러 시나리오가 있다면, 분리된 function 에서 error 를 처리하도록 한다.
- `main` 을 짧게 유지할 수 있게 된다.