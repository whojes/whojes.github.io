---
layout: default
title: 공용 라이브러리
parent: thoughts
grand_parent: 개발
created_at: 2023.04.23
print_title: true
share_enable: true
permalink: develop/thoughts/common-libs
---

MSA 가 유행이라 그런가, 회사에 common library 가 참으로 많다. 홈팀에서 쓰는거, 메세지팀에서 쓰는거, apigw 팀에서 쓰는 거.. 
내가 짠 코드를 남이 잘 써주는건 참 기쁜일이지만, "버전 올렸는데 안돼요" 같은 소리를 듣기는 참 싫을테다. 
결국은 테스트코드를 잘 짜는 수 밖에 없는걸까?

--- 

#### 커먼 버전을 올렸는데 왜 되던게 안될까? 

1. 버전이 안맞아
- 나는 리액터 3.3.x 를 쓰고, 커먼 라이브러리는 3.5.x 를 기반으로 구현했다. 
- 커먼 라이브러리에서 `Mono.deferContextual` 를 호출하는 부분에서 에러가 난다. 
- 해당 메서드는 3.3.x 에는 없고 3.5.x 는 있어서, 내가 쓰는 버전이 오버라이드 되어 커먼코드에서 에러가 난다.
- 내가 리액터 버전을 올려야지 뭐..

2. 버전을 못올려
- 나와 커먼 라이브러리 모두 리액터 3.3.x 를 쓰고 있는 중에, 업데이트를 하기 위해 내 프로젝트의 리액터를 3.5.x 로 올렸다.
- 커먼 라이브러리에서 `Mono.subscriberContext` 를 호출하는 부분에서 에러가 난다.
- 해당 메서드는 3.3.x 에는 있고 3.5.x 에는 없어서, 커먼에서 쓰고 있었는데 내가 올리는 버전에는 없어서 에러가 난다.
- 커먼에 리액터 버전 올려주세요~ 요청하고, 올려줄때까지 기다린다.. 

3. 여러 버전을 지원해야해
- 나는 리액터를 3.5.x 로 올렸는데, 철수는 아직 3.3.x 를 쓰고 있고, 올릴 여력이 없다고 한다.
- 커먼에 리액터 버전 올려주세요- 요청했더니, "철수가 쓰고있어서 올리면 철수가 안된대"
- 에잉 젠장.. 그럼 커먼입장에서는 어떤 버전으로 구현해야 하는가??

4. 

--- 

#### 결국 커먼에서는 다양한 버전을 잘 지원을 해주어야한다.

`Mono.class.getPackage().getImplementationVersion()` 은 현재 런타임에 로딩된 리액터의 버전을 반환한다. 이를 활용하여 `ConditionalOnReactorVersion`등의 코드를 짜서 해결해볼 수 있겠다.

```kotlin

/**
  * whojes-common-reactor
  * dependency {
  *   api("whojes-common-reactor-3.4")
  *   api("whojes-common-reactor-3.5")
  * }
  */ 
interface MonoAdapter {
    fun subscriberContext(): Mono<Context>
}

/**
  * whojes-common-reactor-3.4
  * dependency {
  *   compileOnly("reactor:3.4") 
  * }
  */
@Configuration
@ConditionalOnReactorVersion(max = "3.4.+")
class MonoAdapterImpl: MonoAdapter {
    override fun subscriberContext(): Mono<Context> {
        return Mono.subscriberContext()
    }
}

/**
  * whojes-common-reactor-3.5
  * dependency {
  *   compileOnly("reactor:3.5") 
  * }
  */
@Configuration
@ConditionalOnReactorVersion(min = "3.5.0")
class MonoAdapterImpl: MonoAdapter {
    override fun subscriberContext(): Mono<Context> {
        return Mono.deferContextual(contextView -> Mono.just(Context.of(contextView)))
    }
}

```

이렇게 짜두고, 커먼 라이브러리 코드 (whojes-common-reactor) 에서는 MonoAdapter 빈을 주입받거나, 빈이 아니더라도 싱글턴으로 만들던지 해서 사용하면 되겠다.

--- 

#### 테스트코드를 작성을 참 잘해야한다.

해당 코드는 naver pinpoint 에서 jackson 을 지원하는 플러그인의 [테스트코드](https://github.com/pinpoint-apm/pinpoint/blob/master/plugins-it/jackson-it/src/test/java/com/navercorp/pinpoint/plugin/jackson/ObjectMapperIT.java#L46-L51)인데, 

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

`@RunWith` 에 지정된 `PinpointPluginTestSuite` 를 따라가다 보면, `@Dependency` 에 지정되어 있는 라이브러리의 버전을 [리졸브](https://github.com/pinpoint-apm/pinpoint/blob/master/test/src/main/java/com/navercorp/pinpoint/test/plugin/DependencyResolver.java#L285-L324) 해서 각 버전의 라이브러리를 가져와서 여러번의 테스트를 돌린다.
가령, 위에서 `[2.8.0,]` 노테이션은 `2.8.0 이상의 버전` 이라는 뜻인데, 저 테스트슈트에서는 지정된 메이븐 레포로 가서 저 노테이션에 맞는 모든 라이브러리 버전을 가져온 후, 모든 버전에 대해 테스트코드를 돌린다.

이건 테스트를 돌리는데 참 오래걸리고, 참조하는 라이브러리가 두개 이상이 되면 테스트 수가 아주 빠르게 늘어나고, 처음 짜기도 힘들지만 (혹시 범용되는 라이브러리가 있다면 알려주세요) 모든 라이브러리 버전에 대한 테스트가 돌기 때문에 마음이 편안해질 수 있겠다.

