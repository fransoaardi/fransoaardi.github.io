---
layout: post
title: 몇몇 알고리즘 문제들 다시풀기
date: 2021-05-14 18:30:00 +0900
categories: [Wiki]
tags: [algorithm]
toc: true
comments: true
---

1. string parser 만들기 

```
3[ac]2[b] = acacacbb
3[[a]2c]4[b] = accaccaccbbbb
```

이 문제를 푸는데 상당히 오랜 시간이 걸렸다. 계속 재귀적으로 풀어보려고 하다가 수렁에 빠졌던것 같은데, `]` 를 만날때까지 stack 에 차곡차곡 쌓았다가, `[` 한칸을 해소할때까지 pop 하며 처리하는 식의 반복으로 해결할 수 있었다.

```python
from collections import deque


class Solution:
    def __init__(self):
        pass

    def solve(self, raw):
        def is_digit(c):
            return c and c in "123456789"

        tmp = deque()
        for s in raw:
            if s != "]":
                tmp.append(s)
                continue

            inner = deque()

            while True:
                if tmp:
                    x = tmp.pop()
                    if not is_digit(x) and not x == "[":
                        inner.append(x)
                    elif is_digit(x):
                        last_char = inner.pop()
                        inner.extend(int(x) * [last_char])
                    else:   # x == "["
                        inner.reverse()
                        tmp.append("".join(inner))
                        inner = deque()
                        break

        # 최종적으로 [] 이 제거된 부분을 정리해준다.
        ans = []
        num = 1
        for x in tmp:
            if is_digit(x):
                num = int(x)
            else:
                ans.append(num * x)
                num = 1
        return "".join(ans)


if __name__ == "__main__":
    s = Solution()
    print(s.solve("3[ac]2[[b]b]"))
    print(s.solve("3[[a]2c]4[b]"))
    print(s.solve("3[[a]2c]"))
    print(s.solve("3[ac]2[b]"))
    print(s.solve("[]"))
    print(s.solve("3[[ac]b][x]"))
    print(s.solve("2ac"))

```

2. bi-gram, synonym 처리하기

[a quick brown fox jumps, [["fast", "quick"], ["quick", "haste"]]]

output
["a quick", "quick brown", "fast brown", "haste brown", "brown fox", "fox jumps"]

3. list merge 하기

```
아래와 같은 구간이 있다고 할때, 구간을 합쳐보는 문제이다.

[[1,2], [2,3]] = [[1,3]]
[[1,3], [2,4], [8,9]] = [[1,4], [8,9]]
```

```python
"""
    x1 y1
            x2 y2       # 이 경우에만 ans 에 append 해준다
                        # 아래 두 경우는 두 경계를 합쳐준다. x1 > x2 일 수는 없다(sort 를 하기 때문에)
    x1     y1           
        x2     y2
    x1               y1
        x2       y2
"""

class Solution:
    def __init__(self):
        pass

    def solve(self, d):
        d.sort(key=lambda x: (x[0], x[1]))

        ans = [d[0]]
        for cur_x, cur_y in d[1:]:
            last_x, last_y = ans[-1]
            if cur_x > last_y:
                ans.append([cur_x, cur_y])
            else:
                ans[-1] = [min(last_x, cur_x), max(last_y, cur_y)]

        return ans


if __name__ == "__main__":
    s = Solution()
    print(s.solve([[1, 2], [2, 3]]))
    print(s.solve([[1, 4], [2, 3]]))
    print(s.solve([[1, 4], [2, 3], [5, 6]]))
    print(s.solve([[1, 3], [2, 4], [8, 9]]))
    print(s.solve([[1, 3], [8, 9], [2, 4]]))
```
