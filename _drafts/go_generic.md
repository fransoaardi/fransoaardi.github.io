---
layout: post
title: go-generics
date: 2020-12-11 01:20:00 +0900
categories: [Wiki]
tags: [golang, golang2]
toc: true
---

# introduction
- golang 에는 generics 가 없고, 앞으로도 없을것이라던 얘기가 있었는데 결국 들어왔다.
- community 의 요청을 못이긴것 같고, 아래 부분에 뭔가 말에 뼈가 있다. `generic 이 있었으면 해결했을 수 있는 문제들`이 있었는지 확인해보고 싶다고 한다.

> https://blog.golang.org/generics-next-step

```
Second, we know that many people have said that Go needs generics, but we don’t necessarily know exactly what that means. Does this draft design address the problem in a useful way? If there is a problem that makes you think “I could solve this if Go had generics,” can you solve the problem when using this tool?
```

# references

> golang 의 generics 
- https://blog.golang.org/generics-next-step

- https://go.googlesource.com/proposal/+/refs/heads/master/design/go2draft-type-parameters.md#type-parameters-draft-design

> `https://go2goplay.golang.org/` 에 있는, 기본 예제를 바탕으로 테스트 코드를 작성해보았다.

```golang
package main

import (
	"fmt"
)

func Print[T any](s []T) {
	for _, v := range s {
		fmt.Print(v) 
	}
	fmt.Println()
}

func main() {
    // Print 의 T 에 string 임을 알려준다, args 에 []string 을 전달한다
    Print[string]([]string{"Hello, ", "playground\n"})
    // Print 의 T 에 int 임을 알려준다, args 에 []int 를 전달한다
	Print[int]([]int{1,2,3})
        
    // Print 의 T 에 Crier (interface) 를 전달
    Print[Crier]([]Crier{Duck{}, Cat{}})	// Output: quackmeow
    
    // Print 의 T 에 Duck (struct) 를 전달
	Print[Duck]([]Duck{Duck{}})	// Output: quack
    
    // Print 에 T 를 전달 안하고 implicit 하게 전달해봄
    Print([]Crier{Duck{}, Cat{}})	// Output: quackmeow

//	Print[Duck]([]Crier{Duck{}, Cat{}}) // error occur: cannot use ([]Crier literal) (value of type []Crier) as []Duck value in argument
//	Print[Duck]([]Crier{Duck{}}) // error occur: cannot use ([]Crier literal) (value of type []Crier) as []Duck value in argument
}

type Crier interface{
	Cry() string
}

type Duck struct{}
func (Duck) Cry() string{
	return "quack"
}
// String() 은 Stringer interface 의 implementation. 위의 Print() 에서 사용하는 fmt.Print 는 내부적으로 Stringer 인지 체크해서 출력하기 때문에, String() 을 따로 구현했다.
func (d Duck) String() string {
	return d.Cry()
}

type Cat struct{}
func (Cat) Cry() string{
	return "meow"
}
// String() 은 Stringer interface 의 implementation. 위의 Print() 에서 사용하는 fmt.Print 는 내부적으로 Stringer 인지 체크해서 출력하기 때문에, String() 을 따로 구현했다.
func (c Cat) String() string {
	return c.Cry()
}
```