---
layout: default
title: Java collectors
parent: java/kotlin
grand_parent: 개발
created_at: 2019.12.27
print_title: true
share_enable: true
permalink: develop/java/collector
---

Collector 는, 아무래도, Collection Framework 의 꽃이 아닐까? 이름부터 그렇잖아. Collector!

stream 의 끝에는 거의 forEach 나 collect 가 기다리고 있다. 여러 필터, 정렬, 매핑 등등으로부터 살아남은 스트림의 엘레먼트들은 forEach 를 만나 소멸하거나 collector 를 만나 새로운 Return Type으로 반환이 되게 된다. forEach는 argument 로 Consumer<? super ElementType> 를 받으므로 그냥 함수 꼴을 하고 있는 pseudo-일급객체 하나를 넣어주면 되지만, collect는 argument가 `Collector<? super ElementType, A, R>` 을 받는데, 아니 도대체가 javadoc만읽어서는 잘 이해가 되지 않았던 것이다. functional interface 도 아니고, 어차피 `java.util.stream.Collectors` 에 유용한건 이미 다있으니 그냥 갖다가 쓰면 되니까 썼는데, 그래도 하나쯤 custom collector 가 있어야하지 않을까? 하는 노파심에, 그래! 해보자 하고 알아보게 되었다.


Collector<T, A, R> 클래스는 Target, Accumulator, Return의 generic 타입을 가지는 인터페이스이고, 이를 구현한 클래스가 supplier, accumulator, combiner, finisher와 characteristics 를 오버라이드하여 stream 의 collect 메서드에서 사용할 수 있게 된다. 


이를 이용해 `java.util.stream.Collectors` 의 `joining(String delimeter)` 메서드를 직접 구현해보면,

```java
public class CustomCollector<T> extends Collector<T, StringBuilder, String> {
  private String delimeter_;
  private CustomCollector(String del) {
    delimeter_ = del;
  }
  public static <T> CustomCollector<T> joining(String del) {
    return new CustomCollector<>(del);
  }
  
  @Override
  public Supplier<StringBuilder> supplier() {
    return StringBuilder::new;
  }
  @Override
  public BiConsumer<StringBuilder, T> accumulator() {
    return (sb, value) -> sb.append(value).append(delimeter_);
  }
  @Override
  public BinaryOperator<StringBuilder> combiner() {
    return (sb1, sb2) -> sb1.append(sb2);
  }

  @Override
  public Function<StringBuilder, String> finisher() {
    return sb -> sb.subSequence(0, sb.length() - delimeter_.length()).toString();
  }

  @Override
  public Set<Characteristics> characteristics() {
    return ImmutableSet.of(Characteristics.UNORDERED);
  }
}
```

정도가 되겠다. Accumulator 로 StringBuilder 를 사용했고, combiner(2016년 스택오버플로에 의하면 parallel stream으로 병렬적으로 여러 작업이 이루어지고 이 작업들을 합칠 때만 사용됨)및 finisher, delimeter 를 삽입하는 방법은 최적화를 하지 않고 생각나는대로 짜서 효율은 모르겠지만, 어쨌든 이렇게 해 놓으면 `java.util.stream.Collectors.joining` 과 동일하게 사용할 수 있다. 물론 실제로는 `java.util.StringJoiner` 를 accumulator로 사용하는 것 같다.

```java  
void main() {
    List<String> list = Arrays.asList("dalee","jumping","higher");
    String value = list.stream().collect(CustomCollector.joining("::"));
    System.out.println(value);
    // dalee::jumping::higher  
}  
```  
  
`Collector.of` 메서드를 이용하면 이것 역시 함수형태(함수형 프로그래밍을 위해 함수 형태를 일급객체로 사용하기위한 Java의 몸부림이라고 볼수있겠다.) argument를 이용해 사용할 수 있다.

```java
String delimeter = "::";
Collector<String, ?, String> c = Collector.of(
  StringBuilder::new,  // supplier
  (sb, t) -> sb.append(t).append(delimeter), // accumulator
  StringBuilder::append, // combiner
  sb -> sb.subSequence(0, sb.length() - delimeter.length()).toString(), // finisher
  Characteristics.UNORDERED // characteristics
);
```

더하여, double colon 으로 이뤄지는 method reference 에 관하여, 함수 형태의 람다식의 사용을 간편하게 해주는 것으로, 크게 4가지의 사용법이 있다.

```java
(args) -> Class.staticMethod(args) === Class::staticMethod
(obj, args) -> obj.instanceMethod(args) === ObjectType::instanceMethod
(args) -> obj.instanceMethod(args) === obj::instanceMethod
(args) -> new ClassName(args) === ClassName::new
```

즉, 다음과 같이 줄일 수 있다.

```java
Supplier<StringBuilder> s = () -> new StringBuilder();
Supplier<StringBuilder> s = StringBuilder::new;
// 동치

Function<String, StringBuilder> f = (input) -> new StringBuilder(input);
Function<String, StringBuilder> f = StringBuilder::new;
BiFunction<String, String, StringBuilder> f = StringBuilder::new; // Error (StringBuilder 의 생성자는 param 을 1개만 받음)
// 동치

BiFunction<String, String, String> bf = (input1, input2) -> input1 + input2;
BiFunction<String, String, String> bf = String::concat;
// 동치

Integer temp = 4;
Comparable<Integer> cc = input -> temp.compareTo(input);
Comparable<Integer> cc = temp::compareTo;
// 동치미
```
