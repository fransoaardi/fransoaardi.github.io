---
layout: post
title: Golang 새롭게 알게 된 것들에 대한 기록 (1)
date: 2021-08-02 23:22:00 +0900
categories: [Wiki]
tags: [golang]
toc: true
comments: true
---

# introduction

공부하며 새롭게 알게된 사실들을 기록 차원에서 정리했다.
계속해서 내용을 추가할 예정이다.

# contents

목차

- receiver function 호출 시 implicit 하게 일어나는 일들
- call by value vs call by reference 의 성능차이
- golang 에서 map 은 reference 가 아니고, reference 로 param 을 passing 하는게 없다

## receiver function 호출 시 implicit 하게 일어나는 일들

receiver function 구현시 ptr 로 구현하지 않으면 value 가 변하지 않을것은 뻔하다.
4가지 테스트를 해봤는데 각각 value receiver function 호출을 value 와 pointer 로 한 것과 ptr receiver function 호출을 value 와 pointer 로 한 것 이다. 
아래에 Person 은 value receiver, Person1 은 ptr receiver 이다.

실험결과 다 예상대로 동작했으나, `p2` 와 `p3` 의 결과를 주목해야된다.
golang 에서 `p2(pointer)` 는 `(*p2).changeName("jongsuk")` 으로 동작했는데, `(*p2)` 는 새로운 `Person` 의 copy 일 뿐이라 `p2` 의 값이 변화하진 않았다.

그리고 `p3(value)` 는 `(&p3).changeName("jongsuk")` 으로 동작해서, 자연스럽게 `p3` 의 값이 변화했다.

실제로 `p2` 와 `p3` 를 `ChangeName` 호출 해볼때 `fmt.Printf("%p", &p)` 와 같이 address 를 확인하고 `p2`, `p3` 각각의 address 를 비교해봤을때 `p2` 는 달랐고, `p3` 는 동일했다.

```go
package main

import (
	"fmt"
)

type Person struct {
	Name string
}

func (p Person) ChangeName(name string) {
	p.Name = name
}

type Person1 struct {
	Name string
}

func (p *Person1) ChangeName(name string) {
	p.Name = name
}

func main() {
	p1 := Person{ Name: "종석님" }
	p1.ChangeName("jongsuk")
	fmt.Println(p1)
	
	p2 := &Person{ Name: "종석님" }
	p2.ChangeName("jongsuk")
	fmt.Println(p2)
	
	p3 := Person1{ Name: "종석님" }
	p3.ChangeName("jongsuk")
	fmt.Println(p3)
	
	p4 := &Person1{ Name: "종석님" }
	p4.ChangeName("jongsuk")
	fmt.Println(p4)

// Output:
// {종석님}
// &{종석님}
// {jongsuk}
// &{jongsuk}
}
```

## call by value vs call by reference 의 성능차이

**TL;DR: 결론. 정말 모르겠다.**

call by value 와 call by reference 의 성능차이는 뭔가 명확하게 정리된 문서를 발견하진 못했다.
이전에 읽었던 문서들에서는 call by value 와 reference 의 차이가 크게 나지 않으니 굳이 최적화를 하겠다고 call by reference 할 필요는 없고 정말로 필요한 경우에 적절하게 사용하라고 했던 걸로 기억한다. 

golang 이 c style 이니 당연히 value 보다는 reference 로 넘기는게 성능상 이점이 있을거라고 생각되지만 어떤 문서에서 이러한 얘기를 봤다.

go 는 function 의 stack frame 에 안전하게 allocate 될 수 있는지를 확인하기 위해 escape analysis 를 한다. heap 에 allocate 되는것보다 stack frame 에 allocate 되는게 훨씬 비용이 저렴하기 때문이다. passing by value 하는게 escape analysis 를 단순화 하기 때문에 stack 에 allocation 될 확률을 높여준다고 한다.

출처: [https://goinbigdata.com/golang-pass-by-pointer-vs-pass-by-value/](https://goinbigdata.com/golang-pass-by-pointer-vs-pass-by-value/)

> Passing by value often is cheaper.

> Even though Go looks a bit like C, its compiler works differently. And C analogy does not always work with Go. Passing by value in Go may be significantly cheaper than passing by pointer. This happens because Go uses escape analysis to determine if variable can be safely allocated on function’s stack frame, which could be much cheaper then allocating variable on the heap. Passing by value simplifies escape analysis in Go and gives variable a better chance to be allocated on the stack.

실험해보고 싶어서 아래와 단순히 적당히 큰 struct 를 생성해서 call by value func 와 call by reference func 를 호출해보는 benchmark 코드를 작성해봤다. 그런데 동시에 드는 생각은 위 글 대로라면 escape analysis 결과에 따라 달라질 것이고 유의미하지 않다는 생각이 든다.

예를들어, 전달된 value 가 내부에 slice, map 과 같은 type 을 갖고있고 이 값이 return 된다면 escape analysis 결과 heap 에 allocate 되는 결과로 이어져 변수가 너무 크다. 그래도 기록을 위해서 아래에 코드와 escape analysis 결과를 아래에 남긴다.

```golang
package main

import (
	"math/rand"
	"time"
)

func main() {
	x := genRandX()
	passValue(x)
	passReference(&x)
}

type X struct {
	A string
	B string
	C string
	D int
	E int
	F []int
}

func genRandX() X {
	var randString = []string{"one", "two", "three", "four", "five"}
	var randInt = []int{0, 1, 2, 3, 4}

	src := rand.NewSource(time.Now().UnixNano())
	r := rand.New(src)
	return X{
		A: randString[r.Intn(5)],
		B: randString[r.Intn(5)],
		C: randString[r.Intn(5)],
		D: randInt[r.Intn(5)],
		E: randInt[r.Intn(5)],
		F: []int{1, 2, 3, 4},
	}
}
```

```
❯❯❯ go build -o main -gcflags="-m" && ./main
./main.go:27:43: inlining call to time.Time.UnixNano
./main.go:27:43: inlining call to time.(*Time).unixSec
./main.go:27:43: inlining call to time.(*Time).sec
./main.go:27:43: inlining call to time.(*Time).nsec
./main.go:27:23: inlining call to rand.NewSource
./main.go:28:15: inlining call to rand.New
./main.go:39:6: can inline passValue
./main.go:43:6: can inline passReference
./main.go:8:6: can inline main
./main.go:10:11: inlining call to passValue
./main.go:11:15: inlining call to passReference
./main.go:27:23: moved to heap: rand.rng·3
./main.go:24:27: []string{...} does not escape
./main.go:25:21: []int{...} does not escape
./main.go:28:15: &rand.Rand{...} does not escape
./main.go:35:11: []int{...} escapes to heap
./main.go:39:16: leaking param: val to result ~r1 level=0
./main.go:43:20: leaking param: ref to result ~r1 level=1
```

## golang 에서 map 은 reference 가 아니고, reference 로 param 을 passing 하는게 없다

아래 간단해보이는 코드를 실행해보면 상당히 당황스러운 결과가 출력된다. 

결과를 해석해보면 첫번째 출력에서 `m` 은 `nil` 이다. 아직 `make` 가 안됐기 때문이다. 
`fn` 에 전달된건 `nil` 이기 때문에 `two` 역시 `nil`이다.
`m` 에는 새롭게 allocation 이 일어나서 새로운 주소값을 갖게되고 그 map 의 key 1 에는 2 라는 값이 할당됐다.
따라서 `three` 는 주소값을 갖고있다.
다만, `three` 의 주소값이 할당됐을뿐 main 의 `m` 의 값을 바꾸진 않았다. 따라서 `four` 에서 `m` 은 여전히 `nil` 이다.
그래서 `m[1]` 은 `nil` 에서 1 을 조회했기때문에 int 의 zero-value 인 0 을 반환한다.
`m[1]` 에 3을 할당하는 과정에서는 `panic` 이 일어난다. `m` 은 아직 allocate 안된 `nil` 이기 때문이다.


이 결과를 통해 map 이 reference 가 아니기때문에 fn 으로 전달된 m 에 새로운 allocate 가 일어났어도 main 의 m 은 바뀌지 못했음을 알 수 있다.

아래 [링크](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)에 있는 글에 따르면 map 과 channel 은 runtime type 의 pointer(`A map value is a pointer to a runtime.hmap structure.`) 이라고 한다. slice 는 다르다.

> Maps, like channels, but unlike slices, are just pointers to runtime types. As you saw above, a map is just a pointer to a runtime.hmap structure.

```go
package main

import "fmt"

func fn(m map[int]int) {
	fmt.Printf("two: %p\n", m)
	m = make(map[int]int)
	m[1] = 2
	fmt.Printf("three: %p\n", m)
}

func main() {
    var m map[int]int
	fmt.Printf("one: %p\n", m)
    fn(m)
	fmt.Printf("four: %p\n", m)
	fmt.Println(m[1])
    m[1] = 3
}

// Output:
// one: 0x0
// two: 0x0
// three: 0xc0000121b0
// four: 0x0
// 0
// panic: assignment to entry in nil map
//
// goroutine 1 [running]:
// main.main()
// 	/tmp/sandbox755613608/prog.go:19 +0x1bd
```

references:
- [If a map isn’t a reference variable, what is it?](https://dave.cheney.net/2017/04/30/if-a-map-isnt-a-reference-variable-what-is-it)
- [there-is-no-pass-by-reference-in-go](https://dave.cheney.net/2017/04/29/there-is-no-pass-by-reference-in-go)