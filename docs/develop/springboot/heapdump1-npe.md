---
layout: default
title: JVM Heapdump 1
title_sub: 힙덤프로 NPE 분석하기
nav_order: 7
parent: springboot
grand_parent: 개발 
created_at: 2025.05.04
print_title: true
share_enable: true
permalink: develop/springboot/heapdump1-npe
---

코드만 보면 NPE 가 날 수가 없는데 NPE 가 났다.  
  
```kotlin
abstract class Target {
    private val str: String = ""
    fun getStringLength(): Int = str.length
}

@Component
class TargetImpl : Target() {
	fun foo(): String = "foo"
	fun bar(): String = "bar"
}

@RestController
@RequestMapping("/test")
class TestController(
    private val target: Target,
) {
    @GetMapping
    fun get(): Int {
        return target.getStringLength()
    }
}
``` 

`curl localhost:8080/test` 를 호출하니 NPE가 발생. 
그런데 또 product 환경에서만 발생하고, alpha 환경에서는 발생하지 않았다. 엥.. 

```
java.lang.NullPointerException: Cannot invoke "String.length()" because "this.str" is null
	at io.github.whojes.heapdumptest.npe.Target.getStringLength(Target.kt:10) ~[main/:na]
	at io.github.whojes.heapdumptest.npe.TestController.get(Target.kt:23) ~[main/:na]
	at java.base/jdk.internal.reflect.DirectMethodHandleAccessor.invoke(DirectMethodHandleAccessor.java:103) ~[na:na]
	at java.base/java.lang.reflect.Method.invoke(Method.java:580) ~[na:na]
	at kotlin.reflect.jvm.internal.calls.CallerImpl$Method.callMethod(CallerImpl.kt:97) ~[kotlin-reflect-2.1.20.jar:2.1.20-release-217]
	at kotlin.reflect.jvm.internal.calls.CallerImpl$Method$Instance.call(CallerImpl.kt:113) ~[kotlin-reflect-2.1.20.jar:2.1.20-release-217]
	at kotlin.reflect.jvm.internal.KCallableImpl.callDefaultMethod$kotlin_reflection(KCallableImpl.kt:207) ~[kotlin-reflect-2.1.20.jar:2.1.20-release-217]
	at kotlin.reflect.jvm.internal.KCallableImpl.callBy(KCallableImpl.kt:112) ~[kotlin-reflect-2.1.20.jar:2.1.20-release-217]
```

아니 어떻게 this.str 이 null 이 될 수 있을까? 

---

이럴 때는 바로 힙덤프를 떠본다. arthas 도 좋은데, 어쨌든 런타임에서 살아있는 pod보다 죽어있는 pod가 더 많기에 힙덤프 분석이 범용적으로 유용하다.  
OQL 에서 `SELECT * FROM io.github.whojes.heapdumptest.npe.TestController` 로 내 컨트롤러가 어떤 객체를 보고 있는지 먼저 확인할 수 있다.  

<p align="center">
  <br><img alt="img-name" src="/assets/images/heapdump/1_npe_1.png" class="content-image-1"><br>
</p>
알파 환경에서는 `TargetImpl` 객체를 참조하지만, 라이브 환경에서는 `TargetImpl$$SpringCGLIB$$0` 객체를 참조하고 있다. CGLIB 에 관련해서는 여러 블로그가 있으니, 구글링해보면 알 수 있겠고... 그럼 문제를 정리해보자면,

1. 왜 라이브에서만 CGLIB 프록시 객체가 생겼는가??  
2. 내 코드 상에서 null 일 수 없는 `this.str` 가 왜 null 일까??  
  
1번이 사실상 먼저 궁금하지만, 우선순위는 아니다. cglib 프록시 객체가 왜 생겼는지는 다양한 설정에 따라 다르므로 우선 넘어가고, cglib 프록시 객체 생긴 꼬라지를 살짝 보자면,
<p align="center">
  <br><img alt="img-name" src="/assets/images/heapdump/1_npe_2.png" class="content-image-1"><br>
</p>

내가 실제로 호출한 `getStringLength` 메서드의 프록시 메서드 객체가 보이질 않는다. 임의로 만들어 둔 `foo` 메서드와 `bar` 메서드의 프록시 객체는 잘 보인다. 
  
---

프록시 메서드 객체가 없으면 어떻게 될까? 답은 간단하다. `TargetImpl$$SpringCGLIB$$0` 프록시 객체 역시 `TargetImpl` 의 하위 클래스로, 프록시 메서드 객체 없으면 그냥 `getStringLength` 메서드가 호출된다.