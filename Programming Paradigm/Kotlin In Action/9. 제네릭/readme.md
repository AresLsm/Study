# 제네릭

- 코틀린 제네릭은 Java와 아주 비슷하다. 제네릭 함수와 클래스를 Java와 비슷하게 선언할 수 있다.
- Java와 마찬가지로 제네릭 타입의 타입 인자는 컴파일 시점에만 존재한다.  
  (실행 시점에는 타입 정보가 소거된다.)
- 타입 인자가 실행 시점에 지워지므로 타입 인자가 있는 타입(제네릭 타입)을 is 연산자로 검사할 수 없다.
- 인라인 함수의 타입 매개변수를 reified로 표시해 실체화하면, 실행 시점에 그 타입을 is로 검사하거나 `java.lang.Class` 인스턴스를 얻을 수 있다.
- 변성은 기저 클래스가 같고 타입 파라미터가 다른 두 제네릭 타입 사이의 상위, 하위 타입 관계가 타입 인자 사이의 상위, 하위 타입 관계에  
  어떤 영향을 받는지를 명시하는 방법이다.
- 제네릭 클래스의 타입 파라미터가 out 위치에서만 사용되는 경우(producer), 그 타입 파라미터를 out으로 표시해 공변적으로 만들 수 있다.
- 공변적인 경우와 반대로 제네릭 클래스의 타입 파라미터가 in 위치에서만 사용되는 경우(consumer) 그 타입 파라미터를 in으로 표시해 반공변적으로 만들 수 있다.
- 코틀린의 읽기 전용 `List` 인터페이스는 공변적이다. 따라서 `List<String>`은 `List<Any>`의 하위 타입이다.
- 함수 인터페이스는 첫 번째 타입 파라미터에 대해서는 반공변적이고, 두 번째 타입 파라미터에 대해서는 공변적이다.
- 코틀린에서는 제네릭 클래스의 공변성을 전체적으로 지정하거나(선언 지점 변성), 구체적인 사용 위치에서 지정할 수 있다.(사용 지점 변성)
- 제네릭 클래스의 타입 인자가 어떤 타입인지 정보가 없거나, 타입 인자가 어떤 타입인지가 중요하지 않을 때 star projection 구문을 사용할 수 있다.
