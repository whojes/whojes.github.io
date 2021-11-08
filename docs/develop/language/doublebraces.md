---
layout: default
title: double curly braces
parent: java/kotlin
grand_parent: 개발
created_at: 2021.11.07
print_title: true
share_enable: true
permalink: develop/java/doublebraces
---

java 에서 인터페이스만 익명클래스로 생성 가능한 줄 아랐다.
```java
// interface
public interface Car {
  String getMaker();
  String getModelName();
}

public class Main {
  public static void main(String... args) {
    Car car = new Car() {
      @Override String getMaker() { return "whojes"; }
      @Override String getModelName() { return "model"; }
    };
  }
}
```

근데 클래스도 익명클래스로 만들 수 있었따!

{% raw %}
```java
public class Car {
  private String _maker;
  private String _modelName;
  public Car(String maker, String modelName) {
    this._maker = maker;
    this._modelName = modelName;
  }
  protected void setMaker(String maker) {
    this._maker = maker;
  }
}

public class Main {
  public static void main(String... args) {
    Car car = new Car("whojes", "model") {{
      setMaker("not whojes");
    }};
  }
}
```
{% endraw %}

요래 `double curly braces` 하면 익명 클래스의 객체가 되고, 이 안에는 세컨더리 컨스트럭터로 동작한다.
그래서 아니 이게 뭔가, 싶어서 조금 찾아봤는데..

---
### 동작 설명 
<https://stackoverflow.com/questions/1958636/what-is-double-brace-initialization-in-java>{:target="_blank"}

동작 원리는 대충 이렇게 된다고 한다. 

> - *The first brace creates a new Anonymous Inner Class. These inner classes are capable of accessing the behavior of their parent class.*  
> - *The second set of brace creates an instance initializers like static block in Class. If you remember core java concepts then you can easily associate instance initializer blocks with static initializers due to similar brace like struct. Only difference is that static initializer is added with static keyword, and is run only once; no matter how many objects you create.*

우선 첫번째 braces 는 익명 이너클래스를 만드는것 => interface 에서만 되는 줄 알았는데, 클래스로도 되잖아! 정말 나는 아는게 없군?
두번째 braces 는 스태틱 초기화블록과 비슷하게 초기화 블록의 역할을 한다. 다만 스태틱과 다르게 인스턴스 초기화에 사용될 뿐..  

즉, 특별하게 존재하는 구문은 아니고 다음 표현에서 인덴트를 제거한 것이다. 
```java
class Car {
  ...
}

class Main {
  public static void main(String... args) {
    Car car = new Car() { // 익명 이너 클래스
      {
        this.setSomething("crazy");
      }
    };
    System.out.println(car.getClass().getName()); // Main$1
  }
}
```

### 그러나... 

<https://jesperdj.com/2016/07/19/dont-use-the-double-brace-initialization-trick/>{:target="_blank"}  

간단히 하면 이 일명 `익명 초기화 블록` 은 직렬화 혹은 가비지 컬렉팅 과정에서 문제가 될 수 있는데, 이렇게 익명 인스턴스를 만드는 블록에서 어느 클래스 인스턴스 A의 어느 메서드를 사용한다 치면, 한 번 사용하고 버려지더라도 익명 인스턴스에서 그 메서드의 레퍼런스를 들고 있는 꼴이 돼서 그 인스턴스 A 는 더이상 사용되지 않더라도 가비지 컬렉션의 대상으로 선택되지 않는다고 한다. 즉, 메모리 릭이 발생하는데..

***잘 쓰면 되겠네 그럼..*** 은 아니고.. 단순한 환경에서 잘 알고 사용한다 치더라도 잠재적인 문제를 야기한다 하니 우선은 지양하는걸로..

---

젠장 모멘트

{% raw %}
> *First of all, it is a trick, which means that the code is obscure – if you’ve never seen this before, it’s hard to quickly understand what is happening. Especially less experienced developers will be baffled by the {{ and }} and might think that these are tokens with special significance. Also, instance initializers are a seldom used Java feature, which many developers won’t have seen before.*
{% endraw %}

ㅋㅋㅋ 젠장..  
