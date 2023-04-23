---
layout: default
title: 공용 라이브러리
parent: etc
grand_parent: 개발
created_at: 2023.04.23
print_title: true
share_enable: true
permalink: develop/etc/common-libs
---

MSA 가 유행이라 그런가, 회사에 common library 가 참으로 많다. 홈팀에서 쓰는거, 메세지팀에서 쓰는거, apigw 팀에서 쓰는 거.. 
내가 짠 코드를 남이 잘 써주는건 참 기쁜일이지만, "버전 올렸는데 안돼요" 같은 소리를 듣기는 참 싫을테다. 

버전을 올렸는데 왜 되던게 안될까? 

- 내가 사용하는 라이브러리 버전과 커먼에서 사용하는 라이브러리 버전이 호환이 되지 않는다.
- 


### 커먼코드에서 다른 라이브러리를 사용할 때는 Deprecate 될 때마다 꼬박꼬박 잘 따라가 줘야한다. 



### 테스트코드를 작성을 참 잘해야한다.

해당 코드는 naver pinpoint 에서 jackson 을 지원하는 플러그인의 [테스트코드](https://github.com/pinpoint-apm/pinpoint/blob/master/plugins-it/jackson-it/src/test/java/com/navercorp/pinpoint/plugin/jackson/ObjectMapperIT.java#L46)인데, 

```java
@RunWith(PinpointPluginTestSuite.class)
@PinpointAgent(AgentPath.PATH)
@ImportPlugin("com.navercorp.pinpoint:pinpoint-jackson-plugin")
@Dependency({"com.fasterxml.jackson.core:jackson-databind:[2.8.0,]"})
public class ObjectMapperIT {

    private static final Charset UTF_8 = StandardCharsets.UTF_8;
    ...
}
```

`@RunWith` 에 지정된 PinpointPluginTestSuite 를 따라가다 보면, `@Dependency` 에 지정되어 있는 라이브러리의 버전을 [리졸브](https://github.com/pinpoint-apm/pinpoint/blob/master/test/src/main/java/com/navercorp/pinpoint/test/plugin/DependencyResolver.java#L285-L324) 해서 각 버전의 라이브러리를 가져와서 여러번의 테스트를 돌린다.
가령, 위에서 `[2.8.0,]` 노테이션은 `2.8.0 이상의 버전` 이라는 뜻인데, 저 테스트슈트에서는 지정된 메이븐 레포로 가서 저 노테이션에 맞는 모든 라이브러리 버전을 가져온 후, 모든 버전에 대해 테스트코드를 돌린다.

이건 테스트를 돌리는데 참 오래걸리고, 참조하는 라이브러리가 두개 이상이 되면 테스트 수가 아주 빠르게 늘어나고, 처음 짜기도 힘들지만 (혹시 범용되는 라이브러리가 있다면 알려주세요) 모든 케이스에 걸리게 된다.

