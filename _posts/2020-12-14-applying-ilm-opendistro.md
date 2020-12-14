---
layout: post
title: applying-ilm-opendistro
date: 2020-12-14 14:00:00 +0800
categories: [Wiki]
tags: [elk, aws]
toc: true
---

# introduction
- 본 글은 opendistro 에 ilm rule 적용을 다룬다.

- AWS elasticsearch SaaS(Service as a service) 를 사용하는데, 사실 aws opendistro 는 elastic 의 kibana 와는 갈래가 나뉘었다. 

> 전쟁의 서막
- [elastic 의 발표](https://www.elastic.co/kr/blog/on-open-distros-open-source-and-building-a-company)
  - 우리는 품질을 높이기 위한 노력인데 억울하다..

- [aws 의 발표](https://aws.amazon.com/ko/blogs/korea/keeping-open-source-open-open-distro-for-elasticsearch/)
  - 우리는 opensource 로 공개했을뿐이다.

- 매일 index 가 생성되는데, index management 가 없으니, 계속해서 데이터를 쌓아놓긴 메모리/용량이 아쉽고, 수동으로 saved objects 를 삭제하며 관리할 수는 없으니 ilm(index lifecycle management) 을 적용하기로 하였다.

- Note: opendistro 에서는 ism(index state management) 라는 명칭을 쓰는것 같은데, ilm / ism 동일한 의미로 사용하였다.

# environment
- index template 를 미리 생성해놓았다.
- k8s 에서 발생하는 로그를 fluentbit 를 이용해서 daily 로 `log-index-2020.12.14` 같은 패턴으로 index 를 생성하는데, 00시 기준 처음 들어온 raw_data 기준으로 새로운 index 가 생성된다. es(elasticsearch) 는 새로 들어온 데이터를 기준으로 template 에 정의된 mapping을 따르고, 나머지 필드에 대해서는 dynamic mapping 이 된다. 

# progress
## ilm creation
- ilm 은 `hot`, `warm`, `delete` 3가지 state 를 가질 수 있는, FSM(finite state machine) 형태로 정의할수있다.
- data access 를 효율적으로 하기위해 hot/warm 등으로 분리하려는 의도보다는 생성된지 30일(임의 설정 값)이 지난 index 들을 delete 하기 위한 의도로 사용하는것이라 아래와 같이 state 를 `"hot"`, `"delete"` 만 정의하였다.

- policy 생성 json
> 
```json
{
  "policy": {
    "description": "delete an index after 30 days",
    "default_state": "hot",
    "schema_version": 1,
    "states": [
      {
        "name": "hot",
        "actions": [],
        "transitions": [
          {
            "state_name": "delete",
            "conditions": {
              "min_index_age": "30d"
            }
          }
        ]
      },
      {
        "name": "delete",
        "actions": [{
            "delete": {}
        }]
      }
    ]
  }
}
``` 

## template 에 ilm 적용
- 아래 settings 를 그대로 PUT 하면 안되고, 미리 생성했던 template 내용을 포함시켜서 execute 한다. (patch 가 아닌 overwrite 개념으로 동작한다)
```
PUT _template/your-index
{
  "index_patterns": ["your-index-*"],
  "settings": {
    "opendistro.index_state_management.policy_id": "newly-created-ilm-policy-name"
  }
}
```
- 내일 00시 기준 새로 생성될 index 는 위의 policy 가 적용될것이다.

## 미리 생성된 index 에 policy 적용
- 미리 생성된 index 는 policy 가 적용되어있지 않고, checkbox 선택 후, 직접 index policy 적용이 필요하다.

- 적용 전

![indices not managed by ilm](/assets/img/posts/2020-12-14-ilm/indices-not-managed.png){: width="350" class="normal"}

- 적용 후

![indices managed by ilm](/assets/img/posts/2020-12-14-ilm/indices-managed.png){: width="350" class="normal"}

- `hot -> delete` transition 이 일어난 index

![indices in transition by ilm](/assets/img/posts/2020-12-14-ilm/indices-in-transition.png){: width="350" class="normal"}


# references
> `elastic` vs `aws` 전쟁의 서막
- https://www.elastic.co/kr/blog/on-open-distros-open-source-and-building-a-company
- https://aws.amazon.com/ko/blogs/korea/keeping-open-source-open-open-distro-for-elasticsearch/

> template 레벨에 ilm 적용 설정
- https://discuss.opendistrocommunity.dev/t/can-you-automatically-manage-indices/2034

> ism example policy
- https://opendistro.github.io/for-elasticsearch-docs/docs/ism/policies/#example-policy
