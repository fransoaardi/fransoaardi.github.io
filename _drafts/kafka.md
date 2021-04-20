kafka 관련 논문
---

abstract
- online, offline message consumption 둘다 적합한 시스템
> 각각의 차이가 뭐지?

1. introduction

- user activity events, operation metrics 두가지가 전형적인 로그 데이터.
- Hadoop/ data warehouse 에 로그 넣어놓는 방식이 offline consumption 인가봄

- kafka is distributed and scalable, and offers high throughput. 

Kafka provides an API similar to a messaging system and allows applications to consume log events in real time.


2. Related work

이전부터 여러 기업용 메세징 시스템은 비동기 데이터 처리의 이벤트 버스로 중요한 역할을 해왔음.
근데 로그 가공에 좋지 않은 이유들이 몇개 있음
- 전송 보장을 위해 많은 기능들이 있어서, 로그 수집에는 과한 경우가 있음
> 로그 몇개 잃는다고 큰일나지 않는데, 이거때문에 API 와 구현의 복잡도가 올라감
- 쓰루풋을 중요하게 생각하지 않았다. 
> TCP/IP roundtrip 이 필요함 
- 분산 처리에 좋지 않다
- 대부분이 로그가 바로바로 처리될거라고 가정했다. 이런 제품들은, 로그가 바로 처리되지 않으면 성능이 나빠진다.





