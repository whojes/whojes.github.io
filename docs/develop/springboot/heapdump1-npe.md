---
layout: default
title: JVM Heapdump 1
title_sub: 힙덤프로 NPE 분석하기
nav_order: 8
parent: springboot
grand_parent: 개발 
created_at: 2025.05.04
print_title: true
share_enable: true
permalink: develop/springboot/heapdump1-npe
---

코드만 보면 NPE 가 날 수가 없는데 NPE 가 났다. 알고 보면 다 보이겠지만, 알고 보는 문제는 몇 개 없다. 분석할 줄 알아야 한다.  
  
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
이럴 때는 바로 힙덤프를 떠본다.  
  
OQL 에서 `SELECT * FROM io.github.whojes.heapdumptest.npe.TestController` 로 내 컨트롤러가 어떤 객체를 보고 있는지 먼저 확인할 수 있다.  

<p align="center">
  <img alt="img-name" src="/assets/images/heapdump/1_npe_1.png" class="content-image-1"><br>
</p>
알파 환경에서는 `TargetImpl` 객체를 참조하지만, 라이브 환경에서는 `TargetImpl$$SpringCGLIB$$0` 객체를 참조하고 있다. CGLIB 에 관련해서는 여러 블로그가 있으니, 구글링해보면 알 수 있겠고... 그럼 문제를 정리해보자면,

1. 왜 라이브에서만 CGLIB 프록시 객체가 생겼는가??  
2. 내 코드 상에서 null 일 수 없는 `this.str` 가 왜 null 일까??  
  
1번이 사실상 먼저 궁금하지만, 우선순위는 아니다. cglib 프록시 객체가 왜 생겼는지는 다양한 설정에 따라 다르므로 우선 넘어가고, cglib 프록시 객체 생긴 꼬라지를 살짝 보자면,
<p align="center">
  <img alt="img-name" src="/assets/images/heapdump/1_npe_2.png" class="content-image-1"><br>
</p>

내가 실제로 호출한 `getStringLength` 메서드의 프록시 메서드 객체가 보이질 않는다. 임의로 만들어 둔 `foo` 메서드와 `bar` 메서드의 프록시 객체는 잘 보인다.  
  
  
---  
프록시 메서드 객체가 없으면 어떻게 될까? 답은 간단하다. `TargetImpl$$SpringCGLIB$$0` 프록시 객체 역시 `TargetImpl` 의 하위 클래스로, 프록시 메서드 객체 없으면 그냥 `getStringLength` 메서드가 호출된다. 여기서 다시 문제를 정리해 보자면, 
  
1. 왜 `getStringLength` 는 프록시 메서드 객체가 없을까?  
2. 내 코드 상에서 null 일 수 없는 `this.str` 가 왜 null 일까??  
  
1번이 사실상 먼저 궁금하지만, 우선순위는 아니다. `this.str` 는 언제 empty string 으로 초기화가 될까? 

코틀린은 잘 모르겠으니, 코틀린 바이트코드로 전환 후 자바 코드로 디컴파일 해보면 다음과 같다.  

```java 
public abstract class Target {
   @NotNull
   private final String str = "";

   public final int getStringLength() {
      return this.str.length();
   }
}

public class TargetImpl extends Target {
   @NotNull
   public String foo() {
      return "foo";
   }

   @NotNull
   public String bar() {
      return "bar";
   }
}
```
  
`str` 필드는 인스턴스 변수로써, 생성자 호출 전에 호출이 된다. `TargetImpl$$SpringCGLIB$$0` 이 객체는 근데 생성자가 뭐가 호출이 될까? 생성자가 여러개 일 수도 있잖아. 기본 생성자를 그냥 호출할까?  
  
--- 
  
[Objenesis](https://github.com/spring-projects/spring-framework/blob/6.1.x/spring-core/src/main/java/org/springframework/objenesis/SpringObjenesis.java) 라는 라이브러리가 있다. CGLIB 은 프록시 객체 생성 시에 타겟 객체의 생성자를 호출하지 않고, 다음과 같이 객체를 생성한다. 
```java
public <T> T newInstance(Class<T> clazz, boolean useCache) {
	if (!useCache) {
		return newInstantiatorOf(clazz).newInstance();
	}
	return getInstantiatorOf(clazz).newInstance();
}
```

`new` 키워드를 사용하지 않고 `instantiator` 를 가져와서 `newInstance` 를 호출하는 방식으로 호출한다. 디버거를 찍어보면 instantiator 로 `SunReflectionFactoryInitiator` 가  구현체이고, 코드는 다음과 같다.  

```java
@Instantiator(Typology.STANDARD)
public class SunReflectionFactoryInstantiator<T> implements ObjectInstantiator<T> {

   private final Constructor<T> mungedConstructor;

   public SunReflectionFactoryInstantiator(Class<T> type) {
      Constructor<Object> javaLangObjectConstructor = getJavaLangObjectConstructor();
      mungedConstructor = SunReflectionFactoryHelper.newConstructorForSerialization(
          type, javaLangObjectConstructor);
      mungedConstructor.setAccessible(true);
   }

   public T newInstance() {
      try {
         return mungedConstructor.newInstance((Object[]) null);
      }
      catch(Exception e) {
         throw new ObjenesisException(e);
      }
   }
   ...
```

일반적인 객체 생성이 아니라, `java.lang.Object` 의 생성자를 reflection 으로 받아와서 객체를 생성한다. 이렇게 객체를 만들게 되면 stack 상에는 `TargetImpl` 로 타입이 있지만 그 포인터가 가리키는 힙에는 그냥 공간만 할당되어있을 뿐 아무 정보(객체 생성시에 초기화되는 인스턴스 필드 포함)가 없이 생성이 되고, 원래 `TargetImpl` 클래스가 가진 생성자와 jvm이 수행하는 초기화 루틴 또한 호출이 되지 않아 정말 빈껍데기의 객체가 생성이 된다.  
  
즉, `getStringLength` 가 정상적으로 오버라이드가 되어 프록시 메서드가 생성이 됐다면, 프록시 객체의 초기화 되지 않은 `str` 에 접근할 필요가 없고, 프록시 객체 안에 품고 있는 실제 `TargetImpl` 객체의 `str` 에 접근을 해서 NPE 가 발생하지 않았을 것이다.
<p align="center">
	<img alt="img-name" src="/assets/images/heapdump/1_npe_4.png" class="content-image-1"><br>
</p>
  
---  
아직 덜 풀린 수수께끼. 
1. 왜 라이브에서만 CGLIB 프록시 객체가 생겼는가??  
2. 왜 `getStringLength` 는 프록시 메서드 객체가 없을까?  

1번은 여러 이유로 가능할 수 있다. 지금 케이스에서는 다음과 같은 설정이 있었다. 

```kotlin
@Configuration
@EnableAspectJAutoProxy(proxyTargetClass = true)
@Profile("live")
class Configuration {
    @Bean
    fun testAspect(): TestAspect = TestAspect()
}

@Aspect
class TestAspect {
    @Around("execution(* io.github.whojes.heapdumptest.npe.Target.*(..))")
    fun pointcut(joinPoint: ProceedingJoinPoint): Any {
        return joinPoint.proceed()
    }
}

```
  
지금은 이게 내 코드에 있었지만, 여러 라이브러리를 쓰는 이상 이정도 aspect 는 언제든 들어올 수 있다. 실제로 내가 겪은 환경에서는 `@Scheduled` 가 달려있는 메서드의 수행시간을 측정하기 위해 aspect 가 하나 있었고, 이게 하필 라이브에서만 수집하게 해놔서 라이브에서만 발생했었다. 사실 잘못된 것이 아니기 때문에 이걸 수정할 수는 없다.  
  
2번을 보자면, `arthas` 라는 런타임 분석을 통해 확인해볼 수 있다. 
<p align="center">
  <img alt="img-name" src="/assets/images/heapdump/1_npe_3.png" class="content-image-1"><br>
</p>
`getStringLength` 메서드는 final 으로 돌고있다. 왜? kotlin은 `open` 을 달지 않으면 기본적으로 final 이기 때문이다.  
  
난 open 다 안달고 쓰는데?? 그건 `kotlin-spring` 라이브러리가 스프링 빈일 경우 다 open 으로 (kotlin all-open 검색) 바꿔주기 때문이다. 하필 또 `abstract class` 여서 bean 으로 스프링에 등록되지 않는 클래스여서 저 메서드만 딱 `open` 이 안되었다.  
  
---

해결법은 결국 `getStringLength` 메서드에 `open` 을 붙여주면 된다.  
  
문제를 알고 보면 해답은 매우 간단했다.  
 