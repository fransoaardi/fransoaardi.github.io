---
layout: post
title: goroutine 은 언제 죽는가
date: 2021-01-15 14:29:00 +0900
categories: [Wiki]
tags: [golang]
toc: true
comments: true
---

# introduction
- `The Go programming language (by 앨런도너반, 브라이언 커니건 지음)` 책을 읽다가, 아래와 같은 코드를 보았다.

> code from 257p `The Go programming language (by 앨런도너반, 브라이언 커니건 지음)`

```golang
func mirroredQuery() string {
    responses := make(chan string, 3)
    go func() { responses <- request("asia.gopl.io" )} ()
    go func() { responses <- request("europe.gopl.io" )} ()
    go func() { responses <- request("americas.gopl.io" )} ()
    return <- responses // 가장 빠른 응답 반환
}
```

- `responses` 는 cap 3 인 buffered channel 이므로, 3개가 다 들어와도 문제가 안생기고, 가장 빨리 결과가 들어온 것을 return 할것이라 특이한건 없어보일 수 있다.
- 하지만, `request` func 의 구현에 따라 큰 차이가 있을 수 있다. 예를들면, `http.DefaultClient` 혹은 `http.Client{}` 와 같이 생성한 client 는 `Timeout` 필드가 `0` 이고, 이는 `Timeout` 이 없는것으로 처리되어, `context` 의 `Timeout` 을 활용하지 않는다면, 결과를 돌려받을때까지 pending 된다. 그렇다면 위의 `mirroredQuery` 의 goroutine 은 끝나지 않을 수도 있다.
- 항상 궁금했던 내용인데, 알고있기로는 `main` 이 끝날때는 모든 goroutine 은 종료되고, 다른 func 에서 호출한 goroutine 은 해당 func 가 끝나도, goroutine 은 종료되지않고, 실행이 완료될때까지 남아있는것으로 알고있는데, 이를 확인하고싶었다.
- 추가로, Dave Cheney 가 발표했던 [영상](https://www.youtube.com/watch?v=nok0aYiGiYA&ab_channel=GopherAcademy)에서 소개한 `trace` tool 을 이용해서 goroutine 의 생애를 좀 더 파보기로 했다.

# process
## presumption

- 아래와 같이, `main` 에서 `mirroredQuery` 대신 비슷한 형태의 축소된 예제를 작성하고 실행한다고 가정한다.
```
func main(){
    a()
}
```

## control

- `a` 에서 실행하는 3회의 goroutine 은 
1. 모두 순식간에 끝나는, 또한 끝날것이 보장되는 함수라고 가정한다. 
2. 종료가 보장되지 않는 함수라고 가정한다.

# experiments

```golang
package main

import (
	"fmt"
	"github.com/pkg/profile"
	"time"
)

func main() {
	defer profile.Start(profile.TraceProfile, profile.ProfilePath(".")).Stop()  // trace.out 을 떨어뜨리는 helper function
	fmt.Println(a())
	time.Sleep(time.Second)
}

func a() string {
	done := make(chan bool)
    defer close(done)  
    // defer 에 done close 시켜서 goroutine 의 종료를 강제하여 종료가 보장되는 환경 설정
    // 지면관계상 종료가 보장되지 않는 환경은, 따로 작성하지 않는다. done 전달을 없애고, 아래 infinite 의 select 구문을 없애면 된다.
    // Note: 3개 function 이 모두 infinite() 라면 deadlock 이 발생할 것이다.
	resp := make(chan string, 3)
	go func() { resp <- infinite(1, done) }()
	go func() { resp <- infinite(2, done) }()
	go func() { resp <- notinfi() }()
	return <-resp
}

func infinite(a int, done chan bool) string {
	defer fmt.Println("I'm done", a)
	fmt.Println("infinite occurred", a)
	for {
		select {
        case <-done:    // a() 호출이 끝나서 done 이 close 되면 이 func 는 종료될것이다.
                        // select 구문을 없애면 infinite 는 종료되지 않을 function 이 된다. 
			return ""
		}
	}
}

func notinfi() string {
	return ""
}
```

- go `trace` tool 을 사용해서 확인해보았다.

```bash
$ go build && ./goroutine_trace && go tool trace trace.out
```

- `view trace` 메뉴로 들어가면 trace 결과가 나오는데, `?` 로 help 메뉴 볼 수 있다.
- `w`, `s` 로 확대/축소를 할수있다.
- 아주 짧은 시간 안에 처리가 되었고, 상대적으로  `time.Sleep(time.Second)` 가 길어서, 비율이 이상한데, 앞에 500microsec 구간을 확인해보면 아래와 같다.

![goroutine_terminated](/assets/img/posts/2021-01-15-goroutine_lifecycle/goroutine_terminate.png)

- 이미지에 zoom 이 덜되어 생략됐지만, G8,G9,G10 이 실행 후 종료되었고, 더 이상 실행되지 않는 모습이다. 
- 반면에, 종료가 보장되지 않는(`for {}` 로 block 했다) goroutine 의 경우 아래와 같다.

![goroutine_not_terminate](/assets/img/posts/2021-01-15-goroutine_lifecycle/goroutine_not_terminate.png)

- G8, G9 가 main 이 종료되는 1sec 전까지 계속 실행되고 있는것을 확인할 수 있다.
- 좀 더 확대해보면 아래와 같다.

![goroutine_not_terminate_zoom](/assets/img/posts/2021-01-15-goroutine_lifecycle/goroutine_not_terminate_zoom.png)

- G10 은 종료되었고, `a()` 는 진작 return 을 했을것이고, G8, G9 가 230microsec 즈음부터 시작하여 위에서 본 것처럼 1sec 까지 실행되고있고, main 이 종료되어서야 종료되었다.

# lessons learned

- 실험결과에 따르면 main func 가 종료되면 모든 goroutine 은 종료된다.

- main 이 아닌, func 내에서 호출된 goroutine 은 해당 func 가 종료되었더라도, 종료되지않고, 독립된 lifecycle 을 가진다.

- Dave Cheney 는 [글](https://dave.cheney.net/2016/12/22/never-start-a-goroutine-without-knowing-how-it-will-stop)에서 goroutine 의 종료시점을 명확하게 파악하지 않으면 조용히 leak 이 발생할 수 있을것이라고 얘기하고 있다. goroutine 은 return 없는 func 를 실행해서, 중간에 종료가 되지 않을거라고 생각하기는 어려운데, 꼼꼼히 확인해봐야된다. 

- 개인적으로 Dave Cheney 의 [발표영상](https://www.youtube.com/watch?v=nok0aYiGiYA&ab_channel=GopherAcademy)을 보며, trace tool 을 한번 써보고싶다 라고 생각했는데, 얕게라도 써봐서 즐거웠다.

# references

- Dave Cheney 의 발표영상:
    - https://www.youtube.com/watch?v=nok0aYiGiYA&ab_channel=GopherAcademy
- Never start a goroutine without knowing how it will stop by Dave Cheney
    - https://dave.cheney.net/2016/12/22/never-start-a-goroutine-without-knowing-how-it-will-stop
- The Go programming language (by 앨런도너반, 브라이언 커니건 지음) 257p