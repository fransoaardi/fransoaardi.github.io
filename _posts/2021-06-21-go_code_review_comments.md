---
layout: post
title: Go Code Review Comments
date: 2021-06-21 00:45:00 +0900
categories: [Wiki]
tags: [golang, style-guide]
toc: true
comments: true
---

# introduction

팀 내에서 서로의 코드리뷰 퀄리티와 코드 생산성 향상을 위해 각종 문서들을 읽어보고 토론하는 시간을 갖고있다.
이번에는 golang code review comments 를 함께 읽어보는 시간을 갖기로 했다. 
아래 내용들은, 읽으면서 몰랐던 부분이나, 정리해두고 싶은 내용들을 다룬다.

아래의 코드와 내용들은 [https://github.com/golang/go/wiki/CodeReviewComments](https://github.com/golang/go/wiki/CodeReviewComments) 문서에서 발췌한 내용이 대부분임을 미리 밝힌다.

이미 진행했던 연관 문서 링크를 아래에 첨부한다.

[Uber go style guide (1) 바로가기](https://fransoaardi.github.io/posts/uber_go_style_guide_1/)
[Uber go style guide (2) 바로가기](https://fransoaardi.github.io/posts/uber_go_style_guide_2/)

# contents

## Gofmt

`gofmt` 나 `goimports` 를 실행해서, 대부분의 style 이슈를 해결하라.

## Comment Sentences

주석들은 중복처럼 보여도 완전한 문장으로 작성한다. `godoc` 문서로 추출될때 잘 짜여지도록 하기 위함이다. 주석은 설명하려는 대상의 이름으로 시작하고, 마침표로 끝낸다. 

- 마침표로 끝내야된다는 사실은 기억해둬야될것 같다.

## Contexts

`context.Context` 는 security credentials, tracing information, deadlines, cancellation signals 등을 갖고있고 API 호출 전반에 전달한다. Context 를 사용하는 함수들은 context 를 첫번째 parameter 로 받아들여야된다.

request-specific 하지 않은 함수는 `context.Background()` 를 쓸 수 있지만, 확실히 `context.Background()` 를 사용하지 않아야 될 이유가 있지 않은 경우가 아니라면 `context` 자체를 전달하라.

Context 는 struct 에 member 로 추가하지 말고, 파라미터로 전달하라. 단 하나의 예외는, standard library 혹은 third party library 의 interface 를 맞추기 위한 경우다.

custom Context type 을 만들거나 함수 signature 에 Context 외의 다른 interface 를 사용하지 말라.

만약 전달할 데이터가 있다면, parameter / receiver / global 에 넣거나, 정말로 필요하면 Context 에 담아라.

Context 는 immutable 이라 같은 context 를 deadline, cancellation signal, credentials, parent trace 등을 공유하는 다양한 호출에 여러번 전달하는게 괜찮다.

## Copying

예상치못한 aliasing 을 피하기 위해, 다른 패키지의 struct 를 복사하는 것을 주의하라. 예를들어, `bytes.Buffer` 는 `[]byte` 를 갖고 있어서, `Buffer` 를 복사하는 경우 `[]byte` slice 의 원본 array 에 있는 내용을 alias 하게 되어 예상치 못한 일이 일어날 것이다.

보통, `*T` 의 methods 가 있는 `T` 는 복사하지 말라.

## Crypto Rand

`math/rand` 를 이용해서 key 를 만들지 말라. Seed 가 생성되지 않으면 완전히 값이 예상 가능하고, `time.Nanoseconds()` 로 seeded 되더라도 조금의 엔트로피가 추가될 뿐이다. 대신 `crypto/rand` 를 사용하라.

```golang
import (
	"crypto/rand"
	// "encoding/base64"
	// "encoding/hex"
	"fmt"
)

func Key() string {
	buf := make([]byte, 16)
	_, err := rand.Read(buf)
	if err != nil {
		panic(err)  // out of randomness, should never happen
	}
	return fmt.Sprintf("%x", buf)
	// or hex.EncodeToString(buf)
	// or base64.StdEncoding.EncodeToString(buf)
}
```

## Declaring Empty Slices

빈 slice 를 생성할때 `var` 를 이용하는게 낫다.
```golang
var t []string // good
t := []string{} // bad
```

전자는 nil slice 를 생성하지만, 후자는 nil 이 아닌, zero length 의 slice 를 할당한다. `len`, `cap` 은 전자 후자 모두 0 이지만, nil slice 가 더 선호되는 스타일이다.

JSON objects 를 encode 할때 nil slice 는 `null` 이 되고, `[]string{}` 은 `[]` 이 되는 것 처럼, 제한적인 상황이 잇다.

interface 설계할때 nil slice 와 non-nil, zero-length slice 를 구분하는 것은 지양하라.

## Doc Comments

모든 top-level, exported 된 이름들은 doc comments 를 가져야되고, 중요한 unexported type 혹은 function 도 주석이 필요하다.

## Don't Panic

일반적인 에러 처리에 panic 을 사용하지 말라.

## Error Strings

error strings 는 보통 다른 context 뒤에 출력되기 때문에, 대문자로 시작하지 않고, 구두점을 찍지 않는다. 로깅은 보통 line-oriented 되고 다른 메세지와 결합되지 않기 때문에, 이런 규칙을 따르진 않는다.

## Examples

새로운 패키지를 생성할때, 의도한대로 사용하는 실행가능한 예제를 포함하라.

## Goroutine Lifetimes

새로운 goroutine 을 생성할때, 언제 종료할지 혹은 종료 할것인지 아닌지를 확실하게 하라.

goroutine 은 채널에서 blocking 되어 leak 이 일어날 수 있는데, garbage collector 도 종료시키지 않는다.

goroutine 이 leak 이 발생하지 않아도, 필요없는 goroutine 을 남겨두는것은 진단하기 힘든 문제를 야기할 수 있다. 닫힌 채널에 전달하는 것은 panic 을 일으킨다. 아직 사용중인 input 을 결과가 필요하지 않을때 변경하는것은 data race 를 일으킬 수 있다. 가변적인 시간동안 goroutine 을 남겨두는 것은 예상치 못한 메모리 사용을 일으킨다.

concurrent code 들을 단순하게 작성하여, goroutine 의 lifetime 이 명백하게 유지하고, 그럴 수 없다면 언제/왜 고루틴이 종료되는지에 대해 명세하라.

## Handle Errors

`_` 를 사용해서 error 를 버리지 말라. (많은 경우 버리고있었다)
만약 function 이 error 를 return 하는 경우에는 function 이 실행에 성공했는지를 확실하게 하라. 정말로 예외적인 경우에는 panic 하라. (panic 관련해서는 의견들이 많이 갈리는 것 같은데, uber style guide 에도 panic 은 main 외에서는 일으키지 말라고 하는것에 동의한다)

## Imports

name 충돌을 피하기 위해 import renaming 하는것을 지양하라. 좋은 패키지 이름들은 renaming 이 필요하지 않다. 충돌이 발생했을때, `most local` 혹은 `project-specific` 한 import 를 rename 하라.

빈 줄로 imports 들을 정렬하는데 standard library 는 첫번째에 둬라. (`goimports` 쓰면 자동으로 된다)

## Import Blank

오직 side effect 를 위해서 import 하는 패키지들은 main 에서 혹은 test 에서만 import 되어야 한다.

## Import Dot

import dot 은 import 한 패키지를 현재 package 의 scope 에 선언하는 것을 뜻한다. 

현재 package 의 요소인지, import . 으로 참조한 package 의 요소인지 불명확하기 때문에 사용하지 않는게 좋다.

test 작성시, circular dependency 가 일어나는 경우에 한해서 `import .` 을 사용할 수 있다. 

## In-Band Errors

C 와 같은 언어들에서는 에러를 표현하거나 잘못된 결과로 `-1`, `null` 과 같은 return 을 하는게 흔했다.

Go 는 multiple return 을 지원하므로 client(caller) 가 value 체크를 하는것 대신 전달한 value 가 valid 한지 아닌지를 알려주는 추가적인 변수를 전달해야 된다.

## Indent Error Flow

normal code 를 최소한의 indent 로 유지하고 error handle 한것을 미리 처리하라. 이렇게 하면 normal 한 경우를 빠르게 스캔해볼 수 있어 코드의 가독성을 높일 수 있다.

## Initialisms

Initialisms 와 acronyms 는 일관되게 처리한다. `URL` 은 `URL` 혹은 `url` 로 쓰고, `Url` 는 하지 않는다. 

`ID` 는 `identifier` 의 줄임말이기 때문에, `appId` 대신 `appID` 로 사용하라.

protocol buffer 로 생성된 코드는 이 규칙의 예외이다. 

## Interfaces 

코드 아래에 직역한 내용도 있으나 내용이 다소 복잡하고 직역을 제대로 하지 못하여 코드를 좀 더 풀어서 설명해보았다.

```golang
// code snippet-1

package consumer  // consumer.go

type Thinger interface { Thing() bool }
func Foo(t Thinger) string { … }

package consumer // consumer_test.go

type fakeThinger struct{ … }
func (t fakeThinger) Thing() bool { … }

if Foo(fakeThinger{…}) == "x" { … }
```

위의 코드(snippet-1)를 보면 Thinger interface 는 concrete struct 의 구현부터 정의된게 아니라, consumer 가 public API 인 `Foo` 에서 사용하기 위함이다. 또한 `Foo` 를 테스트하기 위해 mocking 기능을 갖는 `fakeThinger` 를 추가해서 테스트 할 수 있다.

```golang
// DO NOT DO IT!!!
// code snippet-2
package producer

type Thinger interface { Thing() bool }

type defaultThinger struct{ … }
func (t defaultThinger) Thing() bool { … }

func NewThinger() Thinger { return defaultThinger{ … } }
```

위의 코드(snippet-2) 를 보면, producer 가 `Thinger` 라는 interface 를 정의하여 `NewThinger` concrete type 인 `defaultThinger` 를 return 하며 `Thinger` interface 로  return 하고 있는데, 이렇게되면 확장성도 떨어진다. 차라리 consumer 가 `Thing()` 을 mocking 할 수 있게 열어주는게 낫다.

아래는 직역한 내용이다.

Go interfaces 는 보통 interface 를 사용하는 package 에서 사용된다. 그 interface 를 구현하는 package 에 속하지 않는다.

Interface 를 구현하는 package 는 pointer 혹은 struct 타입인 concrete type 을 return 해야된다. 그래야 새로운 methods 들이 광범위한 리팩토링 없이 추가 가능해진다.

interface 를 정의할때, API 를 구현하는 측에서 mocking 할 수 있게 고려하지 말고, public 한 API 를 이용해서 테스트 할 수 있게 고려하라.

사용되기 전에 interface 를 미리 정의하지 말라: 현실적인 예제 없이는 그 interface 가 필요한지 조차 알기 힘들다.

Interface 를 return 하지 말고, concrete type 을 return 해서 사용자가 interface 를 이용해서 mock 할 수 있게 하라.

## Line Length

N 자 미만으로 한 줄을 유지하라 라는 규칙은 아니지만, 느낌대로.. 잘 잘라보라는 다소 모호한 설명이다. 

아래는 직역이다.

Go 에서 한 줄에 글자 제한을 정해놓은건 없지만, 불편하게 긴 문장은 피하라. 비슷한 맥락에서 길게 적어도 가독성이 괜찮을때는 짧게 적으려고 line break 를 일부러 넣을 필요는 없다. 

타당한 갯수의 파라미터와 적절히 짧은 변수명을 갖고있다면  wrapping 이 필요 없을 것이다. 길게 쓰는것은 이름을 길게 만들때 주로 나타나므로 긴 이름을 없애는게 큰 도움이 된다.

다르게 표현하면, 줄의 길이가 길어서 자르지 말고, 의미가 바뀔때 줄을 나눠라. 그래도 너무 길다면, 이름이나 의미들을 바꿔보면 좋은 결과가 있을것이다.

line 길이에 대한것은, function 길이에 대한 것과 동일하다. 함수 선언도 N 자 이상으로 선언하지 말라는 규칙은 없지만, 작은게 반복되는 함수나 너무 긴 함수는 분명 존재하고 함수의 범위를 자르는 것이 해법이다. 

## Mixed Caps

mixed capital 을 사용하여 `MAX_LENGTH` 가 아닌 `maxLength` 가 옳다.

이는, unexported variable 은 소문자로 시작하기 때문이다.

## Named Result Parameters

return 값을 named 로 미리 함수 signature 에 선언해놓는 것을 의미한다.

함수가 2~3개의 같은 타입의 값을 return 할때 결과의 의미가 모호한 경우 이름을 붙여주는게 좋을 수 있다. 단순히 function 내에서 변수를 선언하는걸 없애기 위해서 named result 를 사용하진 말라.

구체적인 예시는 아래와 같이, 단순히 `float64` 2개를 return 하는것 보다는 `lat`, `long` 처럼 naming 한 결과를 내려주는게 의미가 더 명확하다.

```golang
func (f *Foo) Location() (float64, float64, error)

is less clear than:

// Location returns f's latitude and longitude.
// Negative values mean south and west, respectively.
func (f *Foo) Location() (lat, long float64, err error)
```

함수가 짧으면 Naked return 하는건 문제가 없다. 만약, 중간 사이즈 정도 된다면 return value 를 명확히 하라. 같은 맥락에서, 단순히 naked return 을 하기위해 named result 하는건 피하라.

명확한 문서화는 함수 구현에서 1~2줄 줄이는 것보다 항상 더 중요하다.

마지막으로, 지연된 closure(deferred closure) 에서 결과 parameter 를 바꾸기 위해서 naming 하는게 필요할 수 있다. (이 부분은 무슨말인지 이해가 되지 않는다)


## Naked Returns

이건 그냥 의미의 설명만 있었다. 아래 코드를 보면 이해가 된다.
```golang
func split(sum int) (x, y int) {
	x = sum * 4 / 9
	y = sum - x
	return
}
```

## Package Comments

godoc 에 나타날 다른 주석들과 마찬가지로, package comments 는 빈칸 없이 package 구문과 인접해서 나타나야된다.

대문자로 시작하고, 올바른 영어로 작성하라.

## Package Names

모든 references 는 package 이름으로 될 것이므로, package name 은 제외해도 된다. 예를들어 `chubby.ChubbyFile` 대신 `chubby.File` 을 쓰라.

`util`, `common`, `misc`, `api`, `types`, `interfaces` 와 같이 무의미한 package name 은 피하라.

## Pass Values

몇 바이트 아끼겠다고 function 에 pointer 를 넘기지 말라. `string` 이나 `io.Reader`와 같은 interface 를 넘기는 경우 굳이 `*string`, `*io.Reader` 로 넘길 필요 없이 고정된 크기의 값으로 넘길 수 있다. 다만, 큰 struct 이거나 작은 struct 이지만 크기가 커질 경우에 적용되지는 않는다.

## Receiver Names

receiver name 으로 1~2글자면 충분하다. `me`, `this`, `self` 와 같은 이름은 쓰지말라. Go 에서 receiver 는 그저 또 하나의 파라미터이기 때문에 그에 맞게 명명 되어야 한다. receiver 의 용도가 명확하고 문서화 될 필요가 없기 때문에, 이름이 그렇게 설명적일 필요는 없다.

`familiarity admits brevity` (자주 등장하면 간결해도 된다).

어떤 receiver 에서 `c` 로 불렀다면 다른 receiver 는 `cl` 로 호출하지 말라.

## Receiver Type

새로운 go programmer 들에게 특히, value receiver, pointer receiver 중 어떤걸 쓸지 정하는건 어렵다. 의심스러우면 pointer 를 사용하는데 value receiver 가 말이 되는 경우들이 있다. 

몇가지 가이드라인이 있다.

- receiver 가 map/func/chan 이면 pointer 를 사용하지 말라. 만약 receiver 가 slice 이고 method 가 reslice 하거나 reallocate 하지 않는다면 pointer 를 쓰지 말라.
- receiver struct 가 `sync.Mutex` 혹은 비슷한 동기화 필드를 갖는다면 복사를 방지하기 위해 pointer 이어야 한다.
- 만약 receiver 가 큰 strut 이거나 array 이면 pointer receiver 가 더 효율적이다. 
- 이 함수가 concurrent 하게 호출 됐거나 할때 receiver 를 변경할 수 있는가? value type 은 method 가 호출됐을때 receiver 의 복사본을 만들기 때문에 외부에서의 업데이트가 receiver 에 반영되지 않는다. 만약 이러한 변경이 원본 receiver 에 확인이 필요하다면 receiver 는 pointer 여야 한다.
- 만약 receiver 가 struct/array/slice 혹은 변하는 것을 가리키고 있다면, pointer receiver 를 사용하라.
- method 가 receiver 를 바꿀 필요가 있다면, pointer 여야 한다.
- receiver 가 작은 array 이거나, value type 의 struct 이고, 변하지 않는 필드와 pointer 를 갖고있지 않은 단순한 int/ string 과 같은 값이라면 value receiver 이 말이 된다. value receiver 는 만들어질 수 있는 garbage 를 줄이는 역할을 한다. 만약 value method 로 전달되고 heap 에 할당 할 필요 없이 stack 에 복사할 수 있다. 프로파일링 해보기 전에 value receiver 를 선택하지 말라.
- pointer 나 struct 하나 정하고, receiver type 을 섞지 말라.
- 마지막으로, 모르겠으면 pointer receiver 를 사용하라.

## Synchronous Functions

동기적인 function 을 비동기적인 function 보다 선호하라.

동기적인 함수는 고루틴을 내부에 유지해서 leak 이나 data race 를 더 쉽게 이해할 수 있게한다. 또한 테스트도 더 쉽다: 동기화할 필요 없이 데이터를 전달해서 테스트해볼 수 있기 때문이다.

만약 caller 가 concurrency 가 필요하다면 여러 고루틴을 이용해서 쉽게 할 수 있다. 불필요한 concurrency 를 호출자가 없애는건 힘들거나 가끔 불가능할때도 있다. (내부에 비동기로 구현되어있으면 caller 가 대응하기 힘들다는 의미로 이해했다)

## Useful Test Failures

테스트들은 input, 실제 받은 결과(got), 기대했던 결과(expected), 무엇이 잘못됐는지에 대한 도움되는 메세지를 남기며 실패해야된다. `assertFoo` helper 를 쓰는데 혹할 수 있지만 쓸모있는 에러 메세지를 남기는것이 중요하다. 당신이나 당신의 팀이 테스트하고 있는게 아니라고 생각하라.

전형적인 go test 는 아래와 같이 실패한다.
```go
if got != tt.want {
	t.Errorf("Foo(%q) = %d; want %d", tt.in, got, tt.want) // or Fatalf, if test can't test anything more past this point
}
```

몇몇 테스트 frameworks 들은 `expected != got` 처럼 쓰지만 go 는 그렇지 않다. (순서를 바꿔서 표현한다는 뜻)

많은 테스트르를 하고싶을때 `table-driven` test 를 작성하고 싶을 수 있다.

혹은 같은 함수를 다른 input 으로 호출하는 것을 각기 다른 function 으로 wrap 하는 아래와 같은 방법도 있다. 아래 방법은 둘다 `testHelper` 를 테스트하는 거지만 각기 다른 test function 으로 나눴고, 이 테스트가 실패하면 서로 다른 function name 으로 실패할 것이다.

```go
func TestSingleValue(t *testing.T) { testHelper(t, []int{80}) }
func TestNoValues(t *testing.T)    { testHelper(t, []int{}) }
```

## Variable Names

변수명은 짧아야 된다. 

기본 법칙: 선언과 사용이 멀 수록 이름이 설명적이어야 된다. method receiver 의 경우는 한글자/두글자면 충분하다. loop 인덱스나 readers 는 각각 단일 문자인 `i`, `r` 로 사용 가능하다. 흔치 않은 상황이나 global variables 는 더 설명적인 명명이 필요하다.