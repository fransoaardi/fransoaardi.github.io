---
layout: post
title: Leetcode(1)
date: 2021-02-22 21:55:00 +0900
categories: [Algorithm]
tags: [algorithm, python]
toc: true
---

# introduction
python 에 익숙해질겸 leetcode 를 풀고있는데, 생각할만한 부분, 기록해놓을 만한 부분들을 모았다.

# problems

## 125: valid-palindrome 

- https://leetcode.com/problems/valid-palindrome

1. `map(func, input)` 결과를 `list()` 를 이용해서 바로 list 로 바꿔버릴 수 있다.

```python
lowered = "".join(list(map(tolower, s)))
```

2. `map` 할 필요 없이, `filter` 를 이용해도 됐다. 
3. `str.isalnum` 으로 alphanumeric filter 가능하고, str.lower 쓰면 굳이 map 이 필요없었다.

```python
s1 = ''.join(filter(str.isalnum, s))
```