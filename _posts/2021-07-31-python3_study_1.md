---
layout: post
title: Python3 새롭게 알게 된 것들에 대한 기록 (1)
date: 2021-07-31 20:14:00 +0900
categories: [Wiki]
tags: [python]
toc: true
comments: true
---

# introduction

좋게 말하면 Golang 에 진심이었지만 Python3 를 겉핥기식으로만 공부한건 아닐까 하는 생각이 들고 이렇게는 혹독한 세상에서 살아남을 수 없겠다는 생각이 들어서 python3 의 내실을 다져보기로 했다. 이 글을 읽게될 여러분들중 대다수분들에게는 별것 아닌 내용일 수 있지만 공부하며 새롭게 알게된 사실들을 기록 차원에서 정리했다.

계속해서 내용을 추가할 예정이다.

# contents

목차

- `int class` 는 `base` keyword argument 를 넘길 수 있다
- `__init__` 은 진정한 의미의 생성자가 아니다
- `instance method` 가 아닌 `staticmethod` 를 사용했을때의 장점
- class 에 `__slots__` 를 정의해서 attribute 를 제한할 수 있다

## int class 는 `base` keyword argument 를 넘길 수 있다

궁금해서 int class 정의를 읽다가 아래와 같은 사실을 알 수 있었다. 
아래 내용은 `builtins.py` 에서 발췌했다.
```python
def __init__(self, x, base=10): # known special case of int.__init__
    """
    int([x]) -> integer
    int(x, base=10) -> integer
    
    Convert a number or string to an integer, or return 0 if no arguments
    are given.  If x is a number, return x.__int__().  For floating point
    numbers, this truncates towards zero.
    
    If x is not a number or if base is given, then x must be a string,
    bytes, or bytearray instance representing an integer literal in the
    given base.  The literal can be preceded by '+' or '-' and be surrounded
    by whitespace.  The base defaults to 10.  Valid bases are 0 and 2-36.
    Base 0 means to interpret the base from the string as an integer literal.
    >>> int('0b100', base=0)
    4
    # (copied from class doc)
    """
    pass
```

즉, 아래처럼 x 가 number 가 아니거나 base 가 주어진 경우에 x 는 주어진 base 로 숫자를 표현하는 string/byte/bytearray literal 인 경우 int 로 반환된다.

```python
a = int('0b110', base=0)
b = int('0x10', base=0)
c = int('2a', base=16)
d = int('12', base=5)
print(a, b, c, d)
# Output:
# 6 16 42 7
```

## `__init__` 은 진정한 의미의 생성자가 아니다

reference: [https://dev.to/delta456/python-init-is-not-a-constructor-12on](https://dev.to/delta456/python-init-is-not-a-constructor-12on)

python3 에 `__new__` 가 있다. `__init__` 은 `self` 에 attribute 들을 할당하고 사실은 `__init__` 호출 전에 `__new__` 가 먼저 호출되어 allocate 를 하고 그 instance 를 전달해주고 있었다. 결국 아래처럼 구현된 것으로 볼 수 있다.

```python
class P1:
    def __new__(cls, *args, **kwargs):
        print('p1 new')
        return super().__new__(cls)
```

궁금해서 실험을 좀 더 해봤는데, 만약 return 부분에 `3` 혹은 `None` 과 같은 다른 타입을 리턴한다면 `__init__` 은 `__new__` 에서 return 한 타입의 `__init__` 이 호출된다. 그래서 아래와 같은 코드를 작성해서 확인해봤다.

```python
class P1:
    def __new__(cls, *args, **kwargs):
        print('p1 new')
        # return super().__new__(cls) # 이 부분을 변경해본다(Output one)
        # return P2()                 # 이 부분을 변경해본다(Output two)

    def __init__(self):
        print("p1 init")
        pass

class P2:
    def __init__(self):
        print("p2 init")
        pass
# Output one:
# p1 new
# p1 init
# Output two:
# p1 new
# p2 init
```

## `instance method` 가 아닌 `staticmethod` 를 사용했을때의 장점

reference: [https://stackoverflow.com/a/22589883](https://stackoverflow.com/a/22589883)

1. self 가 필요없다. (staticmethod 는 첫번째 인자로 self 를 받지 않는다)
2. 새로운 instance 가 생성될때마다 method 를 생성하지 않아도 되어서 메모리 사용량을 줄인다.

이부분이 재밌는데, 아래처럼 코드를 작성해서 확인해보니, static method 는 한 class 에 유일하다.

```python
class P1:
    def instance_method(self):
        print('instance method')

    @staticmethod
    def static_method():
        print('static method')


if __name__ == "__main__":
    one = P1()
    two = P1()
    print('instance methods are the same:', one.instance_method is two.instance_method)
    print('static methods are the same:', one.static_method is two.static_method)

# Output:
# instance methods are the same: False
# static methods are the same: True
```

3. staticmethod 는 self 에 접근하지 않는게 명확해서, 코드의 가독성을 높인다. 


## class 에 `__slots__` 를 정의해서 attribute 를 제한할 수 있다

Golang 개발을 하다가 python 을 봤을때 생각하는 장점이자 단점은, class attribute 설정이 자유롭다는데 있다. javascript 도 그런데 예상치 못한 필드가 예상치 못한 코드에서 추가되거나 했을때 코드 가독성이 많이 떨어지는 경험이 많았다. 하지만 class 에 `__slots__` 를 설정해놓으면, class attribute 를 제한할 수 있다는 사실을 새로 알게됐다.

아래는 연습문제로 '기사'의 클래스를 갖는 게임 캐릭터를 만들어본 것인데, 예를들어 `health, mana, armor` 외에는 attribute 를 제한했고, `hello` 라는 attribute 설정에 실패한 내용이다.
```python
class Knight:
    __slots__ = ["health", "mana", "armor"]

    def __init__(self, *, health: float, mana: float, armor: float) -> None:
        self.health = health
        self.mana = mana
        self.armor = armor

    def __repr__(self) -> str:
        return f"{self.health} {self.mana} {self.armor}"


if __name__ == "__main__":
    k1 = Knight(health=100, mana=50, armor=30)
    k1.hello = 'world'
    print(k1)

# Output:
# k1.hello = 'world'
# AttributeError: 'Knight' object has no attribute 'hello'
```