---
layout: post
title: 몇몇 알고리즘 문제들 다시풀기
date: 2021-05-14 18:30:00 +0900
categories: [Wiki]
tags: [algorithm]
toc: true
comments: true
---

# introduction

살면서 풀어내야만 하는 algorithm 문제들이 있다. 못풀고 냅두면 계속 생각나고 아쉬움이 남는다.
이러한 아쉬움과 미련을 떨쳐내기 위해, 시간을 들여서라도 문제를 해결한 내용들을 모았다.
문제를 다시 풀며 약오름과 문제 앞에서 한없이 나약한 느낌을 지울 수는 없지만 그래도 조금은 발전된 모습을 위해 수많은 시행은 꼭 필요하다고 믿는 편이다. 

# contents

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

4. 문자열을 정해진 규격에 맞게 나누기

문제 설명: `Lorem ipsum ...` 과 같은 raw string 이 제공이 되고, 
한 줄에 l 이상이 되지 않게 단어를 최대한 붙이는것으로 한다.
단어의 단위는 공백을 구분하는것으로 한다. (`Lorem ipsum, dolor` 는 `Lorem`, `ipsum,`, `dolor` 세 단어 이다) 
l 보다 긴 한 단어는, 넘어갈 수 있다.
가운데 공백은 왼쪽에 우선순위를 주면서 최대한 균등하게 나눈다 (공백을 포함한게 l 을 넘어선 안된다)

아래는 결과 예시이다.

```golang
// Lorem   ipsum   dolor
// sit amet, consectetur 
// adipiscing  elit, sed
// do   eiusmod   tempor
// sdasdasddasdsdasdasdasdsa
// dasds         sdsdsds
// ... 
```

아래와 같이 코드를 작성하여 해결할 수 있었다. 
우선 두가지 파트로 나뉘는데, 첫번째는 적당한 단어의 합으로 만들어주는것이고 두번째는 padding 처리를 하는 부분이다.
적당한 단어의 합을 만들기 전에 padding 처리를 하는것은 불가능하기 때문에 두단계가 꼭 필요하다고 생각한다.

```go
package main
import (
    "fmt"
    "strings"
)

// "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation // ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint /// occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum."

func main() {
    length := 40
    raw := `Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud exercitation // ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur. Excepteur sint /// occaecat cupidatat non proident, sunt in culpa qui officia deserunt mollit anim id est laborum.`
    // raw := `Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor incididunt`
    
    ans := solve(raw, length)
    for _, a := range ans{
        fmt.Print(a)
    }
    
}  

func solve(raw string, l int) []string {
    tmp := splitter(raw, l)
    return addPad(tmp, l)
}

func splitter(raw string, l int) []string {
    split := strings.Split(raw, " ")
    
    cnt := 0
    var ans, tmp []string
    for _, s := range split {
        if cnt + (1+len(s)) <= l {
            tmp = append(tmp, s)
            cnt += (1+len(s))
        }  else {
            if len(tmp) == 0 {
                ans = append(ans, s)
                cnt = 0
            } else {
                ans = append(ans, strings.Join(tmp, " "))    
                tmp = nil
                tmp = append(tmp, s)
                cnt = len(s)    
            }
        }
    }
    return ans
}

func addPad(in []string, l int) []string{
    var ans []string
    for _, s := range in {
        sp := strings.Split(s, " ")
        if len(s) >= l || len(sp) <= 1{
            ans = append(ans, s + "\n")
        } else {
            need := l - len(s)
            spaces := make([]string, len(sp)-1)
            
            for i := 0; i<len(spaces); i++ {
                spaces[i] = "|"
            }
            
            for i:= 0; i<need; i++ {
                spaces[i%(len(sp)-1)] += "%"
            }
            
            tmp := ""
            for i, sIn := range sp {
                tmp += sIn
                if i < len(spaces) {
                    tmp += spaces[i]
                }
            }
            ans = append(ans, tmp+"\n")
        }
    }
    return ans
}
```