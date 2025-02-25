# 스트림 병렬화는 주의해서 적용하라

- 주류 언어 중 동시성 프로그래밍 측면에서 Java는 항상 앞서갔다. 처음 릴리즈된 1996년부터  
  스레드, 동기화, wait, notify 를 지원했다. Java5부터는 동시성 컬렉션인  
  `java.util.concurrent` 라이브러리와 `Executor` 프레임워크를 지원했다.  
  Java7부터는 고성능 병렬 분해(parallel decom-position) 프레임워크인 fork-join 패키지를  
  추가했다. 그리고 Java8부터는 `parallel()` 메소드만 한 번 호출하면 파이프라인을 병렬 실행할  
  수 있는 스트림을 지원했다. 이처럼 Java로 동시성 프로그램을 작성하기가 점점 쉬워지고는 있지만,  
  이를 올바르고 빠르게 작성하는 일은 여전히 어려운 작업이다. 동시성 프로그래밍을 할 때는 안전성(safety)과  
  응답 가능(liveness) 상태를 유지하기 위해 애써야 하는데, 병렬 스트림 파이프라인 프로그래밍에서도  
  다를 바 없다. Item 45에서 다루었던 메르센 소수를 생성하는 프로그램을 다시 보자.

```java
public static void main(String[] args) {
    primes().map(p -> TWO.pow(p.intValueExact()).subtract(ONE))
        .filter(mersenne -> mersenne.isProbablePrime(50))
        .limit(20)
        .forEach(System.out::println);
}

static Stream<BigInteger> primes() {
    return Stream.iterate(TWO, BigInteger::nextProbablePrime);
}
```

- 이 프로그램의 속도를 높이기 위해 스트림 파이프라인의 `parallel()`을 호출하겠다는 순진한  
  생각을 했다 해보자. 이렇게 하면 성능은 변할까? 얼마나 빨라지거나 느려질까?  
  안타깝게도 이 프로그램은 아무것도 출력하지 못하면서 CPU는 90% 가까이 잡아먹게 된다.  
  즉 응답 불가(liveness failure) 상태가 된 것이다.

- 무슨 일이 벌어진 걸까? 프로그램이 이렇게 느려진 원인은 사실 어이없게도 스트림 라이브러리가  
  이 파이프라인을 병렬화하는 방법을 찾아내지 못했기 때문이다. 환경이 아무리 좋더라도  
  **데이터 소스가 `Stream.iterate()`이거나 중간 연산으로 `limit()`을 쓰면 파이프라인**  
  **병렬화로는 성능 개선을 기대할 수 없다.** 그런데 위의 메르센 소수 프로그램은 두 문제를  
  모두 갖고 있다. 그뿐만이 아니다. 파이프라인 병렬화는 `limit()`을 다룰 때 CPU 코어가 남는다면  
  원소를 몇 개 더 처리한 후 제한된 개수 이후의 결과를 버려도 아무런 해가 없다고 가정한다. 그런데  
  이 코드의 경우, 새로운 메르센 소수를 찾을 때마다 그 전 소수를 찾을 때보다 두 배 정도 오래 걸린다.  
  즉 원소 하나를 계산하는 비용이 대략 그 이전까지의 원소 전부를 계산한 비용만큼 든다는 뜻이다. 그래서  
  이 순진무결해보이는 파이프라인은 자동 병렬화 알고리즘이 제기능을 못하게 마비시킨다. 이 예시의 교훈은  
  간단한데, 스트림 파이프라인을 마구잡이로 병렬화하면 안된다. 성능이 오히려 끔찍하게 나빠질 수 있다.

> 간단하게 생각해 파이프라인 병렬화가 작업을 CPU 코어 수만큼 병렬로 수행한다 해보자.  
> 메르센 소수 프로그램을 쿼드 코어(코어 4개)에서 수행하면 19번째 계산까지 마치고 마지막 20번째  
> 계산이 수행되는 시점에는 CPU 코어 3개가 한가할 것이다. 따라서 21, 22, 23번째 메르센 소수를  
> 찾는 작업이 병렬로 수행되는데, 20번째 계산이 끝나더라도 이 계산들은 끝나지 않는다. 각각 20번째  
> 계산보다 2배, 4배, 8배 더 시간이 소요되기 때문이다.

- 대체로 **스트림의 소스가 `ArrayList`, `HashMap`, `HashSet`, `ConcurrentHashMap`의**  
  **인스턴스이거나 배열, int범위, long범위일 때 병렬화의 효과가 가장 좋다.** 이 자료구조들은 모두  
  데이터를 원하는 크기로 정확하고 손쉽게 나눌 수 있어서 일을 다수의 스레드에 분배하기에 좋다는 특징이  
  있다. 나누는 작업은 `Spliterator`가 담당하며, 이 객체는 `Stream`이나 `Iterable`의  
  `spliterator()` 메소드로 얻어올 수 있다.

- 이 자료구조들의 또 다른 중요한 공통점은 원소들을 순차적으로 실행할 때의 참조 지역성(locality of reference)이  
  뛰어나다는 것이다. 이웃한 원소의 참조들이 메모리에 연속해서 저장되어 있다는 뜻이다. 하지만 참조들이 실제로 가리키는  
  객체는 메모리에서 서로 떨어져있을 수 있는데, 그러면 참조 지역성이 나빠진다. 참조 지역성이 낮으면 스레드는 데이터가  
  주 메모리에서 캐시 메모리로 전송되어 오기를 기다리며 대부분의 시간을 멍하니 보내게 된다. 따라서 참조 지역성은  
  다량의 데이터를 처리하는 벌크 연산을 병렬화할 때 아주 중요한 요소로 작용한다. 참조 지역성이 가장 뛰어난 자료구조는  
  기본 타입의 배열이다. 기본 타입 배열에서는 참조가 아닌 데이터 자체가 메모리에 연속해서 저장되기 때문이다.

- 스트림 파이프라인의 종단 연산의 동작 방식 역시 병렬 수행 효율에 영향을 준다. 종단 연산에서 수행하는  
  작업량이 파이프라인 전체 작업에서 상당 비중을 차지하면서 순차적인 연산이라면 파이프라인 병렬 수행의  
  효과는 제한될 수밖에 없다. 종단 연산 중 병렬화에 가장 적합한 것은 축소(reduction)이다. 축소는  
  파이프라인에서 만들어진 모든 원소를 하나로 합치는 작업으로, `Stream`의 `reduce()` 메소드들 중 하나,  
  혹은 `min()`, `max()`, `count()`, `sum()` 같이 완성된 형태로 제공되는 메소드 중 하나를  
  선택해서 수행한다. `anyMatch()`, `allMatch()`, `noneMatch()` 처럼 조건에 맞으면 바로  
  반환되는 메소드도 병렬화에 적합하다. 반면 가변 축소(mutable reduction)를 수행하는 `Stream`의  
  `collect()` 메소드는 병렬화에 적합하지 않다. 컬렉션들을 합치는 부담이 크기 때문이다.

- 직접 구현한 `Stream`, `Iterable`, `Collection`이 병렬화의 이점을 제대로 누리게 하고 싶다면  
  `spliterator()` 메소드를 반드시 재정의하고 결과 스트림의 병렬화 성능을 강도 높게 테스트해야 한다.  
  고효율 `spliterator()`를 작성하기란 절대 쉬운 일이 아니다.

- **스트림을 잘못 병렬화하면 응답 불가를 포함해 성능이 나빠질 뿐만 아니라 결과 자체가 잘못되거나**  
  **예상 못한 동작이 발생할 수 있다.** 결과가 잘못되거나 오동작하는 것은 안전 실패(safety failure)라 한다.  
  안전 실패는 병렬화한 파이프라인이 사용하는 mappers, filters, 혹은 프로그래머가 제공한 다른 함수 객체가  
  명세대로 동작하지 않을 때 벌어질 수 있다. `Stream` 명세는 이때 사용되는 함수 객체에 관한 엄중한 규약을  
  정의해놨다. 예를 들어 `Stream`의 reduce 연산에 건네지는 accumulator(누적기)와 combiner(결합기) 함수는  
  반드시 결합법칙을 만족하고(associative), 간섭받지 않고(non-interfering), 상태를 갖지 않아야(stateless)한다.  
  이상의 요구사항을 지키지 못하는 상태라도 파이프라인을 순차적으로만 수행한다면야 올바른 결과를 얻을 수도 있다.  
  하지만 병렬로 수행하면 참혹한 실패로 이어지기 십상이다.

- 따라서 앞서의 병렬화한 메르센 소수 프로그램은 설혹 완료되더라도 출력된 소수의 순서가 올바르지  
  않을 수 있다. 출력 순서를 순차 버전처럼 정렬하고 싶다면 종단 연산 `forEach()`를  
  `foreEachOrdered()`로 바꿔주면 된다. 이 연산은 병렬 스트림들을 순회하며 소수를 발견한 순서대로  
  출력되도록 보장해줄 것이다.

- 심지어 데이터 소스 스트림이 효율적으로 나눠지고, 병렬화하거나 빨리 끝나는 종단 연산을 사용하고,  
  함수 객체들도 간섭하지 않더라도, 파이프라인이 수행하는 진짜 작업이 병렬화에 드는 추가 비용을  
  상쇄하지 못한다면 성능 향상은 미미할 수 있다. 실제로 성능이 향상될지를 추정해보는 간단한 방법이 있다.  
  스트림 내의 원소 수와 원소당 수행되는 코드 줄 수를 곱해보자. 이 값이 최소 수십만은 되어야  
  성능 향상을 맛볼 수 있다.

- 스트림 병렬화는 오직 성능 최적화 수단임을 기억해야 한다. 다른 최적화와 마찬가지로 변경 전후로 반드시 성능을  
  테스트하여 병렬화를 사용할 가치가 있는지 확인해야 한다. 이상적으로는 운영 시스템과 흡사한 환경에서  
  테스트하는 것이 좋다. 보통은 병렬 스트림 파이프라인도 공통의 fork-join 풀에서 수행되므로, 즉 같은  
  스레드 풀을 사용하므로 잘못된 파이프라인 하나가 시스템의 다른 부분의 성능에까지 악영향을 줄 수 있음을 유념하자.

- 이 아이템을 보고 앞으로 스트림 파이프라인을 병렬화할 일이 적게 느껴진다면, 그건 진짜 그렇기 때문이다.  
  스트림을 많이 사용하는 수백만줄짜리 코드를 여러 개 관리하는 프로그래머도 스트림 병렬화가 효과를 보는  
  경우가 많지 않다고 한다. 그렇다고 스트림을 병렬화하지 말아야 한다는 뜻은 아니다.  
  **조건이 잘 갖춰지면 `parallel()` 메소드 호출 하나로 거의 프로세서 코어 수에 비례하는 성능 향상을 만끽할 수 있다.**  
  머신러닝과 데이터 처리 같은 특정 분야에서는 이 성능 개선의 유혹을 뿌리치기 어려울 것이다.

- 스트림 파이프라인 병렬화가 효과를 제대로 발휘하는 간단한 예시를 보자.  
  아래는 n보다 작거나 같은 소수의 개수를 계산하는 함수다.

```java
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .mapToObj(BigInteger::valueOf)
	.filter(i -> i.isProbablePrime(50))
	.count();
}
```

- 위 코드에 `parallel()` 하나만 추가하면 시간 단축이 꽤나 된다.

```java
static long pi(long n) {
    return LongStream.rangeClosed(2, n)
        .parallel()
        .mapToObj(BigInteger::valueOf)
	.filter(i -> i.isProbablePrime(50))
	.count();
}
```

- 무작위 수들로 이뤄진 스트림을 병렬화하고 싶다면 `ThreadLocalRandom`, `Random`보다는  
  `SplittableRandom` 인스턴스를 사용하자. `SplittableRandom`은 정확이 이런 경우에 사용하고자  
  설계된 것이라 병렬화하면 성능이 선형으로 증가한다. 한편 `ThreadLocalRandom`은 단일 스레드에서  
  사용하고자 만들어졌다. 병렬 스트림용 데이터 소스로도 사용할 수는 있지만 `SplittableRandom`만큼  
  빠르지는 않을 것이다. 마지막으로 그냥 `Random`은 모든 연산을 동기화하기 때문에 병렬 처리하면  
  최악의 성능을 보일 것이다.

<hr/>

## 핵심 정리

- 계산도 올바로 수행하고 성능도 빨라질 것이라는 확신 없이는 스트림 파이프라인 병렬화는 시도조차 하지 말자.  
  스트림을 잘못 병렬화하면 프로그램을 오동작하게 만들거나 성능을 급격히 떨어뜨린다. 병렬화하는 편이 낫다고  
  믿더라도 수정 후의 코드가 여전히 정확한지 확인하고 운영 환경과 유사한 조건에서 수행해보며 성능지표를  
  유심히 관찰하자. 그래서 계산도 정확하고 성능도 좋아졌음이 확실해졌을 때, 오직 그럴 때만 병렬화 버전을  
  운영 코드에 반영하자.

<hr/>
