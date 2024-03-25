---
layout: post
title: 잘 잊어버리는 유용한 linux commands 
date: 2021-09-06 10:49:00 +0900
categories: [Wiki]
tags: [linux]
toc: true
comments: true
---

- tcp/8000 포트 사용중인 pid 를 찾는다

```
$ lsof -t -i tcp:8000 | xargs kill -9
```
