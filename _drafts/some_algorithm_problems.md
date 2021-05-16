---
layout: post
title: 몇몇 알고리즘 문제들 다시풀기
date: 2021-05-14 18:30:00 +0900
categories: [Wiki]
tags: [algorithm]
toc: true
comments: true
---

1. parser 만들기 

3[ac]2[b] = acacacbb
3[[a]2c]4[b] = accaccaccbbbb

[ 를 만나면 recursive call 을 하거나
[ 를 만나






2. bi-gram, synonym 처리하기

[a quick brown fox jumps, [["fast", "quick"], ["quick", "haste"]]]

output
["a quick", "quick brown", "fast brown", "haste brown", "brown fox", "fox jumps"]

3. list merge 하기

[[1,2], [2,3]] = [[1,3]]
[[1,3], [2,4], [8,9]] = [[1,4], [8,9]]



