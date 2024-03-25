---
layout: post
title: Golang 의 standard project layout
date: 2021-05-17 20:00:00 +0900
categories: [Wiki]
tags: [golang, convention]
toc: true
comments: true
---

# introduction

https://github.com/golang-standards/project-layout

# 

처음부터 이렇게 하진 말고, 이 구조는 큰 프로젝트에 어울릴 수 있는 구조다.
```
If you are trying to learn Go or if you are building a PoC or a simple project for yourself this project layout is an overkill. Start with something really simple instead (a single main.gofile andgo.mod is more than enough
```

정의되어 있다고 다 쓸 필요는 없다. (필요한걸 갖다써라)
```
Just because it's there it doesn't mean you have to use it all. 
```

# contents

## `/cmd`

```
Main applications for this project.

The directory name for each application should match the name of the executable you want to have (e.g., /cmd/myapp).

Don't put a lot of code in the application directory. If you think the code can be imported and used in other projects, then it should live in the /pkg directory. If the code is not reusable or if you don't want others to reuse it, put that code in the /internal directory. You'll be surprised what others will do, so be explicit about your intentions!

It's common to have a small main function that imports and invokes the code from the /internal and /pkg directories and nothing else.
```

다른데서 재사용 가능한 코드는 pkg 에 넣고, 재사용 못하게 할 코드는 internal 에다 넣는다.
다른사람들이 정말로 어떻게 가져다쓰게 될지는 예상 불가능하기 때문에, 재사용여부에 대해 명시적으로 밝히는게 좋다.

## `/internal`

private app 과 library code 를 포함한다. 
go comiler 가 강제(외부에서 internal 하위 코드는 가져올 수 없다)하는 layout pattern 이다.

`internal` 은 꼭 최상위에 있어야될 필요가 없고, 또한 여러개를 가져도 된다.

`/internal/app` 에 app 코드 넣고, `/internal/pkg` 에는 공유된 코드를 넣을 수도 있다.

## `/pkg`

외부 app 에서 사용해도 괜찮은 라이브러리 코드를 담는다.

코드 규모가 커져서 추가적인 nesting 이 필요한 경우가 아니라면 꼭 해야되는건 아니다. non-go app components 가 늘어나면 고려해보라.

internal, pkg 디렉토리에 대해 잘 설명이 되어있는 글이라고 한다.
- https://travisjeffery.com/b/2019/11/i-ll-take-pkg-over-internal/

- https://www.youtube.com/watch?v=PTE4VJIdHPg&ab_channel=GopherConEurope

## `/vendor`

## `/api`

openAPI, swagger, JSON schema, protocol definition 등을 담는다.

## `/web` 

static web assets, server side templates, spa 등을 담는다.

## `/configs`
- config file template, default configs
- confd or consul-template files

## `/init`

System init (systemd, upstart, sysv) and process manager/supervisor (runit, supervisord) configs.

## `/scripts`

Scripts to perform various build, install, analysis, etc operations.

## `/build`

빌드, ci 관련된 부분을 `/build/package`, `/build/ci` 처럼 등록함

## `/deployments`

`/deploy` 로 사용하기도 하고, `docker-compose`, `k8s/helm`, `mesos` 등.. 배포 관련 스크립트들을 넣는다.

## `/test`

## `/docs`
- godoc generated, design docs, user docs

## `/tools`
이 프로젝트를 위한 tool. `/pkg`, `/internal` 에서 코드 import 가능함

## `/examples`
예제 모음

## `/third_party`
- 외부 helper tool

## `/githooks`
- git hook 모음

## `/assets`
- image, logo, 기타 모음

## `/website`
- project website 인데, `/web` 이랑 용도가 겹치는것 같음

## don't: `/src`
- `java` 의 패턴을 가져오지 말라
- go module 이전에 `$GOPATH/src` 로 관리됐었는데, 그러면 `/src/${pjtName}/src` 로 중복되니까 하지 말라
