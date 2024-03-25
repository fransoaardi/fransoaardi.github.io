---
layout: post
title: kibana & opendistro index lifecycle management(ilm) 적용하기
date: 2020-12-14 14:00:00 +0900
categories: [Wiki]
tags: [elk, aws]
toc: true
comments: true
---

# changelog

- 2020-12-14: 문서작성
- 2021-04-06: es, opendistro ilm 관련 API 경로 차이 관련 내용 추가

# introduction

- 본 글은 kibana & opendistro 에 ilm rule 적용을 다룬다.
- AWS elasticsearch service SaaS(Service as a service) 를 사용하는데, 사실 aws opendistro 는 elastic 의 kibana 와는 갈래가 나뉘었다. 

> 전쟁의 서막

- [elastic 의 발표](https://www.elastic.co/kr/blog/on-open-distros-open-source-and-building-a-company)
  - 우리는 품질을 높이기 위한 노력인데 억울하다..
- [aws 의 발표](https://aws.amazon.com/ko/blogs/korea/keeping-open-source-open-open-distro-for-elasticsearch/)
  - 우리는 opensource 로 공개했을뿐이다.
- 매일 index 가 생성되는데, index management 가 없으니, 계속해서 데이터를 쌓아놓긴 메모리/용량이 아쉽고, 수동으로 saved objects 를 삭제하며 관리할 수는 없으니 ilm(index lifecycle management) 을 적용하기로 하였다.
- Note: opendistro 에서는 ism(index state management) 라는 명칭을 쓰는것 같은데, ilm / ism 동일한 의미로 사용하였다.

# environment

- aws 에서 제공하는 elasticsearch SaaS 와 aws EC2 에 직접 설치한 elasticsearch & kibana 두 가지 종류를 사용중이다.
- index template 를 미리 생성해놓았다.
- EC2 에 직접 띄운 elasticsearch 에는 filebeat 를 이용해서, SaaS 에는 k8s 에서 발생하는 로그를 fluentbit 를 이용해서 daily 로 양쪽 다 `log-index-YYYY.MM.DD` 패턴으로 index 를 생성하는데, 00시 기준 처음 들어온 raw_data 기준으로 새로운 index 가 생성된다. es(elasticsearch) 는 새로 들어온 데이터를 기준으로 template 에 정의된 mapping을 따르고, 나머지 필드에 대해서는 dynamic mapping 이 된다. 
- 매일 생성된 index 가 3달치 가까이 쌓여있고, 이를 일정 주기마다 지우려고 한다.

# progress

## opendistro

### ilm creation

- ilm 은 `hot`, `warm`, `delete` 3가지 state 를 가질 수 있는, FSM(finite state machine) 형태로 정의할수있다.
- data access 를 효율적으로 하기위해 `"hot"`/`"warm"` 등으로 분리하려는 의도보다는 생성된지 30일(임의 설정 값)이 지난 index 들을 delete 하기 위한 의도로 사용하는것이라 아래와 같이 state 를 `"hot"`, `"delete"` 만 정의하였다.

> policy 생성 json: IM 이라고 적힌 index management 라는 화면에서 gui 를 이용해서 직접 생성했다.

```json
{
  "policy": {
    "description": "delete an index after 30 days",
    "default_state": "hot",
    "schema_version": 1,
    "states": [{
      "name": "hot",
      "actions": [],
      "transitions": [{
          "state_name": "delete",
          "conditions": {
              "min_index_age": "30d"
          }
      }]
    },
    {
        "name": "delete",
        "actions": [{
            "delete": {}
        }]
    }]
  }
}
``` 

- 내용추가: elasticsearch doc 에 관련 내용을 검색해보면 이 [es doc 링크](https://www.elastic.co/guide/en/elasticsearch/reference/master/ilm-put-lifecycle.html)가 나온다. 하지만, 이 기능은 x-pack 기능이고, opendistro 에서는 REST API 경로가 다르다. [opendistro doc 링크](https://opendistro.github.io/for-elasticsearch-docs/docs/im/ism/)에 따르면 es 는 `PUT _ilm/policy/<policy_id>`, opendistro 는 `PUT _opendistro/_ism/policies/<policy_id>` 로 다르다. opendistro 에서 es 경로처럼 호출하면, 401(`Unauthorized`) error 가 발생하는데, permission 을 잘못준 건 분명 아니고, 경로가 잘못됐기 때문이다. 

dev tools 를 이용해서 `GET _opendistro/_ism/policies/<policy_id>` 해보면 아래와 같이 잘 만들어진 policy 를 확인해볼 수 있다.

```json
{
  "_id" : "delete-index-after-30d",
  "_version" : 1,
  "_seq_no" : 0,
  "_primary_term" : 1,
  "policy" : {
    "policy_id" : "delete-index-after-30d",
    "description" : "delete an index after 30 days",
    "last_updated_time" : 1607917600048,
    "schema_version" : 1,
    "error_notification" : null,
    "default_state" : "hot",
    "states" : [
      {
        "name" : "hot",
        "actions" : [ ],
        "transitions" : [
          {
            "state_name" : "delete",
            "conditions" : {
              "min_index_age" : "30d"
            }
          }
        ]
      },
      {
        "name" : "delete",
        "actions" : [
          {
            "delete" : { }
          }
        ],
        "transitions" : [ ]
      }
    ]
  }
}
```

### template 에 ilm 적용

- 아래 settings 를 그대로 PUT 하면 안되고, 미리 생성했던 template 내용을 포함시켜서 execute 한다. (patch 가 아닌 overwrite 개념으로 동작한다)

```json
PUT _template/your-index
{
  "index_patterns": ["your-index-*"],
  "settings": {
    "opendistro.index_state_management.policy_id": "newly-created-ilm-policy-name"
  }
}
```
- 내일 00시 기준 새로 생성될 index 는 위의 policy 가 적용될것이다.

### 미리 생성된 index 에 policy 적용
- 미리 생성된 index 는 policy 가 적용되어있지 않고, checkbox 선택 후, 직접 index policy 적용이 필요하다.

- 적용 전 (Managed by Policy 값이 No 이다)

![indices not managed by ilm](/assets/img/posts/2020-12-14-ilm/indices-not-managed.png)

- 적용 후 (Managed by Policy 값이 Yes 이다)

![indices managed by ilm](/assets/img/posts/2020-12-14-ilm/indices-managed.png)

- `hot -> delete` transition 이 일어난 index (State 가 `delete` 로 변경됐다, 이후 제거됐다)

![indices in transition by ilm](/assets/img/posts/2020-12-14-ilm/indices-in-transition.png)

## kibana

### ilm creation

- opendistro 와는 다르게 `settings` > `Elasticsearch` > `Index Lifecycle Policies` menu 에 web-console 이 제공됨
- opendistro 와 다르게 Hot, Warm, Cold, Delete phase 이렇게 4가지 (opendistro 에 `Cold` 는 없었다) 가 존재한다. 역시나 Delete 만 사용하고 싶어서, `30 days from index creation` 을 설정하였다.

### template 에 ilm 적용

- UI 가 다소 이질적이었는데, `Index Lifecycle Policies` 에서 방금 생성한 policy 우측에 `Actions` > `Add policy to index template` 를 통해 적용한다. Index template 에 apply policy 할 줄 알았는데, 접근방식이 반대였다.

### 미리 생성된 index 에 policy 적용

- 이것도 특이했는데, 기존에 생성된 indices 에 한꺼번에 적용은 불가능했고, checkbox 1개씩 선택해서 따로따로 적용만 가능했다.
- `checkbox` > `Manage index` 클릭 > `Add lifecycle policy` 적용하면 바로 1달 지난 index 에 대해서는 `delete` phase 로 즉시 변했다. 

![before adding ilm in kibana](/assets/img/posts/2020-12-14-ilm/index-management-before-kibana.png)

![after adding ilm in kibana](/assets/img/posts/2020-12-14-ilm/index-management-after-kibana.png)

# references

> `elastic` vs `aws` 전쟁의 서막

- https://www.elastic.co/kr/blog/on-open-distros-open-source-and-building-a-company
- https://aws.amazon.com/ko/blogs/korea/keeping-open-source-open-open-distro-for-elasticsearch/

> template 레벨에 ilm 적용 설정

- https://discuss.opendistrocommunity.dev/t/can-you-automatically-manage-indices/2034

> ism example policy

- https://opendistro.github.io/for-elasticsearch-docs/docs/ism/policies/#example-policy
