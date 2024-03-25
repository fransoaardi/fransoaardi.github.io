---
layout: post
title: OAuth2.0 RFC6749 읽기
date: 2020-12-30 16:22:00 +0900
categories: [Wiki]
tags: [oauth]
toc: true
---

# reference
- https://tools.ietf.org/html/rfc6749

# introduction
## Roles 1.1
- 4 가지 roles 를 정의함 
- resource owner: 보호된 리소스(protected resource) 에 접근을 부여할 수 있는 주체. 사람인 경우는 `end-user` 라고 함
- resource server: protected resource 호스팅하고 access token 을 이용해 요청받고 응답할 수 있는 서버
- client: 리소스 소유자를 대신하여 권한을 부여하여 보호 된 리소스를 요청하는 애플리케이션.
- authorization server: 
리소스 소유자를 성공적으로 인증하고 권한을 얻은 후 서버가 클라이언트에 액세스 토큰을 발급.

- RFC6749 에서는 auth-server 와 resource-server 사이의 상호작용은 다루지 않는다. auth-server 와 resource-server 는 같은 서버일수도, 1대의 auth-server 가 여러 resource-server 를 위한 access-token 을 발급할 수도 있다.

## Protocol Flow 1.2

> 유명한 그림
```
     +--------+                               +---------------+
     |        |--(A)- Authorization Request ->|   Resource    |
     |        |                               |     Owner     |
     |        |<-(B)-- Authorization Grant ---|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(C)-- Authorization Grant -->| Authorization |
     | Client |                               |     Server    |
     |        |<-(D)----- Access Token -------|               |
     |        |                               +---------------+
     |        |
     |        |                               +---------------+
     |        |--(E)----- Access Token ------>|    Resource   |
     |        |                               |     Server    |
     |        |<-(F)--- Protected Resource ---|               |
     +--------+                               +---------------+
```

> 리얼월드 HTTP by 시부카와 요시키 (O'REILLY) p205 발췌
- Authorization vs Authentication
    - Authorization(권한부여): 인증된 사용자가 누구인지 파악한 후, 그 사용자에게 어디까지 권한을 부여할지 결정한다.
    - Authentication(인증): 로그인하려는 사용자가 누구인지 확인한다. 브라우저를 조작하는 사람이 서비스에 등록된 어느 사용자ID의 소유자인지 확인한다. 

## Authorization Grant 1.3
- resource owner 가 authorize 했음을 표현하는, client 가 access_token 을 얻어내기 위해 사용하는 credential. 
- `authorization code`, `implicit`, `resource owner password credentials`, `client credentials` 4가지 grant types 가 있다. 

- Authorization code:
- Implicit:
    - 단순화된 authorization code flow, Javascript 와 같은 script language 를 사용하는 browser 에 구현된 client 에 최적화됨.
    - authorization code 발급 없이, access token 을 직접 발급받음
    -  round trip 이 적으니, 효율적이지만, 보안적으로 따져봐야되고, `authorization code` 방식이 가능하면, 특히 고민해봐야된다.

- Resource owner password credentials:
    - client 와 resource owner 간에 높은 신뢰(client 가 device 의 운영체제의 일부이거나, 높은 권한을 갖고있는 app 인 경우) 가 있는 경우이거나, 다른 방식이 불가능할때 사용한다.

    - credential 을 client 에 저장할 필요 없이, 긴 수명을 갖는 access token 이나 refresh token 을 교환하는 식으로 대체할수있다.

- Client credentials:
    - client 가 resource owner 인 경우
    - requesting access to protected resources based on an authorization previously arranged with the authorization server.

- Refreshing an Expired Access Token
```
+--------+                                           +---------------+
  |        |--(A)------- Authorization Grant --------->|               |
  |        |                                           |               |
  |        |<-(B)----------- Access Token -------------|               |
  |        |               & Refresh Token             |               |
  |        |                                           |               |
  |        |                            +----------+   |               |
  |        |--(C)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(D)- Protected Resource --| Resource |   | Authorization |
  | Client |                            |  Server  |   |     Server    |
  |        |--(E)---- Access Token ---->|          |   |               |
  |        |                            |          |   |               |
  |        |<-(F)- Invalid Token Error -|          |   |               |
  |        |                            +----------+   |               |
  |        |                                           |               |
  |        |--(G)----------- Refresh Token ----------->|               |
  |        |                                           |               |
  |        |<-(H)----------- Access Token -------------|               |
  +--------+           & Optional Refresh Token        +---------------+

               Figure 2: Refreshing an Expired Access Token

```

2.  Client Registration

   Before initiating the protocol, the client registers with the
   authorization server.  The means through which the client registers
   with the authorization server are beyond the scope of this
   specification but typically involve end-user interaction with an HTML
   registration form.

   Client registration does not require a direct interaction between the
   client and the authorization server.  When supported by the
   authorization server, registration can rely on other means for
   establishing trust and obtaining the required client properties
   (e.g., redirection URI, client type).  For example, registration can
   be accomplished using a self-issued or third-party-issued assertion,
   or by the authorization server performing client discovery using a
   trusted channel.


   When registering a client, the client developer SHALL:

   o  specify the client type as described in Section 2.1,

   o  provide its client redirection URIs as described in Section 3.1.2,
      and

   o  include any other information required by the authorization server
      (e.g., application name, website, description, logo image, the
      acceptance of legal terms).
