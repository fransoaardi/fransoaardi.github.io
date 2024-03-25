---
layout: post
title: Why Generics? by Ian Lance Taylor
date: 2022-02-23 00:29:00 +0900
categories: [Wiki]
tags: [golang]
toc: true
comments: true
---

## Introduction

Golang 1.18 Beta 2 가 release 됐다는 소식의 [go blog](https://go.dev/blog/go1.18beta2) 를 읽으며, 아직 깔아서 해보기는 솔직히 귀찮고 가장 큰 변화인 generics 관련 글이나 한번 더 읽어보자 싶어서 아래의 짧은 영상과 글을 찾아보게 되었다. 상세한 내용을 정리하진 않고, 왜 generics 를 구현하게 되었고 개발을 위해서 중요하게 생각했던 디자인 철학을 중점으로 가볍게 정리했다.

확실히 2019년 갓 draft 가 나온 당시의 영상이라 그런지, 이미 구현이 다 되어 1.18 release 를 앞둔 지금과는 조금 다르다. 물론 컨셉은 거의 비슷하다. 예를들어 `contract` 라는 keyword 로 언급된 내용은 1.18 에서는 `constraints` 로 불리고 있고, `()` 대신 `[]` 로 generic 관련 type 이 전달된다. 

## Link

- https://go.dev/blog/why-generics

## Content

- Python 과 같은 dynamic typed languages 는 어떤 타입이든 사용가능한 function 을 만드는데 크게 제약이 없다. 
- Golang 은 static typed language 이라 이러한 function 을 작성할 수 없다. 
- Java, C++ 와 같은 다른 static typed languages 들은 그래서 generic 을 제공한다.
- Golang 은 제공하지 않는다. 다만 interface 를 이용해서 작성을 간접적으로 할 수 있게 한다. 

> Interface types in Go are a form of generic programming. They let us to capture the common aspects of the different type and express them as a method.

- 하지만, 많은 양의 코드를 작성해야됐고 이를 지양하려고 한다.
- 혹은 다른 방식으로는 interface 를 이용해서 static type 의 장점을 잃어버리기도 했다.
- `reflect` 를 이용할 수 있지만, 상당히 어색하고 결국 assertion 이 필요한데 static type 의 장점을 잃어버린다.
- 혹은 code generator 를 이용하는 방법이 있다.
- 모든 방식들은 상당히 어색했고, 결국 개발자는 자신이 원하는 함수를 따로 작성하고, 작성한 함수를 위한 테스트케이스를 작성하고 그 테스트를 주기적으로 실행해줘야 된다.

> Generics can give us a powerful building blocks, so let us share codes and build programs more easily.

Generics 를 추가하는것은 분명 큰 장점이 있지만, 여러가지 이유들 중 Golang 을 간단하고 명료하게 유지하고 짧은 빌드 시간과 빠른 실행시간 등등을 보존하는 선에서 개발되어야 된다고 얘기한다.