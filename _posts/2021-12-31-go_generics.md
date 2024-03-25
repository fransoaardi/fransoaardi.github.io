---
layout: post
title: "GopherCon 2021: Generics! by Robert Griesemer & Ian Lance Taylor" 
date: 2021-12-31 11:35:00 +0900
categories: [Seminar]
tags: [golang]
toc: true
comments: true
---

# Introduction

GopherCon 2021 이 12월6일-10일 에 진행됐었다고 한다. 정신없이 지내느라 전혀 확인하지 못했지만, 이번 GopherCon 에 Robert Griesemer 와 Ian Lance Taylor 가 Go 1.18 에 추가될 Generics 관련한 발표를 한다는 소식을 우연히 전해들어 알고는 있었고, 한참이 지난 지금에야 영상을 확인하고 정리를 해볼 수 있었다. 아직 1.18 을 써보기 전이고, 업무상으로 언제 1.18 을 사용해볼 수 있을지는 모르겠지만 개인적으로 기대가 많이 되고 큰 변화가 예상되는 내용이다.

이 포스팅은 아래의 영상을 보고, 해석하여 거의 그대로 옮기는 내용에 지나지 않는다.

[Generics! 발표 Link](https://www.youtube.com/watch?v=Pa_e9EeCdy8)

# Contents

Generics 는 go 1.18 에 추가될것이다.

New language features in Go 1.18
- Type parameters for functions and types
- Type sets defined by interfaces
- Type inference

## Type parameters

- Type paraemeter lists 는 `[]` 를 이용해서 표현하고 대문자를 이용하여 type 임을 강조한다.
    - 예시: `[P, Q constraint1, R constraint2]`

기존에는 `min` function 을 구현할때 아래와 같이 했다.
```golang
func min(x, y int) {
    if x < y {
        return x
    }
    return y
}
```

하지만 앞으로는 generics 를 이용해서 아래와같이 선언하고 호출하면 된다. 
```golang
func min[T constraints.Ordered](x,y T) T {
    if x < y {
        return x
    }
    return y
}

m := min[int](2,3)
```

Instantiation
1. Substitute type arguments for type parameters.
2. Check that type arguments implement their constraints

Instantiation fails if step 2 fails.

```golang
// fmin 은 (min[float64]) 로 판단된다. Instantiati on 은 non-generic function 을 생성하고 평범한 함수를 호출하듯 사용하면 된다.
fmin := min[float64]
m := fmin(2.71, 3.14)
```
```golang
// Type 도 type parameter lists 를 가질 수 있다.
type Tree[T interface{}] struct {
    left, right *Tree[T]
    data T
}

// Type parameter 를 receiver 로 method 를 구현한 것을 확인할 수 있다.
func (t *Tree[T]) Lookup(x T) *Tree[T]

var stringTree Tree[string]
```

## Type sets

Type parameter lists 에도 type 을 지정할 수 있다. 이것을 type constraint 라고 한다. 
위의 예에서는 `[T constraints.Ordered]` 이다. 크기를 비교할 수 있는(Orderable) 값들만 T 로 넘길 수 있다. T 는 `<` 를 사용할 수 있다.

> Type constrainst 는 interface 이다.

`constraints.Ordered` 는 아래와 같이 정의되어 있다. `Ordered` 는 integer, floating-point, string 을 정의한다. 이 타입들은 모두 `<` operator 를 사용할 수 있다. 
```golang
// 중요: Ordered interface 는 method 를 정의하지 않았다.
package constraints

type Ordered interface {
    // ~ 는 새롭게 추가된 토큰으로, 아래 예시에서 underlying type 이 string 인 모든 type 들을 의미한다
    Integer|Float|~string
}
```

Type constraint 의 두가지 기능 (당연한 얘기이다)
1. The type set of a constraint 는 유효한 type 의 모음이다.
2. 만약 constraint type 에 포함된 모든 type 들이 operation 기능을 제공하면, 그 operation 은 해당 type 을 사용할 수 있습니다.



```golang
// in line 으로 정의할 수 있다.
[S interface{~[]E}, E interface{}]

// interface{E} 를 E for type elements E 로 적을 수 있다.
[S ~[]E, E interface{}]

// 새로운 identifier 인 any 를 interface{} 의 alias 로 활용 할 수 있다.
[S ~[]E, E any]
```

## Type inference

앞서 정의한 `min` 은 type inference 될것이다.
```golang
var a, b, m float64
// Passing type arguments leads to more verbose code.
m = min[float64](a,b)
// The type argument float64 is inferred from the arguments a and b.
m = min(a,b)
```

> The details of type inference are complicated but using it is easy.
>> type inference 의 과정은 복잡하겠지만, 우리는 일단 쉽게 잘 사용하면 될 것 같다.

---

## 예시를 통해 보는 Generics

```golang
// Point 를 []int32 로 정의한다
type Point []int32
// Point 를 String 을 구현한다.
func (p *Point) String() string{ ... }
// Scale 을 아래와 같이 정의하고, E 로 Point 를 넘길 생각이다. 
func Scale[E constraints.Integer](s []E, scaleVal E) []E {
    r := make([]E, len(s))
    for i, v := range s {
        r[i] = v * scaleVal
    }
    return r
}

func ScaleAndPrint(p Point) {
    r := Scale(p, 2)
    fmt.Println(r.String()) // Compile 할 수 없다; 왜냐하면 r 은 Point 가 아닌 []int32 로 이고, []int32 는 String() 을 구현하지 않았기 때문이다.
}
```

그래서 아래와 같이 `Scale` 을 변경해줘야된다.
```golang
func Scale[S ~[]E, E constraints.Integer](s S, scaleVal E) S {
    r := make(S, len(s)))
    // 이하 동일
}
```

Constraint type inference 는 type parameter constraint 로 부터 우선 판단하고, 다른 argument 들을 추론해낸다고 한다. 위의 예에서 `scaleVal` 인 `E` 가 `int32` 인 것을 확인하고, `~[]E` 인 `S` 를 `Point` 로 추론해내는 방식인 것 같다. 따라서 `Scale` 은 `Scale[Point, int32]` 를 호출한다.

Constraint type inference 에 관한 자세한 내용은 [링크](https://go.dev/s/generics-proposal)에 있다고 한다.

## When to use generics

> Writing Go: Write code, don't design types.

먼저 type parameter 를 정의한다면 잘못됐을 가능성이 높다. 유용할 것이라는게 확실해지면 이후에 추가하는게 더 쉬울 것이다.

### When are type parameters useful?

- slices, maps, and channels of any element type 에서 동작하는 함수를 정의하는 경우
- 범용적인 자료구조인 경우
    - When operating on type parameters, perfer functions to methods.
- 모든 타입들에 대해 method 가 동일할때

## When not to use generics

- 특정 type 에 대해 method 를 호출하는 경우. (그냥 interface 를 쓰면 된다)
```golang
// good
func ReadFour(r io.Reader) ([]byte, error)
// bad: 이럴필요가 없다
func ReadFour[T io.Reader](r T) ([]byte, error)
```
- 구현이 타입마다 달라지는 경우.
- 각 type 마다 operation 이 다른 경우에. 예를들어 `json.Marshal` 처럼 `Marshaler` 를 구현한 경우도 있고 아닌 경우도 있을것이라서, reflection 을 사용하는게 유리할 것이라고 한다.

> 계속 동일한 코드를 작성하고 있는 경우에, type parameter 를 이용하라.
> 미리 사용하진 말고 정말 필요할때 사용하라.

Ian Lance Taylor 의 마지막 말은 아래와 같은데, 정말로 섣부르게 사용하는 사람들이 많이 걱정됐나 보다. 
> I hope you use them wisely and well. 

