---
layout: default
title: OpenID Connect
parent: study
grand_parent: 개발
created_at: 2020.01.10
print_title: true
share_enable: true
permalink: develop/study/oidc
---

개념을 이해하기전엔 혼란스러웠는데, 

| *원래의 OAuth(RFC 6749, 2.0) 는 네아로, 네이버 아이디로 로그인 기능을 위한 프로토콜이 아니다.*

네아로 개발자 api 페이지를 가보면 OAuth2.0을 구현했다고 해서 그런가보다 하고 쓰고 있는데, 뭔가 아무리 생각해도 어딘가 조금 납득이 안가는 부분이 있었는데, 이 깨달음을 얻은 후에야 비로소 OAuth로 구현된 네아로 기능의 면면을 확실하게 살펴볼 수 있었으니, 진즉에 공부했어야 마땅했다.

--- 
### OAuth  
  <br>
  그러니까, OAuth가 나오기 전에는, 이런 일이 있었단다. 그.\.\. LinkedIn 사이트에서 이제 이메일을 통해 친구추가를 할 수가 있었는데, 그 이메일을 하나 하나 입력해주기 어려우니까 google의 계정에 등록되어있는 이메일을 끌어다가 가져와서 등록해주겠다, 하는 기능이 있었다. 어떻게? ㅋㅋ google의 id/pwd를 나한테 줘! 그럼 내가 그 아이디로 로그인해서 주소록만 보고 다시 로그아웃할게! 하고 링크드인이 제공한거지. 그리고 실제로 그렇게 해서 적용을 했단다. 이 방법은 딱 보기에도 링크드인이 내 계정을 가지고 진짜로 주소록만 보고 닫을거라는 기이한 확신이 있어야만 가능한 짓인거지.

  그래서, OAuth가 등장했다. 링크드인에게 구글 계정 접속정보를 넘겨주는 것이 아니라, google에서 로그인을 한 후에 google이 가지고 있는 resource에 접근할 수 있는 특정 권한, 진짜로 딱 필요한 권한만을 가지고 있는 access token(여기서는 내 주소록에 접근할 수 있는 권한)을 링크드인에게 전달하면 링크드인은 그 토큰을 가지고 google에서 허가된 정보만 받아올 수 있도록 합리적이고 안전하게 구성해 놓은 프로토콜이 바로 OAuth다. ​

  링크드인은 구글 계정정보를 받는게 아니라 작은 창으로 구글 로그인 창(구글 서버에서 제공)을 띄워주고, 유저는 구글에 로그인을 하고, 구글은 유저에게 '링크드인이 주소록 정보를 요청하는데 줄거야?' 하고 물어보고, 주자! 하면 링크드인은 그 주소록에만 접근할 수 있는 access token을 얻을 수 있게 되어서 다른 계정관련 정보랑 무관하게 딱 주소록만 받아올수가 있게 되는 거시다. 

  Google 계정으로 링크드인에 로그인하기? 그것과는 아예 다른 이야기다.


---

### OpenID
  <br>
  whojes.com 이라는 멋진 사이트가 있다. 

  여기 사이트의 내 계정에 접근하는 방법, 즉 인증(Authentication)을 받는 방법은 전통적으로 whojes.com에 등록한 내 아이디와 패스워드를 입력하는 방법이다. 근데 모든 사이트마다 다 아이디/비밀번호를 제출하고 관리하는게 굉장히 귀찮단 말이야. 어디는 패스워드 규칙도 다르고 어디는 아이디 규칙도 다르고... 이를 해결하기 위해 OpenID Foundation 라는 비영리 단체가 나섰었다. 

  OpenID는 사이트마다 아이디/비밀번호를 입력하고 계정에 접근하는 것보다, 공인된 IdP(Identity Provider, ex. google)에게 물어보는 것으로 Sign in을 대체하려고 했다. whojes.com에 아이디/비밀번호를 입력하기보다 IdP에게 물어보도록 하여 IdP가 나임을 인증해주면 whojes.com은 나의 계정에 접근할 수 있는 권한을 주는 것이 더 깔끔하고 중앙관리가 잘 될거같다 이거지. 그래서 OpenID Foundation은 이 인증 프로토콜을 전파하고 Identity Provider를 해 줄 놈들을 모집하고 OpenID를 많이 지원해달라 하고 실제로 많이 성장하는 듯 보였으나.... 

  딱 봐도, 네이버 아이디로 로그인 기능은 OAuth가 아니라 OpenID가 맞는 기능인거지. 인가의 문제가 아니라 인증의 문제인 것이다. 아니 그럼 네아로에서 구현한 OAuth는 뭘까?

  <p align="center">
    <br><img alt="img-name" src="/assets/images/study/oidc_1.png" class="content-image-1" style="border: 1px solid black"><br>
    <em>출처 네아로</em><br>
  </p>

  물론 OAuth로도 authentication을 구현할 수 있다. 

  ​access token을 받는데, 어떤 권한에 대해서 받냐면 내 개인정보를 볼 수 있는 권한에 대해서 토큰을 받아버리면 된다. whojes.com이 네이버 아이디로 로그인을 하려고 한다 치자. whojes.com은 oauth 때처럼 네이버 로그인창을 살짝 띄우고, 유저는 네이버로 로그인을 하고, 네이버는 유저에게 'whojes.com이 너의 이메일, 성별, 이름을 가져가려고 하는데 줄까?' 하고 물어보고, 그래! 하면 네이버는 그 정보를 주는게 아니라 그 정보를 볼 수 있는 access token을 주는 것이다. whojes.com은 네이버로부터 나라는 인증을 받은게 아니라, 나의 정보를 볼 수 있는 토큰을 받은 것이다.

  ​whojes.com은 충분히 추론할 수 있다. 네이버로 인증하라고 창을 띄워 줬더니 훌륭하게 토큰을 받아왔고, 그 토큰을 가지고 네이버에 물어보니 개인정보를 알 수 있었으니, 이거면 인증하기에 충분하다! 라고 추론을 하는 것이다. 이것은 정확한 의미의 Authentication은 아니다.

  <p align="center">
    <br><img alt="img-name" src="/assets/images/study/oidc_2.png" class="content-image-1"><br>
    <em>나임을 인증해주는것이 아니라, 유저 정보를 조회할 수 있는 토큰을 발급해줌</em><br>
  </p>

---

### OpenID Connect (OIDC)
  <br>
  인가 프로토콜인 OAuth에 인증까지 얹어 제공하는 것이 바로 OIDC이다. OpenID 는 비영리단체에서 운영하는 "내가 아이디를 하나로 통합하겠다!" 하는 서비스이고, OpenID Connect 는 OAuth 프로토콜에 OpenID가 사용하는 인증 프로토콜을 얹은, OAuth의 강화버전이라고 할 수 있다. 즉, OpenID Connect 는 프로토콜이고, OpenID는 프로토콜이면서 동시에 이제는 잘 사용하지 않는 서비스다. 오해금지 (위키에도 가장 먼저 나오는 설명이 서로 혼동하지 말라는 문장)

  OIDC 는 OAuth가 주는 access_token에 더불어 issuer의 claims가 포함된 id_token 도 같이 건네준다. access token은 내가 누군지, 어디서 발급받았는지에 대한 정보는 하나도 들어가 있지 않고 특정 정보에 접근할 권한이 있는지만 체크를 하기 때문에, OIDC 없이 인증을 하려면 위의 네아로처럼 해야한다. (이를 pseudo-authentiation using OAuth 라고 한단다. https://en.wikipedia.org/wiki/OAuth#OpenID_vs._pseudo-authentication_using_OAuth )

​  구글로 로그인하기 기능이 이를 충실히 설명해 놓았다. 
  <p align="center">
    <br><img alt="img-name" src="/assets/images/study/oidc_3.png" class="content-image-1"><br>
  </p>
  <p align="center">
    <br><img alt="img-name" src="/assets/images/study/oidc_4.png" class="content-image-1"><br>
    <em>OAuth 로 구현시 ID_TOKEN 도 발급이 되고, 이것이 인증정보로 바로 사용이 된다.</em><br>
  </p>

  <p align="center">
    <br><img alt="img-name" src="/assets/images/study/oidc_5.png" class="content-image-1"><br>
    <em>검색량 추이</em><br>
  </p>
