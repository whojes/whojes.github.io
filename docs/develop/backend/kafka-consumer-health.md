---
layout: default
title: kafka consumer health
nav_order: 2
parent: springboot
grand_parent: 개발 
created_at: 2021.11.14
print_title: true
share_enable: true
permalink: develop/springboot/kafka-consumer-health
---

k8s 에서 서빙중인 pod 에 각각 사이드카인 envoy 가 붙어있다. L7 filter 용도로 쓰고있어서, 서비스 컨테이너로 패킷이 전달되기 전에 사이드카에 먼저 들렀다가 나가는데...

---

kublet 이 pod 에 shutdown hook 을 날리게 되면, envoy 컨테이너와 서비스 컨테이너가 같이 셧다운 훅을 받는다. 서비스 컨테이너(여기서는 톰캣)는 셧다운 훅을 받고 하던 일을 차례로 정리하기 시작한다. envoy 컨테이너는 자신이 서비스 컨테이너가 내려가기 전에 먼저 내려가게 되면 서비스 컨테이너로 패킷이 가는 길이 막히기 때문에 서비스가 의도하지 않은 상태에서 `500 service unavailable` 상태가 될 수가 있기 때문에, 서비스 컨테이너의 헬스를 체크해서 unhealthy 상태가 되면 서비스를 내린다. [참조](https://istio.io/latest/docs/ops/configuration/mesh/app-health-check/){:target="_blank"}

api 서빙과 kafka consuming 을 동시에 하는 서버가 있다. 정상적으로 동작하고 있던 서버를 내린다고 했을 때 
> 1. api 트래픽을 뺀다. (ingress야 힘을 내)
  2. 어플리케이션에서 graceful shutdown 을 한다. (shutdown hook)
    - api 트래픽은 앞에서 빼줬으니, 컨슈머 빈과 디펜던시 걸려있는 애들을 정리한다.
    - 컨슈머 빈을 포함한 bean 들이 destroy 가 끝나면 health check path 의 status 를 unhealthy 로 변경한다.
  3. 끝! 별 문제 없이 깔끔하다.

그러나 아직 부팅이 끝나지 않은 상태의 pod 에 셧다운 훅이 들어오게 되면 다음과 같은 이상동작이 발생할 수 있다.
> 1. bean initialize - kafka consumer bean initialized, auto startup (`org.springframework.kafka.listenerAbstractMessageListenerContainer` 의 default 동작) 되어서 일하기 시작함
  2. shutdown hook 들어옴
  3. 전체 bean initialize 가 아직 완료는 되지 않아서 (혹은 완료 되어도 health on 하기 전이라서) health 는 들어오지 않음
  4. envoy 가 service unavailable 으로 판단, outbound traffic 을 내림
  5. 카프카 컨슈머에서 외부로 api 호출을 하는 부분이 있다면, 바로 막혀버림

그래서, 결국 kafka consumer bean 의 라이프사이클을 container 의 health 라이프사이클과 맞춰줄 필요가 있다. 

auto-startup 옵션을 꺼주고, `org.springframework.boot.context.event.ApplicationReadyEvent` 를 받아서 그때 startup 을 해주면 문제 끝~

```kotlin 
@Bean
fun kafkaListener(): KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> {
    return ConcurrentKafkaListenerContainerFactory<String, String>().apply {
        setAutoStartup(false)
        ...
    }
}

@Bean
fun startupHandler(listeners: List<MessageListenerContainer>): ApplicationListener<ApplicationReadyEvent> {
  return ApplicationListener { listeners.forEach { it.start() } }
}

```

spring boot 버전에 따라 `KafkaListenerEndpointRegistry` 빈으로부터 해당 작업 (`messageListenerContainer.start()`)
을 진행해 줘야 할 수도 있다.