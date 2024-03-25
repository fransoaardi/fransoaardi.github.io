---
layout: post
title: Golang yaml 라이브러리 연구
date: 2022-10-12 21:00:00 +0900
categories: [Wiki]
tags: [golang]
toc: true
comments: true
---

# Introduction

최근에 업무중에 궁금한 생각이 들었다. `yaml` 명세에는 보통 string 도 `""` 를 추가하지 않고 작성을 하게된다. 
예를들면 아래와 같다.
```yaml
a: hello
```

`boolean` 을 의도하는 경우에는 `true, false` 등을 작성하면 되겠는데 개발 요구사항중에 왜 그런줄은 모르겠지만 `"true", "false"` 라는 string 을 저장하고 싶다는 내용이 있었다. 

이 글에서는 `decode` 하는 경우 `boolean` 으로 오해받기 쉬운 string 의 처리법과 일부 tag 사용에 대해서 가볍게 다룬다.
많은 시간을 할해한 분석은 아니고 decoder 의 동작 원리, parse 는 어떻게 진행되는지 에 대한 내용은 다루지 않는다. 
다음 기회에 좀 더 공부해서 다뤄보려고 한다. 

# Contents 

서론에서 얘기했던 `"true", "false"` 를 저장하는 것은 물론 큰 문제가 아니다. 아래와 같이 `string` 으로 `type` 을 지정해버리면 `yaml raw` 를 `Unmarshal` 하게 되면 당연히 A 의 값으로는 `"true"` 가 할당될 것이다. 

```golang
import (
	"fmt"

	"gopkg.in/yaml.v2"
)

func main() {
	raw := `
a: true
`

	type Simple struct {
		A string `yaml:"a"`
	}

	simple := Simple{}
	if err := yaml.Unmarshal([]byte(raw), &simple); err != nil {
		fmt.Println(err)
	}
	fmt.Printf("simple: %v, type of Simple.A: %T\n", simple, simple.A)
}

// $ go build && ./yaml
// simple: {true}, type of Simple.A: string
```

코드 분석은 `decode.go` 의 `func (d *decoder) unmarshal(n *node, out reflect.Value) (good bool) {` 부분을 따라가보면, `resolve.go` 까지 확인하면 알 수 있다. 

우선 `yaml.Unmarshal` 하는 경우에 struct 의 type 이 명시적이지 않은 경우에 `resolve` 를 호출하게 된다. 

> decode.go

```golang
func (d *decoder) scalar(n *node, out reflect.Value) bool {
	var tag string
	var resolved interface{}
	if n.tag == "" && !n.implicit {
		tag = yaml_STR_TAG
		resolved = n.value
	} else {
		tag, resolved = resolve(n.tag, n.value)
		if tag == yaml_BINARY_TAG {
			data, err := base64.StdEncoding.DecodeString(resolved.(string))
			if err != nil {
				failf("!!binary value contains invalid base64 data")
			}
			resolved = string(data)
		}
	}
```

이때 `resolve` 는 `resolveMap` 이라는 map 에서 현재 값을 기준으로 yaml 값의 type 을 추론하게 된다. 
`resolveMap` 은 `resolve.go` 의 `init()` 에 의해 값이 할당되는데, 아래의 `resolveMapList` 의 값들로 채워진다. 

> `resolve.go`

```golang
func resolve(tag string, in string) (rtag string, out interface{}) {
    // ... 
    if hint != 0 && tag != yaml_STR_TAG && tag != yaml_BINARY_TAG {
		// Handle things we can lookup in a map.
		if item, ok := resolveMap[in]; ok {
			return item.tag, item.value
		}
}

var resolveMapList = []struct {
		v   interface{}
		tag string
		l   []string
	}{
		{true, yaml_BOOL_TAG, []string{"y", "Y", "yes", "Yes", "YES"}},
		{true, yaml_BOOL_TAG, []string{"true", "True", "TRUE"}},
		{true, yaml_BOOL_TAG, []string{"on", "On", "ON"}},
		{false, yaml_BOOL_TAG, []string{"n", "N", "no", "No", "NO"}},
		{false, yaml_BOOL_TAG, []string{"false", "False", "FALSE"}},
		{false, yaml_BOOL_TAG, []string{"off", "Off", "OFF"}},
		{nil, yaml_NULL_TAG, []string{"", "~", "null", "Null", "NULL"}},
		{math.NaN(), yaml_FLOAT_TAG, []string{".nan", ".NaN", ".NAN"}},
		{math.Inf(+1), yaml_FLOAT_TAG, []string{".inf", ".Inf", ".INF"}},
		{math.Inf(+1), yaml_FLOAT_TAG, []string{"+.inf", "+.Inf", "+.INF"}},
		{math.Inf(-1), yaml_FLOAT_TAG, []string{"-.inf", "-.Inf", "-.INF"}},
		{"<<", yaml_MERGE_TAG, []string{"<<"}},
	}
```

위의 코드에서 `"y", "Y", "yes", "Yes", "true", ...` 등 여러 값들이 `yaml_BOOL_TAG` 라는 `bool` 로 추론될 값들이다.
즉, yaml raw 에 값 부분에 `true` 라고 전달할때 만약 `Unmarshal` 될 객체에 명시적인 type 이 지정되지 않은 상태라면 (예를들어 `any`, `interface{}` 와 같은) 우선적으로 `bool` 이라고 간주할 것이다. 

아래의 예제는 이 [링크](https://github.com/go-yaml/yaml#example) 에서 가져온 예제를 조금 수정 한 것이다.
`any` 로 정의한 여러 변수들에 `true, True, "true", "True", !!str true` 를 각각 테스트 해보았다.

```golang
package main

import (
	"fmt"
	"log"

	"gopkg.in/yaml.v2"
)

var data = `
a: true
b: True
c: "true"
d: "True"
e: !!str true
`

type T struct {
	A any
	B any
	C any
	D any
	E any
}

func main() {
	t := T{}

	err := yaml.Unmarshal([]byte(data), &t)
	if err != nil {
		log.Fatalf("error: %v", err)
	}
	fmt.Printf("--- t:\n%#v\n\nA: %T B: %T C: %T D: %T E: %T\n", t, t.A, t.B, t.C, t.D, t.E)

	d, err := yaml.Marshal(&t)
	if err != nil {
		log.Fatalf("error: %v", err)
	}
	fmt.Printf("--- t dump:\n%s\n\n", string(d))

	m := make(map[interface{}]interface{})

	err = yaml.Unmarshal([]byte(data), &m)
	if err != nil {
		log.Fatalf("error: %v", err)
	}
	fmt.Printf("--- m:\n%#v\n\nA: %T B: %T C: %T D: %T E: %T\n", m, m["a"], m["b"], m["c"], m["d"], m["e"])

	d, err = yaml.Marshal(&m)
	if err != nil {
		log.Fatalf("error: %v", err)
	}
	fmt.Printf("--- m dump:\n%s\n\n", string(d))
}

// Output:
// --- t:
// main.T{A:true, B:true, C:"true", D:"True", E:"true"}

// A: bool B: bool C: string D: string E: string
// --- t dump:
// a: true
// b: true
// c: "true"
// d: "True"
// e: "true"


// --- m:
// map[interface {}]interface {}{"a":true, "b":true, "c":"true", "d":"True", "e":"true"}

// A: bool B: bool C: string D: string E: string
// --- m dump:
// a: true
// b: true
// c: "true"
// d: "True"
// e: "true"
```

결과는 예상대로 `true, True` 는 `boolean` 으로 해석되었고, `"true", "True"` 는 `string` 으로 해석되었다.
`!!str true` 는 `strTag       = "!!str"` 값에서 `tag` 를 이용하여 명시적으로 표현하는 방식으로 사용해봤는데, 역시나 `string` 으로 해석되었다.
번외로, `e: !!bool "true"` 라고 한다면, 명시적으로 `"true"` 라고 하긴 했지만, `boolean` 값으로 해석이 된다. 

# Conclusion

솔직히 이 내용으로 이렇게 심각하게 할 것도 없고 혼란이 생길 것 같은 `"true"` 와 같은 명세를 하느니 다른 말로 치환하는게 낫다.
예를들면, 실제로 필요했던 내용은 "비가 온다", "비가 오지 않는다" 라는 항목이었는데 `"raining","not_raining"` 과 같이 명시적인 string 형태로 작성할 수도 있는 문제다. 

하지만, 특수한 상황에 `gopkg.in/yaml.v2` 라이브러리는 어떤식으로 동작하는지 문득 궁금해서 분석해보게 되었다.

# References

대부분의 코드는 아래의 코드에서 복사해왔다. `v3` 이 나왔지만 `v2` 를 분석한 이유는 회사의 dependency 가 `v2` 였기 때문이다. 물론 `v3` 을 분석하는게 더 좋았을거란 뒤늦은 후회를 한다.

- https://github.com/go-yaml/yaml/tree/v2.4.0