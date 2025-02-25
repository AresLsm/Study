# 적시에 방어적 복사본을 만들라

- Java는 안전한 언어다. 이것이 Java를 쓰는 즐거움 중 하나다. 네이티브 메소드를 사용하지 않으니  
  C, C++과 같이 안전하지 않은 언어에서 흔히 보는 Buffer Overrun, Array Overrun,  
  Wild pointer 같은 메모리 충돌 오류에서 안전하다. Java로 작성한 클래스는 시스템의 다른  
  부분에서 무슨 짓을 하든 그 불변식이 지켜진다. 메모리 전체를 하나의 거대한 배열로 다루는  
  언어에서는 누릴 수 없는 강점이다.

- 하지만 아무리 Java라해도 다른 클래스로부터의 침범을 아무런 노력 없이 다 막을 수 있는건 아니다.  
  그러니 **클라이언트가 불변식을 깨뜨리려 혈안이 되어있다고 가정하고 방어적으로 프로그래밍해야 한다.**  
  실제로도 악의적인 의도를 가진 사람들이 시스템의 보안을 뚫으려는 시도가 늘고 있다.  
  평범한 프로그래머도 순전히 실수로 클래스를 오작동하게 만들 수 있다. 물론 후자의 상황이 더 흔하다.  
  어떤 경우든 적절치 않은 클라이언트로부터 클래스를 보호하는 데 충분한 시간을 투자하는게 좋다.

- 어떤 객체든 그 객체의 허락 없이는 외부에서 내부를 수정하는 일은 불가능하다.  
  하지만 주의를 기울이지 않으면 자기도 모르게 내부를 수정하도록 허락하는 경우가 생긴다.  
  흔히 발생하는 문제다. 예를 들어, 기간(period)을 표현하는 다음 클래스는 한번 값이  
  정해지면 원래는 변하지 않도록 할 생각이었다.

```java
public final class Period {
    private final Date start;
    private final Date end;

    /**
     * @param start 시작 시각
     * @param end 끝 시각; 시작 시각보다 뒤여야 한다.
     * @throws IllegalAgrumentException 시작 시간이 종료 시각보다 늦을 때 발생한다.
     * @throws NullPointerException start나 end가 null이면 발생한다.
     */
     public Period(Date start, Date end) {
	if (start.compareTo(end) > 0) {
	    throw new IllegalArgumentException("start must be before end");
	}
	this.start = start;
	this.end = end;
     }

     public Date start() { return start; }

     public Date end() { return end; }
}
```

- 얼핏 위 클래스는 불변처럼 보이고, 시작 시각이 종료 시각보다 늦을 수 없다는 불변식이  
  무리없이 지켜질 것 같다. 하지만 `Date`가 가변이라는 사실을 이용하면 어렵지 않게 그 불변식을  
  깨뜨릴 수 있다.

```java
Date start = new Date();
Date end = new Date();
Period period = new Period(start, end);
end.setYear(78);  // period의 내부 수정
```

- 다행이 이같은 문제를 Java8 이후로는 쉽게 해결할 수 있다.  
  `Date` 대신 불변 아이템인 `Instant`를 사용하면 된다.(`LocalDateTime`, `ZonedDateTime`도 가능)  
  **`Date`는 낡은 API이니 새로운 코드를 작성할 때는 더 이상 사용하면 안 된다.**  
  하지만 앞으로 쓰지 않는다고 이 문제에서 해방되는 것은 아니다. `Date`처럼 가변인 낡은 값 타입을  
  사용하던 시절이 워낙 길었던 탓에 여전히 많은 API와 내부 구현에 그 잔재가 남아 있다. 이번 아이템은 예전에  
  작성된 낡은 코드들을 대처하기 위한 것이다.

- 외부 공격으로부터 `Period` 인스턴스의 내부를 보호하려면 **생성자에서 받은 가변 매개변수 각각을 방어적으로 복사**  
  해야 한다.(Defensive Copy) 그런 다음 `Period` 인스턴스 안에서는 원본이 아닌 복사본을 사용한다.

```java
public Period(Date start, Date end) {
    this.start = new Date(start.getTime());
    this.end = new Date(end.getTime());

    if(this.start.compareTo(this.end) > 0) {
	throw new IllegalArgumentException("start must be before end");
    }

    //..
}
```

- 새로 작성한 생성자를 사용하면 앞서의 공격은 더 이상 `Period`에 위협이 되지 않는다.  
  **매개변수의 유효성을 검사하기 전에 방어적 복사본을 만들고, 이 복사본으로 유효성 검사를 한 점에 주목하자.**  
  순서가 부자연스러워 보이겠지만, 반드시 이렇게 작성해야 한다. 멀티스레딩 환경이라면 원본 객체의 유효성을  
  검사한 후 복사본을 만드는 그 찰나의 취약한 순간에 다른 스레드가 원본 객체를 수정할 위험이 있기 때문이다.  
  방어적 복사를 매개변수 유효성 검사 전에 수행하면 이런 위험에서 해방될 수 있다. 이런 위험을  
  time-of-check(검사 시점) / time-of-use(사용 시점) 공격 혹은 TOCTOU 공격이라 한다.

- 방어적 복사에 `Date`의 `clone()`을 사용하지 않은 점에도 주목하자. `Date`는 final이 아니므로  
  `clone()`이 `Date`에 정의한게 아닐 수도 있다. 즉, `clone()`이 악의를 가진 하위 클래스의  
  인스턴스를 반환할 수도 있다. 예를 들어 이 하위 클래스는 start와 end의 참조를 private 정적 리스트에  
  담아뒀다가 공격자에게 이 리스트를 접근하는 길을 열어줄 수도 있다. 결국 공격자에게 `Period` 인스턴스  
  자체를 송두리째 넘기는 꼴이 된다. 이런 공격을 막기 위해서는 **매개변수가 제3자에 의해 확장될 수 있는**  
  **타입이라면 방어적 복사본을 만들 때 `clone()`을 사용해서는 안된다.**

- 생성자를 수정하면 앞서의 공격은 막아낼 수 있지만, `Period`는 아직도 변경 가능하다.  
  접근자 메소드가 내부의 가변 정보를 직접 드러내기 때문이다.

```java
Date start = new Date();
Date end = new Date();
Period p = new Period(start, end);
p.end().setYear(79);  // p의 내부 수정
```

- 이 공격을 막으려면 단순히 **접근자가 가변 필드의 방어적 복사본을 반환** 하면 된다.

```java
public final class Period {

    //..

    public Date start() {
	return new Date(start.getTime());
    }

    public Date end() {
	return new Date(end.getTime());
    }
}
```

- 새로운 접근자까지 갖추면 `Period`는 완벽한 불변으로 거듭난다. 아무리 악의적인, 혹은 부주의한 프로그래머도  
  시작 시각이 종료 시각보다 나중일 수 없다는 불변식을 위배할 방법은 없다.(네이티브 메소드, 리플렉션 제외)  
  `Period` 자신 말고는 가변 필드에 접근할 방법이 없으니 확실하다. 모든 필드가 객체 안에 완벽하게  
  캡슐화되어 있다.

- 생성자와 달리 접근자 메소드에는 방어적 복사에 `clone()`을 사용해도 된다. `Period`가 갖고 있는  
  `Date` 객체는 신뢰할 수 없는 하위 클래스가 아닌 `java.util.Date`임이 확실하기 때문이다.  
  그렇더라도 인스턴스를 복사하는 데는 일반적으로 생성자나 정적 팩토리를 쓰는 것이 좋다.

- 매개변수를 방어적으로 복사하는 목적이 불변 객체를 만들기 위해서만은 아니다. 메소드든 생성자든  
  클라이언트가 제공한 객체의 참조를 내부의 자료구조에 보관해야 할 때면 항시 그 객체가  
  잠재적으로 변경될 수 있는지를 생각해야 한다. 변경될 수 있는 객체라면 그 객체가 클래스에  
  넘겨진 뒤 임의로 변경되어도 그 클래스가 문제없이 동작하는지를 따져보자. 확신할 수 없다면  
  복사본을 만들어 저장해야 한다. 예를 들어 클라이언트가 건네준 객체를 내부의 `Set` 인스턴스에  
  저장하거나 `Map` 인스턴스의 key로 사용한다면, 추후 그 객체가 변경될 경우 객체를 담고 있는  
  `Set` 혹은 `Map`의 불변식이 깨질 것이다.

- 내부 객체를 클라이언트에게 건네주기 전에 방어적 복사본을 만드는 이유도 마찬가지다.  
  클래스가 불변이든 가변이든, 가변인 내부 객체를 클라이언트에게 반환할 때는 반드시 심사숙고해야 한다.  
  안심할 수 없다면 원본을 노출하지 말고 방어적 복사본을 반환해야 한다. 길이가 1이상인 배열은  
  무조건 가변임을 잊지 말자. 그러니 내부에서 사용하는 배열을 클라이언트에게 반환할 때는  
  반드시 방어적 복사를 수행해야 한다. 혹은 배열의 불변 view를 반환하는 대안도 있다.

- 이상의 모든 작업에서 우리는 _"되도록 불변 객체들을 조합해 객체를 구성해야 방어적 복사를 할 일이 줄어든다"_ 는  
  교훈을 얻을 수 있다. `Period`의 경우, Java8 이상으로 개발해도 된다면 `Instant`(`LocalDateTime`,  
  `ZonedDateTime`)을 사용하자. 이전 버전의 Java를 사용한다면 `Date` 참조 대신 `Date.getTime()`이  
  반환하는 long 정수를 사용하는 방법을 써도 된다.

- 방어적 복사에는 성능 저하가 따르고, 또 항상 쓸 수 있는 것도 아니다. 같은 패키지에 속하는 등의 이유로  
  호출자가 컴포넌트 내부를 수정하지 않으리라 확신하면 방어적 복사를 생략할 수 있다. 이러한 상황이라도  
  호출자에서 해당 매개변수나 반환값을 수정하지 말아야 함을 명확히 문서화하는 것이 좋다.

- 다른 패키지에서 사용한다고 해서 넘겨받은 가변 매개변수를 항상 방어적으로 복사해 저장해야 하는 것도 아니다.  
  때로는 메소드나 생성자의 매개변수로 넘기는 행위가 그 객체의 통제권을 명백히 이전함을 뜻하기도 한다.  
  이처럼 통제권을 이전하는 메소드를 호출하는 클라이언트는 해당 객체를 더 이상 직접 수정하는 일이  
  없다고 약속해야 한다. 클라이언트가 건네주는 가변 객체의 통제권을 넘겨받는다고 기대하는 메소드나  
  생성자에서도 그 사실을 확실히 문서에 기재해야 한다.

- 통제권을 넘겨받기로 한 메소드나 생성자를 가진 클래스들은 악의적인 클라이언트의 공격에 취약하다.  
  따라서 방어적 복사를 생략해도 되는 상황은 해당 클래스와 그 클라이언트가 상호 신뢰할 수 있을 때,  
  혹은 불변식이 깨지더라도 그 영향이 오직 호출한 클라이언트로 국한될 때로 한정해야 한다.  
  후자의 예로는 wrapper 클래스 패턴을 들 수 있다. Wrapper 클래스의 특성상 클라이언트는  
  wrapper에 넘긴 객체에 여전히 접근할 수 있다. 따라서 wrapper의 불변식을 쉽게 파괴할 수  
  있지만, 그 영향을 오직 클라이언트 자신만 받게 된다.

<hr/>

## 핵심 정리

- 클래스가 클라이언트로부터 받는, 혹은 클라이언트로 반환하는 구성요소가 가변이라면 그 요소는  
  반드시 방어적으로 복사해야 한다. 복사 비용이 너무 크거나 클라이언트가 그 요소를 잘못  
  수정할 일이 없음을 신뢰한다면 방어적 복사를 수행하는 대신 해당 구성요소를 수정했을 때의  
  책임이 클라이언트에 있음을 문서에 명시하도록 하자.

<hr/>
