---
layout: default
title: shutdown hook
nav_order: 3
parent: springboot
grand_parent: 개발 
created_at: 2021.11.14
print_title: true
share_enable: true
permalink: develop/springboot/shutdown-hook
---

리눅스 환경에서 Jvm 은 `SIGTERM` 이 들어올 경우 graceful shutdown 을 진행한다. 셧다운이 되기 전에 실행할 코드를 넣을 수 있는데, 여기에 필요한 코드는 

```kotlin
Runtime.getRuntime().addShutdownHook(Thread { 
  logger.info("your code here") 
})
```

---

spring boot 앱에서 어떤 셧다운 훅이 등록되었는지를 확인해보자.
```kotlin
val clazz = Class.forName("java.lang.ApplicationShutdownHooks")
val field = clazz.getDeclaredField("hooks")
field.isAccessible = true
val threads = field.get(clazz) as IdentityHashMap<Thread, Thread>
```

여기서 `threads` 를 살펴보면
1. `{LogManager$Cleaner@~} "Thread[Thread-0,5,main]"`
2. `{AbstracApplicationContext$1@~} "Thread[SpringContextShutdownHook,5,main]"`  

이렇게 두 개의 shutdown 훅이 등록되어 있다. (여기에 어플리케이션을 통째로 래핑하고 있는 apm 등을 사용하면 추가될 수도 있다.)

첫번째 훅은 `java.util.logging.LogManager$Cleaner`쓰레드로, 자바 1.4에서 도입된.. 뭐.. 그런거고, 두번째 것이 spring boot 에서 등록한 훅이다. spring boot 프레임워크에서 개발을 할 때에 유저레벨에서 서버가 종료될 때 동작했으면 하는 코드가 있으면 `ContextClosedEvent` 이벤트를 리스닝해서 훅을 등록하는게 낫다. `Runtime.getRuntime()` 에 등록된 훅들이 실행될 때는 순서가 없기 때문에, spring boot context 에 등록하지 않고 따로 `Runtime` 에 등록을 해두게 되면 spring boot context 의 종료가 되기 전에 사용하는 모듈이 내려가버리는 불상사가 발생할 수도 있기 때문이다.

--- 

[apache shardingsphere](https://shardingsphere.apache.org/){:target="_blank"} 는 아파치에서 제공하는 샤딩 라이브러리인데, 여기서는 broadcasting table 에 대해 각각의 샤드로 쿼리를 날릴 때 자체적으로 쓰레드풀을 만들어서 쿼리를 날린다. 셧다운 훅이 왔을 때 그 스레드풀이 내려가는데, 요 동작이 `java.lang.Runtime` 에 직접 등록이 되어있다. [코드](https://github.com/apache/shardingsphere/blob/master/shardingsphere-infra/shardingsphere-infra-executor/src/main/java/org/apache/shardingsphere/infra/executor/kernel/thread/ExecutorServiceManager.java#L47){:target=":blank"}

그렇기 때문에, spring boot 를 사용하는 내 어플리케이션 코드에서는 디펜던시가 있는 `JpaRepository` 가 먼저 내려가기 전에 그 쓰레드풀이 먼저 내려가 버리는 상황이 종종 발생했고, 그로 인해 에러가 올라오고 있었다. 저 코드에서는 `google guava` 라이브러리의 [`MoreExecutors.addDelayedShutdownHook`](https://guava.dev/releases/22.0/api/docs/com/google/common/util/concurrent/MoreExecutors.html#addDelayedShutdownHook-java.util.concurrent.ExecutorService-long-java.util.concurrent.TimeUnit-){:target="_blank"} 를 사용하는데, 이름은 delayed shutdown 이지만 코드를 읽어보면 시간을 기다리고 shutdown 을 날리는게 아니라 우선 shutdown 을 하고 시간동안 기다리는 코드다. 

해당 라이브러리에서 `Runtime` 에 등록하는 셧다운 훅 쓰레드를 참조할 방법도 없고, 등록을 막는 방법도 없어서 스프링 부트 안에서는 따로 해결할 방법이 없었다. 그래서 요 문제를 해결하기에는 다음과 같은 방법이 있다. 
1. 오픈소스인 샤딩스피어 라이브러리를 포크해서 해당 부분만 고친 후에 빌드해서 사용  
  - 옵션으로 주입받을 수 있게 하거나, Runtime 에 등록하는게 아니라 `Closeable` 을 상속한 후에 외부에서 호출할 수 있게
  - 차후에 마이너 버그픽스도 귀찮아지기 때문에 포기
2. 샤딩스피어 쪽에 이슈제기를 해서 고쳐지기를 바란다.
  - 메인 컨트리뷰터들이 중국인이라서 그런가 [이슈 업](https://github.com/apache/shardingsphere/issues/10641){:target="_blank"}을 해놨는데도 댓글조차 안달림(마상) 
3. 리플렉션으로 셧다운 훅을 다 들고와서 저놈들만 빼서 제거
  - 당장은 제일 간단하지만 모두가 불만족하는 코드

---

3번에 대해 코드를 보면

```kotlin
val clazz = Class.forName("java.lang.ApplicationShutdownHooks")
val field = clazz.getDeclaredField("hooks")
field.isAccessible = true
val threads = field.get(clazz) as IdentityHashMap<Thread, Thread>

// 모두가 불만족하는 코드 
val shardingSphereShutdownHooks = threads.map { it.key }.filter { it.name.startsWith("DelayedShutdownHook-for-") }
shardingSphereShutdownHooks.forEach { 
  deferredHooks.add(it)
  Runtime.getRuntime().removeShutdownHook(it)
}

// shutdown handler 등록
@Bean
fun shutdownHandler(): ApplicationListener<ContextClosedEvent> {
  return ApplicationListener { deferredHooks.forEach { it.start() } } // join 도 해서 기다려줘야함. 간단히 할려고 이렇게 씀
}
```

