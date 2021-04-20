golang map 이 있다고 가정했을때, concurrent 한 readonly usage 에 문제가 없진 않을까? 에서 시작한 삽질

https://golang.org/doc/articles/race_detector

map 에서 "a" key 만 계속 읽고, "b" 에 write 를 하는순간
map 이 grow 하면서 새로운 allocation 이 일어난다면 corrupt ? 됐다고 볼수있을까?