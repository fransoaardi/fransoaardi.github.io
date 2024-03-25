---
갑자기 왜 로그설명을 하세요? (많이 심심하구나?)
- 네..
- + 분명히 누군가 다음에 또 물어볼것같아서, 미리합니다.

---
로그란?
> observability by (opentracing, opentelemetry: https://opentelemetry.io)
    - tracing: distributed tracing, pinpoint failures and performance issues.
    - metrics: quantitative information about processes running inside the system, including counters, gauges, and histograms.
    - logging: provides insight into application-specific messages emitted by processes.

> business 적인것, 에러만 본다 ok/error

- 어떤걸 로그로 남기려고하는가?
    - 이걸 남기면 어떤걸 확인할 수 있는가?
    - 어떤걸 dashboard 로 보고싶은지?
        > aggregatable, searchable 

---
> prometheus(scrape) + grafana : k8s 환경의 표준 

ELK: elasticsearch + logstash + kibana
- es: 저장소 
- logstash: 로그 가공+shipping 
- kibana: visualize

예전 모델
 사용자 -> ~!@~!@~ -> logstash -> es 
                                    <- kibana - 사용자
권장 사항
사용자 -> ~!@~!@~ -> beats(filebeat, fluentbit) -> kafka- logstash -> es 
                                    <- kibana - 사용자
AIRS 
 사용자 -> ~!@~!@~ -> beats -> es 
                                    <- kibana - 사용자

---
es SaaS (opendistro): 번역앱, 셔클 
    > AWS credential HEADER 
    
es ec2 (on-premise): edith 
    > 
    
---
filebeat (edith-services 참고)
> x-pack 관련 이슈가 있어서 fluentbit 로 갈아탔습니다.



(내가 로그를 발생시켜서 -> 이걸 es 에 저장하고 -> kibana 를 통해서 보고싶다.)
---
0. 파일을 어떻게 쪼개볼까 >인덱스 구성 어떻게할지 (생각)
1. 로그 포맷 잡는다 json, apache format ETC .. 
2. 잡은 포맷으로 템플릿정의를 한다
3. 템플릿에 맞게 파일로 떨궈서, 파일을 보내게 filebeat/fluentbit 와 같은 log-shipper 설정/실행
    > ansible script 한번 읽고, 
4. 들어간걸 확인한다
5. kibana 보이도록 설정한다

---
es 의 type 
- https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping-types.html
    keyword, text, numbers, boolean 등.. 
- aggregatable, searchable 두가지 측면

{
    "abc": "apple", // text 256 미만까지는 keyword .. 
    "def": 123
}   // 50%

{   // 50%
    "abc": 123
}

"keyword", "text"
    // aggregatable, searchable
"keyword": "abc"    
"text": "강남역 가자", "사당역 가자" .........

"nlp", "platform", "ux", "kjh"
// platform 연령 평균

// "keyword" "text" : 
    "abc" "keyword" 
    "abc.raw"   "text" 

---

index template
- https://cody-ecm.autoever.com/pages/viewpage.action?pageId=223516172

---
로그를 보내면 es/kibana 에서 어떤일이 일어나는지?

1. 로그를 보냄 (shipper -> es)

00시00분00초 이후로 
29 

2. es 에 이미 생성된 index 가 있는지 확인
    있음: index 에 저장
    없음: 새로운 index 생성
        index-template 있는지 확인
            있음:
                있는부분 mapping 을 따름
            없음:
                dynamic 하게 mapping 하고, index 생성

3. kibana 에서 지정한 pattern 에 해당하는 es indices 에서 query 가능

kibana (visualize < dashboard)
edith-common-log-* (kib)
    > edith-common-log-2020.12.29 ... (elas)

---
fluentbit (edith-gateway 참고)
    - input, filter, parser, output  나눠서 pipeline 정의 
        https://docs.fluentbit.io/manual/concepts/data-pipeline/input
    
    - input: source file, 보통 Tail 을 사용 
    - output: elasticsearch
    - filter: 특정 필드 추가하거나, 필터하거나 등등.., parser 적용도 filter 로 한다.



input1 (edith-common)   -> filter (json parse) -> filter   -> es (edith-common-log)
                         /
input2 (edith-dm-log)   /                                         es (edith-dm-log)

--- 

filebeat (edith-services)
