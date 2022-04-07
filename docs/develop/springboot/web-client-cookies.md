---
layout: default
title: webClient 적용기
title_sub: webClient 와 Set-Cookie 헤더
nav_order: 6
parent: springboot
grand_parent: 개발 
created_at: 2022.04.07
print_title: true
share_enable: true
permalink: develop/springboot/web-client-cookie
---

사건의 발단은 webflux 로의 전환이었다. A서버 -> B서버 의 api 흐름을 가지는 시스템에서 A서버를 웹플럭스로 전환하여 cpu usage 및 B서버 호출 병렬처리로 response time 을 절약하려는 취지였으나... 배포 후 카나리 상태에서 트래픽을 조금씩 넣기 시작해서 20퍼센트가 되면 이내 서킷이 열리고 장애가 났다. 왜지... 하고 분석하다가 알게된 사실에 대한 정리

### 메트릭/apm 샘플을 열심히 분석
1. 트래픽 없는 상태에서 기능은 정상수행 된다.
2. B서버의 리스폰스가 엄청 느려지는 샘플이 계속 찍혔다. requestMapping 부분을 aspect로 감싼 부분에서 찍는 로그에 나온 api 의 실행시간은 변화가 없었으니 톰캣 프레임워크쪽에서 펜딩이 걸렸으려나?
3. B서버의 CPU가 점점 튀더니 100을 찍었고, 그 때부터 A서버에서 타임아웃이 나면서 서킷이 열렸다.

### 가설과 기각
1. 병렬처리가 너무 빠른 속도로 되어서 B서버가 견디지 못했다.
  - 병렬처리 빼고 순차처리 해봐도 해소 안됨
2. 웹플럭스 전환 후 어딘가 블락되는 코드가 있어서, B서버와 무관하게 A서버의 잘못이다.
  - 레디스 방문(ReactiveRedisTemplate 적용)과 api 호출이 전부여서 블락될 구간이 없고, 우선 저 B서버의 cpu 튀는 현상이 설명이 안된다.
3. httpClient 커넥션 풀 설정이 잘못돼서 커넥션이 매번 새로 연결되느라 그랬다.
  - http/1.1 스펙은 디폴트가 keep-alive 고, 혹시 몰라 헤더 넣어도 해결 안됐다.
  - B서버의 open files 가 튀지 않아서 우선은 아닌걸로...

### 딥다이브 
하여, 이유를 도무지 알 수 없어 로컬에서 A서버를 구버전으로 왔다갔다 하며 tcp 덤프를 떠본 결과... 문제가 웹플럭스가 아니라  ***http 클라이언트를 restTemplate -> webClient 로 변환한 부분***이었음을 알아냈다. B서버 쪽으로 `Cookie: JSESSIONID=AE0B...` 하는 헤더가 들어오지 않은 것이다. 엥??

[RFC-6265](https://datatracker.ietf.org/doc/html/rfc6265) 에 따르면 마땅히 서버가 리스폰스에 `Set-Cookie` 헤더를 내려주면 클라이언트는 다음 요청부터 쿠키를 넣어서 보내야 하는데, webClient 는 이 동작이 없었던 것이다.

<br/>

```kotlin
@RestController
@RequestMapping("/")
class Controller(
    private val webClient: WebClient,
    private val restTemplate: RestTemplate,
) {
    enum class Type {
        REST_TEMPLATE,
        WEB_CLIENT
    }

    @GetMapping("test")
    fun test(
        @RequestParam("type") type: Type
    ): Map<String, String> {
        val uri = "http://localhost:8080/get-cookie"
        val (user, password) = Auth.userPassword

        return when (type) {
            Type.REST_TEMPLATE -> {
                restTemplate.exchange(
                    uri,
                    HttpMethod.GET,
                    HttpEntity(null, HttpHeaders().also { it.setBasicAuth(user, password) }),
                    object : ParameterizedTypeReference<Map<String, String>>() {}
                ).body!!
            }
            Type.WEB_CLIENT -> {
                webClient.get()
                    .uri(uri)
                    .headers { it.setBasicAuth(user, password) }
                    .retrieve()
                    .bodyToMono(object : ParameterizedTypeReference<Map<String, String>>() {})
                    .block()!!
            }
        }
    }

    @GetMapping("get-cookie")
    fun getHeader(
        httpServletRequest: HttpServletRequest,
    ): Map<String, String> {
        return mapOf("Cookie" to (httpServletRequest.cookies?.get(0)?.let { "${it.name}=${it.value}" } ?: "null"))
    }
}

@Configuration
class Security : WebSecurityConfigurerAdapter() {
    override fun configure(http: HttpSecurity) {
        http
            .httpBasic()
            .and()
                .authorizeRequests()
                .antMatchers("/get-cookie")
                .hasRole(Auth.role)
    }
    override fun configure(auth: AuthenticationManagerBuilder) {
        val encoder = PasswordEncoderFactories.createDelegatingPasswordEncoder()
        val (user, password) = Auth.userPassword
        auth.inMemoryAuthentication()
            .withUser(user).password(encoder.encode(password)).roles(Auth.role)
    }
}
````

이 코드는 `spring-boot-starter-security` 를 사용했다. `/get-cookie` api 는 basicAuth 가 적용되어 있고 리퀘스트의 `Cookie` 헤더를 리턴한다. `/test` api 는 호출시 파라미터에 따라 각각 restTemplate/webClient 를 이용해서 `/get-cookie` 를 호출하며, 그 호출의 응답을 그대로 리턴한다. 

basicAuth 가 적용되어있는 `/get-cookie` api 는 처음 호출시에 securityContext 유지를 위해 세션을 생성하고, 리스폰스에 해당 세션에 대한 정보 (엔진인 톰캣에서 사용하는 JSESSIONID)를 `Set-Cookie` 헤더에 담아서 내보낸다. 클라이언트는 요 `Set-Cookie` 헤더를 보고 다음 요청부터는 해당 정보를 리퀘스트에 담아서 보내고, 서버는 기존에 인증을 했던 세션임을 확인하여 재인증을 진행하지 않는 프로세스이다.


```sh
root@whojes:~/$ curl "localhost:8080/test?type=REST_TEMPLATE"
{"Cookie":"null"}
root@whojes:~/$ curl "localhost:8080/test?type=REST_TEMPLATE"
{"Cookie":"JSESSIONID=407F0D466773B4F5DA32BD0B863BF0EB"}
root@whojes:~/$
root@whojes:~/$ curl "localhost:8080/test?type=WEB_CLIENT"
{"Cookie":"null"}
root@whojes:~/$ curl "localhost:8080/test?type=WEB_CLIENT"
{"Cookie":"null"}
root@whojes:~/$ curl "localhost:8080/test?type=REST_TEMPLATE"
{"Cookie":"JSESSIONID=407F0D466773B4F5DA32BD0B863BF0EB"}
root@whojes:~/$ 
```

확인 결과, httpClient 로 restTemplate 을 이용하면 두번째 호출부터는 `Set-Cookie` 로 전달받은 쿠키를 넣어서 보내주지만, webClient 를 이용하면 해당 과정이 없다. 

<hr/>
물론 이건 `restTemplate` 을 생성할 때 requestFactory 를 세팅해줘야 한다. 
```kotlin
private val restTemplate = RestTemplate().apply {
    requestFactory = HttpComponentsClientHttpRequestFactory(HttpClientBuilder.create().build())
}
```
이렇게 될 경우 `cookieSpec` 이 default 로 들어가면서 `org.apache.http.client.protocol.RequestAddCookies` 필터가 디폴트로 장착되고, 이 인터셉터가 RFC6265 에 걸맞는 작업, 즉 `Set-Cookie` 를 응답으로 받았을 때 다음 리퀘스트부터는 cookie 를 넣어주는 작업을 진행한다.

`webClient` 에는 해당 작업을 해주는 필터가 없다. 아니 못찾은건가??
그리하여, 나는 이 문제를 해결하기 위하여 다음과 같은 커스텀 작업을 해주어야 했다.
```kotlin

private val myCookies = ConcurrentHashMap<String, LinkedMultiValueMap<String, String>>()

fun set() = webClient.post()
    .uri(url)
    .accept(MediaType.APPLICATION_JSON)
    .headers { it.sessionBasicAuth() }
    .cookies { c -> myCookies[url]?.let { c.addAll(it) } }
    .bodyValue(
        mapOf(
            "userNo" to userNo,
            "message" to message,
            "socketId" to socketId
        )
    )
    .exchangeToMono {
        it.defaultCookiesWork(url)
        it.bodyToMono(object : ParameterizedTypeReference<Any>() {})
    }
    .timeout(Duration.ofMillis(connectionTimeout))
    .map { it.isSuccess }

private fun ClientResponse.defaultCookiesWork(url: String) {
    val clientResponse = this
    if (clientResponse.statusCode().is2xxSuccessful) {
        myCookies.compute(url) { _, value ->
            val map = value ?: LinkedMultiValueMap<String, String>()
            clientResponse.cookies().keys.forEach { cookieKey ->
                val cookieValue = clientResponse.cookies()[cookieKey]
                if (cookieValue != null) {
                    map[cookieKey] = cookieValue.map { it.value }
                }
            }
            map
        }
    }
}

```

로컬 메모리에 따로 url 별로 `Set-Cookie` 헤더로 들어온 쿠키를 저장하고, 다음 번 요청을 보낼 때 해당 쿠키를 실어서 보내도록 하는 코드다.

이건 `RequestAddCookies` 클래스를 보고 짠 건 아니다. 다행히 요 케이스에서는 `url`에 해당하는 서버들이 k8s 밖에 있어서 서버별 도메인이 따로 있는 상황이어서 이정도 코드로도 가능했지만, virtual service 를 통해 여러 톰캣이 같은 도메인으로 돌고 있는 k8s 환경이라면 concurrentHashMap 의 키로 url 을 사용하는 방법은 먹히지 않을 것이다.

### 아직 풀지 못한 문제
1. webClient 에도 RFC6265 스펙을 위한 필터 (RequestAddCookies 같은) 가 있을 텐데, 그걸 못찾겠다... 없는건가??
2. setCookie가 동작하지 않아서 securityContext 에서 기존 인증 컨텍스트를 유지하지 않고 매번 새로운 세션을 생성한다면 cpu 가 풀로 쓰인다는건데, 이게 그렇게까지 cpu 를 많이 먹는 작업인가?? `WebSecurityConfigurerAdapter` 에 대한 내부 동작의 이해도가 낮아서 잘 모르겠다.

참조: https://sungminhong.github.io/spring/security/

