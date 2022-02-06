---
layout: default
title: conditional on bean
title_sub: '@ConditonalOnBean'
nav_order: 5
parent: springboot
grand_parent: 개발 
created_at: 2022.02.05
print_title: true
share_enable: true
permalink: develop/springboot/conditional-on-bean
---

`@ConditionalOnBean` 의 설명은 다음과 같다.

| that only matches when beans meeting all the specified requirements are already contained in the BeanFactory.

대충 인자로 들어있는 클래스의 빈이 빈팩토리에 존재한다면 컴포넌트를 생성한다는 말이다. `@ConditionalOnMissingBean` 은 반대로 빈팩토리에 없을 때 컴포넌트를 생성한다는 것이다. 즉,

```kotlin
@Configuration
class ApplicationConfiguration{
    @Bean
    @ConditionalOnBean(TestProperty::class)
    fun someConfig(testProperty: TestProperty): SomeConfig {
        return SomeConfig(testProperty)
    }
}
```

이 코드에서 `TestProperty` 타입의 빈이 있을 경우에 해당 빈을 주입받아 `SomeConfig` 빈을 만들고, `TestProperty` 빈이 없으면 `SomeConfig` 빈을 만들지 않도록 한다.

허나, 다음과 같은 경우일 지라도 `SomeConfig` 빈이 생성될 수가 있다.

```kotlin
@Configuration
class ApplicationConfiguration: ApplicationContextAware {
    private lateinit var applicationContext: ApplicationContext
    private val logger = Logger.getLogger(ApplicationConfiguration::class.simpleName)

    override fun setApplicationContext(applicationContext: ApplicationContext) {
        this.applicationContext = applicationContext
    }

    @Bean
    @ConditionalOnMissingBean(TestProperty::class)
    fun someConfig(testProperty: TestProperty): SomeConfig {
        logger.info("testProperty has been injected! $testProperty")
        logger.info("testProperty found in application context! ${applicationContext.getBean("testProperty")}")
        return SomeConfig(testProperty)
    }
}
// output
// testProperty has been injected! com.whojes.common.config.TestPropertyImpl@4aa3d36
// testProperty found in application context! com.whojes.common.config.TestPropertyImpl@4aa3d36
```

`@ConditionalOnMissingBean` 을 달아놨는데도 주입도 되고 applicationContext 에서 가져올 수도 있다. ***엥?***

<hr>

| already contained in the BeanFactory

이 문장의 의미는 `@ConditionalOn(Missing)Bean` 의 인자로 사용된 빈의 컴포넌트 스캔 순서가 해당 클래스의 컴포넌트 스캔 순서보다 빨라야만 한다는 것이다. 즉 Bean 이 최종적으로 create 되어있는 상태를 보는 게 아니라, 저 애너테이션이 달려있는 부분을 스캔할 때에 빈팩토리에 해당 bean definition 이 등록되어 있어야 한다. (Bean creation 이 아니므로 `@Order`, `@DependsOn` 류의 생성 순서보장은 먹히지 않는다.)

다음과 같은 상황을 생각해보자.

```kotlin
// com.whojes.demo.configuration.ApplicationConfiguration 
@Configuration
class ApplicationConfiguration {
    @Bean
    @ConditionalOnBean(TestProperty::class)
    fun someConfig(testProperty: TestProperty): SomeConfig {
        return SomeConfig(testProperty)
    }
}
```

이 클래스는 `SomeConfig` 빈을 생성하는데, `TestProperty` 빈이 있어야만 생성을 한다. 내가 `TestProperty` 를 생성을 해줘야 하는데, 
```kotlin
@Configuration
class Ao {
  @Bean
  fun testProperty(): TestProperty = TestProperty()
}
```

```kotlin
@Configuration
class Aq {
  @Bean
  fun testProperty(): TestProperty = TestProperty()
}
```
Ao 클래스로 생성을 하던 Aq 클래스로 생성을 하던 런타임에 `TestProperty` 빈을 인젝션받거나 applicationContext 에서 찾는건 문제없이 가능하지만, 컴포넌트 스캔 순서가 Ao -> ApplicationConfiguration -> Aq 이기 때문에 Aq 로 빈을 만들면 `ApplicationConfiguration` 에 있는 `@ConditionalOn(Missing)Bean`은 정상동작하지 않는다. Component Scan 은 alphabetically 이뤄지기 때문에 만약 클래스 이름 따위로 제어하기 싫다면 `@ComponentScan({})` 의 순서를 바꾸거나, Aq 클래스의 `@Configuration` 애너테이션을 떼버리고 `ApplicationConfiguration` 클래스에 `@Import(Aq::class)` 를 달아버리면 된다. 역시 이상한건 매한가지지만...
<hr>

일반적으로 한 프로젝트 안에서 `@ConditionalOn(Missing)Bean` 을 사용하지는 않고, (나는 원소스로 서로 다른 인프라에 제공하기 위해 작성한 적이 있다.) 다른 프로젝트를 위해 사용할 라이브러리를 짤 때나 특정 라이브러리를 사용할 때 해당 애너테이션을 쓰는 경우가 있다.

라이브러리를 제공하는 쪽에서는 자신의 config를 로드하게 하기 위해 다음과 같은 방법을 쓴다. 
1. `@Enable~` 류의 클래스를 작성해둔다.
```kotlin
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(TestPropertyConfiguration::class)
annotation class EnableTestProperty
```
이러면 DemoApplication 의 컴포넌트 스캔 경로 아래에 `@EnableTestProperty` 가 달린 configuration 을 하나 만들면 된다.

2. META-INF/spring.factories 파일을 추가하고, 다음 글귀를 작성한다.
```properties
org.springframework.boot.autoconfigure.EnableAutoConfiguration=com.whojes.common.config.TestPropertyConfiguration
``` 
이러면 의존성을 추가해주기만 하면 springboot application 이 로딩될 때 autoconfiguration 된다. 

<hr>

그래서, 다른 라이브러리를 쓰면서 그 라이브러리에서 생성하는 Bean 에 대해 Conditional 을 걸고 싶다면

1. `@Enable~` 로 제공되는 경우 
  - `@ConditionalOn...` 을 사용하는 config 에 직접 달거나, 이것보다 빠른 순서로 ComponentScan 이 되는 클래스에 달아야 한다.
2. auto-config 로 제공되는 경우
  - 나만의 auto-config 를 만든다. 
    - `ApplicationConfiguration` 클래스를 컴포넌트 스캔이 되지 않는 패키지로 옮긴다.
    - 그리고 `@AutoConfigureAfter (TestPropertyConfiguration::class)` 를 추가해준다.
    - 해당 파일을 `META-INF/spring.factories` 에 추가하여 auto-configure 되도록 한다.
  - 또는, `ApplicationConfiguration` 클래스에 `@Import(TestPropertyConfiguration::class)` 를 붙여준다.

그리고, 다른 프로젝트에 제공하기 위해 라이브러리를 짜는 경우에는 보통 "네가 만든거 있으면 쓰고 없으면 디폴트로 하나 만들어줄게" 하는 형식일 것인데, auto-config 의 경우에는 `META-INF/spring.factories` 에 의한 auto configuration 은 메인 스프링부트 앱의 component scan 이후에 진행이 되므로, 크게 고려할 사항은 없다. `@Enable~` 의 경우에는... 저렇게 쓰지 말아야한다. 사용하는 쪽에서 `@Enable~` 을 어느 클래스에 다느냐에 따라서 달라지기 때문이다.

참고로 `@SpringBootApplication` 이 달려있는 메인 클래스에 `@Enable~` 을 달면 컴포넌트 스캔이 다 이루어진 후에 `@Enable~` 관련 스캔이 이루어진다.
