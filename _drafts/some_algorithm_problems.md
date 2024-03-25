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

# table of contents

1. string parser 만들기 
2. bi-gram, synonym 처리하기
3. list merge 하기
4. 문자열을 정해진 규격에 맞게 나누기
5. Google Kickstart 2021: Cutting Intervals

# contents

## 1. string parser 만들기 

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

## 2. bi-gram, synonym 처리하기

[a quick brown fox jumps, [["fast", "quick"], ["quick", "haste"]]]

output
["a quick", "quick brown", "fast brown", "haste brown", "brown fox", "fox jumps"]

## 3. list merge 하기

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

## 4. 문자열을 정해진 규격에 맞게 나누기

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

## 5. Google Kickstart 2021: Cutting Intervals

이 [문제](https://codingcompetitions.withgoogle.com/kickstart/round/00000000004361e3/000000000082b933)는 google kickstart 2021 에서 만났다. 물론 시간 안에는 풀지 못했다. 공부하는 취지에서 종료 후 Analysis 를 읽고 다시 풀어보는 시간을 가졌다.

이 문제는 n 개의 구간과 특정 구간을 자르는 시행(cut) 의 횟수가 정해졌을때 어떻게하면 이 구간의 갯수를 최대로 할 수 있겠는가 하는 문제이다.

예를들어 (1,3), (2,4), (1,4) 가 주어졌을때 x=2 인 부분을 자르면, 1번 과 3번의 구간에 1칸씩이 추가되어 2개가 늘어난다. 같은 방법으로 x=3 인 부분을 자르면 2번과 3번 구간에 1칸씩 추가되어 또 2칸이 늘어난다. x=2 혹은 x=3 이 아닌 구간을 자르면 늘어나는 구간은 없다.

이 문제에서 주요했던 내용은, n 값은 10**5 까지라서 유효한 구간이었던데에 반해, c 는 10**18, 왼쪽과 오른쪽 값인 L 과 R 은 10**13 까지 늘어날 수 있었다. 따라서 특정 위치인 x 를 자르면 몇칸이 늘어나는지에 대한 값을 모든 x 에 대해 계산하는건 불가능했다. 

대신, 어디든 자르면 `0 <= j <= n` 을 만족하는 j 칸이 늘어나기 때문에, 구간으로 접근해서 j 칸이 늘어나는 위치는 몇곳인지 (특정한 위치는 필요하지 않다)를 계산하는 방법으로 접근하면 됐다.

하지만 cut 하는 c 의 상한이 정해져 있기 때문에, greedy 한 접근방식으로 한번의 cut 으로 가장 많은 칸이 늘어나는 곳을 cut 조건에 맞게 줄여가면서 계산하는 로직이 유효했다. 이 부분을 코드로 나타내면 아래와 같다.

```python
ans = 0
for v in ag_key:
    # c 가 음수가 되면 곤란하여 max(0,c) 구문을 추가한다. 물론 중간에 c 가 음수면 break 하는 방법도 있다.
    ans += v * min(aggregated[v], max(0, c)) 
    c -= aggregated[v]
return ans
```

아래는 작성한 코드의 전문이다. 코드는 크게 3부분으로 나뉘어있다.
```
1. 입력된 interval 의 시작과 끝점을 각각 처리해서 누적을 위한 dict 로 나타내는 부분
2. 잘랐을때 j 만큼의 조각이 추가되는 구간의 갯수를 dict 로 계산하는 부분 (aggregated)
3. greedy 하게 효율 좋은 구간부터 잘라나가는 부분
```

```python
class Solution:
    def solve(self, c, intervals):
        # 시작점+1 은 +1, 끝점은 -1 을 할당한 set 을 생성한다.
        # n 개의 interval 에 cut 을 하면 0~n 까지의 cut 이 추가될 수 있다.
        # A0~ An 까지의 list 를 만들고, j 가 큰 값부터, j * Aj 의 합을 더해가다가 답을 내면 된다.
        interv_dict = dict()
        for st, ed in intervals:
            if st+1 in interv_dict:
                interv_dict[st+1] += 1
            else:
                interv_dict[st+1] = 1

            if ed in interv_dict:
                interv_dict[ed] -= 1
            else:
                interv_dict[ed] = -1
        interv_dict_keys = sorted(interv_dict.keys())

        aggregated = dict()
        before = 0
        j = 0
        for v in interv_dict_keys:
            if j in aggregated:
                aggregated[j] += (v - before)
            else:
                aggregated[j] = (v - before)
            before = v
            j += interv_dict[v]

        ag_key = sorted(aggregated.keys(), reverse=True)

        # print(aggregated, ag_key)

        ans = 0
        for v in ag_key:
            ans += v * min(aggregated[v], c)
            c -= aggregated[v]
            c = max(c, 0)
        return ans


if __name__ == "__main__":
    """
    n: number of intervals
    c: number of cuts
    intervals = [(1, 3), (2, 4), (1, 4)]

    Sample Input:
    n = 3
    c = 3
    intervals = [(3, 7), (1, 5), (4, 7)]

    Expected Output:
    Case #1: 7
    """
    s = Solution()

    t = int(input())
    for i in range(1, t+1):
        n, c = [int(s) for s in input().split(" ")]
        intervals = []
        for _ in range(n):
            st, ed = [int(s) for s in input().split(" ")]
            intervals.append((st, ed))

        ans = s.solve(c, intervals)
        print(f"Case #{i}: {ans+n}")

```

