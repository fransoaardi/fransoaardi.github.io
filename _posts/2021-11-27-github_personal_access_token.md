---
layout: post
title: GitHub Personal Access Token 설정
date: 2021-11-27 00:06:00 +0900
categories: [Wiki]
tags: [github]
toc: true
comments: true
---

# Introduction

개발 장비를 새로 세팅을 하면서 `github.com` keychain 에 개인 계정을 직접 입력하지 않고, GitHub pages (꼭 GitHub pages 만을 위한 내용은 아니고, 모든 GitHub repo 에 해당하는 내용이다) 를 위한 personal access token(이하 PAT) 을 깔끔하게 설정할 수 있는 방법이 있을까 하여 찾아봤다. 아래는 꽤나 당연하고 간단한 내용들이지만 기록을 위한 용도로 작성하였다.

참고: PAT 관리는 아래에서 확인할 수 있다.
- https://github.com/settings/tokens 에 직접 접근
- 우측상단 프로필 클릭 > Settings > Developer settings > Personal access token

# Steps

[Stackoverflow](https://stackoverflow.com/a/18936804) 를 찾아보니 2가지 방법이 있었다.

1. `git clone` 할 때 부터 personal access token 을 이용하는 방법
1. `git remote` 설정에 personal access token 을 설정하도록 변경하는 방법

간단히 정리해보면, `<USERNAME>:<PAT>` 를 추가할 뿐이다. 이 방법은 GitHub actions 의 workflow 구성시에 private 한 repo 를 참조할 일이 필요할때, private repo 에 접근 가능한 PAT 를 발급하고, url 을 `<USERNAME>:<PAT>` 를 포함한 형태로 rewrite 하는 방식 과 동일한 접근방식이다.

## `git clone` 시 설정

```bash
$ git clone https://fransoaardi:<PAT>@github.com/fransoaardi/fransoaardi.github.io.git

# remote url 이 이미 위와 같이 설정되어있다.
$ git remote -v
origin  https://fransoaardi:<PAT>@github.com/fransoaardi/fransoaardi.github.io.git (fetch)
origin  https://fransoaardi:<PAT>@github.com/fransoaardi/fransoaardi.github.io.git (push)
```

## `git remote` 대상 변경

처음에 PAT 설정 없이 `$ git clone` 만 하였고, 아래와 같이 PAT 가 설정되지 않은것이 보인다. 그러나 `git remote set-url` 을 이용해서 origin 을 바꾼 모습이다.

```bash
$ git remote -v
origin  https://github.com/fransoaardi/fransoaardi.github.io.git (fetch)
origin  https://github.com/fransoaardi/fransoaardi.github.io.git (push)

# remote url 변경
$ git remote set-url origin https://fransoaardi:<PAT>@github.com/fransoaardi/fransoaardi.github.io.git

# 변경된 것 확인
$ git remote -v
origin  https://fransoaardi:<PAT>@github.com/fransoaardi/fransoaardi.github.io.git (fetch)
origin  https://fransoaardi:<PAT>@github.com/fransoaardi/fransoaardi.github.io.git (push)
```

# Caveat(s)

두 가지 방법 모두 위에서 말했듯 keychain 과 같이 다른곳에 저장되는 내용은 아니지만 `git remote -v` 를 해보건 `.git/config` 의 내용을 읽어보건 PAT 를 적나라하게 확인 가능하기 때문에 보안상 좋은 방법이라고 생각되진 않는다.

얼핏 보니 `Git Credential Manager` 을 사용하거나 `GitHub CLI` 를 이용하는 방법이 있다고 한다. 물론, 개발환경에서 다양한 github 관련 credential 이 요구될것으로 생각이 되어 굳이 keychain 과 같이 저장하는 방식은 사용하지 않을것이라서 이 방법들은 검토하지 않았다. 

# Reference(s)

- https://stackoverflow.com/a/18936804
- https://docs.github.com/en/get-started/getting-started-with-git/caching-your-github-credentials-in-git