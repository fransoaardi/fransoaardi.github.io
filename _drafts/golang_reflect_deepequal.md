---
layout: post
title: gRPC-with-golang
date: 2020-12-11 01:20:00 +0900
categories: [Wiki]
tags: [grpc, golang]
toc: true
comments: true
---


- `reflect.DeepEqual` 관련 궁금증이 생겼다.


```golang
package main

import (
	"fmt"
	"reflect"
)

type s struct{
	i int
}

func main() {
	m1 := make(map[string]interface{})
	m2 := make(map[string]interface{})
	
	mapInMap := make(map[string]string)
	mapInMap["x"] = "valueX"
	mapInMap["y"] = "valueY"
	
	m1["a"] = 1
	m1["b"] = "valueB"
	m1["c"] = s{
		i: 0,
	}
	
	m2["a"] = 1
	m2["b"] = "valueB"
	m2["c"] = s{}
	
	
	// invalid operation: m1 == m2 (map can only be compared to nil)
	// fmt.Println("m1 == m2", m1 == m2)
	fmt.Println("reflect.DeepEqual(m1,m2)", reflect.DeepEqual(m1, m2))
	
	m1["d"] = mapInMap
	m2["d"] = mapInMap
	
	fmt.Println("m1:",m1)
	fmt.Println("m2:",m2)
	fmt.Println("reflect.DeepEqual(m1,m2)", reflect.DeepEqual(m1, m2))
	
	
	m2["d"] = map[string]string{
		"x": "valueX",
		"y": "valueY",
	}
	
	fmt.Println("m1:",m1)
	fmt.Println("m2:",m2)
	fmt.Println("reflect.DeepEqual(m1,m2)", reflect.DeepEqual(m1, m2))
}
```