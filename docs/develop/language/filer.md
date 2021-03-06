---
layout: default
title: Java filer
parent: java/kotlin
grand_parent: 개발
created_at: 2020.01.31
print_title: true
share_enable: true
permalink: develop/java/filer
---

구글 검색하면 자꾸 Filter 아니냐고 내가 원하는 결과는 감춰서 보여주는 Filer. 기존 Annotation processing 개발 시에 진짜 아무것도 모르고 개발을 해서 컴파일 시간이 과하게 길었었는데, 이번 기회에 제대로 공부하여 수정하기로 마음먹었다. 
​
AbstractProcessor를 상속할 경우에 구현해야 하는 init의 파라미터 ProcessingEnvironment는 많은 툴을 제공해준다. Type 관련 유틸, Element 관련 유틸, Messager, Filer 등을 제공해줘서 프로세싱 할 때 유용하게 사용할 수 있다. 

```java
private Elements elementUtils;
private Types typeUtils;
private Messager messager;
private Filer filer;

@Override
public synchronized void init(ProcessingEnvironment env) {
    super.init(env);
    this.elementUtils = env.getElementUtils();
    this.typeUtils = env.getTypeUtils();
    this.messager = env.getMessager();
    this.filer = env.getFiler();
}
```

jvm 은 자바 소스코드를 클래스파일로 컴파일 할 때 영어로 적힌 소스 코드를 마크업 언어 정도로 해석한다. 모든 line은 Element이고, 각각의 속성에 따라 엘러먼트의 타입이 있다. 가령

```java
package com.example;	// PackageElement
public class Foo {		// TypeElement
	private int a;		// VariableElement
	private Foo other; 	// VariableElement
	public Foo () {} 	// ExecuteableElement
}
```

이런 식이다.  elementUtils 는 이 Element를 잘 조작할 수 있게 도와준다. 

`public class Foo` 라는 것은 TypeElement 이고, 이는 그 자체로는 Foo 라는 클래스의 정보를 갖고 있지 않다. `TypeElement.asType()` 메서드는 TypeMirror 클래스의 객체를 리턴하며, 이 타입미러가 Foo 라는 클래스 타입의 정보를 가지고 있고, typeUtils 는 이 타입미러를 잘 조작할 수 있게 도와준다. (타입미러는 Java의 타입을 나타내는 클래스로, 타입이라 함은 primitive types, declared type(class/interfaces), array types, type variables, null 등 모든 타입을 포괄하는 개념이다.) messager 는 메세지를 잘 예쁘게 꾸며서 내보낼 수 있는...그런...
​
이 글을 왜 쓰냐하면, 개인적인 깨달음이 있었기 때문이다. 새로운 것을 배울 때는 끝까지 배우고 쓰자! 하는 마인드가 바로 그것인데, 나는 보통 새로운 것을 배울 때 '오 이렇게 하면 되네! 여기에 내가 아는 이만큼을 추가하면 이렇게 할 수 있겠네!' 로 하기 때문에 매뉴얼을 덜 읽고 코딩을 하는 모양새가 됐었기 때문이다. 뭔 말인고 하니, 아! AbstractProcessor 를 상속하고 process 메서드 안에 코딩을 해놓으면 컴파일때 작동하네! 하고는 끝까지 안 읽고 그냥 일반 FileWriter로 파일을 쓰고, 그 때 쓴 파일은 컴파일이 안되니 전체 컴파일을 다시 하고, 그러다보니 빌드타임이 무진장 길어지고, ...

AbstractProcessing 에서 SourceGen을 위한 `javax.annotation.processing.Filer` 클래스가 있다. Filer.createSourceFile의 인자로 full classname 을 넣어주면 `javax.tools.JavaFileObject` 객체가 생기는데, 이 객체로 file write를 하면, 빌드디렉토리의 `generated/sources/annotationProcessor/java/main/` 디렉토리 하위에 소스파일이 생기게 되고, 여러 번의 라운드가 끝나고 나서야 `src/` 하위 및 `build/generated/source/` 하위의 소스들이 컴파일되어 바이트 코드가 된다고 하네.

<p align="center">
  <br><img alt="img-name" src="/assets/images/java/filer_1.png" class="content-image-1"><br>
</p>
<p align="center">
  <br><img alt="img-name" src="/assets/images/java/filer_2.png" class="content-image-1"><br>
  <em>애너테이션 프로세싱은 한 빌드에서 여러 라운드를 돈다. filer로부터 생성된 파일이 또다시 프로세싱 해야 하는 애너테이션을 달고 있을 수도 있으니 한 프로세스에서 새로 생긴 파일들만 input으로 하여 다시금 processing을 한다.</em>
  <br>
</p>

내가 했던 기묘한 짓, 즉 빌드디렉토리가 아닌 소스디렉토리에 소스파일을 생성하고는 파일이 없으면 생성, 있으면 클래스파일 생성 후 파일 삭제 하도록 로직을 짠 후 컴파일을 총 두 번 했던 짓은 쓰레기같은 짓이라는 점이다. 이 사실을 알게 된 후가 되니 누군가 내가 했던 것처럼 코드를 짜놨다면 개쓰레기같이 코드를 짰다고 생각할 만한 개짓거리다. 아, 이거 깃헙에 소스 올려놨는데, 진짜로 소스공개 할 때는 하늘에 한 점 부끄러움이 없을 때만 공개해야한다는 깨달음을 얻어버리고 말았다.

​
그까짓 것 잊어버리고, 다시 Filer로 돌아와서,  
Filer에서는 createClassFile, createResource, createSourceFile 세 가지의 비범한 메서드들을 제공한다. 

createResource 는 첫번째 인자로 `javax.tools.FileManager.Location` 을 받는데, 이넘으로 정해져있는 패스 외의 다른 경로를 사용하는 법을 결국 찾지 못해 어차피 컴파일 할 파일들도 아닌데 그냥 java.nio.File 로 사용하였다.

createSourceFile 은 나는 2줄정도밖에 되지 않는 소스코드로 그냥 스트링으로 만들었으나, <https://github.com/square/javapoet>{:target="_blank"} 에서 제공하는 JavaPoet 클래스를 이용하여 더욱 쉽게 자바 소스파일을 생성할 수 있다고 하니 심심하면 사용해보도록 하자. 

<p align="center">
  <br><img alt="img-name" src="/assets/images/java/filer_3.png" class="content-image-1"><br>
  <em>Hello, JavaPoet 을 출력하는 HelloWorld.java 파일을 자동생성하는 소스코드. (exiting)이 붙어있는 설명이 인상적이다.</em>
  <br>
</p>

createClassFile 은 바이트코드를 그대로 작성하여 클래스파일을 직접 만든다는건데, kapex의 말에 따르면 웬만한 경우에 굳이 그럴 필요가 없을 것이고, 그는 꼭 그래야만 한다면 이런 질문을 작성할 수준으로는 안된다고 완곡하게 일침을 가했다. [stack overflow](https://stackoverflow.com/questions/20952897/java-6-annotation-processor-with-filer-createclassfile-implementation){:target="_blank"}

Filer에서 작성된 소스코드는 컴파일 시에 자동으로 함께 컴파일된다고 하니, 내가 했던 쓰레기짓은 다시는 저지르지 않도록 하자.

