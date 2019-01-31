# 소개

## 타입 클래스 해부

- 타입 클래스는 상속이나 기존 코드를 고치지 않고 기능을 확장 할 수 있게 해준다.
- 타입 클래스 패턴은 타입 클래스, 특정 타입에 대한 인스턴스, 사용자에게 노출할 인터페이스 메서드로 구성된다.

### 타입 클래스

- 타입 클래스는 인터페이스나 API 같은 것이다.
- Cats에서 타입 클래스는 최소 하나 이상의 타입 파라미터를 가진다.
- 예를 들면 "JSON 직렬화"는 다음 과 같다.
  ```scala
  // 단순한 JSON AST 정의하기
  sealed trait Json
  final case class JsObject(get: Map[String, Json]) extends Json final case class JsString(get: String) extends Json
  final case class JsNumber(get: Double) extends Json
  case object JsNull extends Json
  // JSON 직렬화 동작은 이 트레이트에 인코딩 되어 있다.
  trait JsonWriter[A] {
    def write(value: A): Json
  }
  ```

- 이 예제에서 `JsonWriter`가 타입 클래스다.

### 타입 클래스 인스턴스

- 인스턴스는 스칼라 기본 타입이나 여러분이 만든 도메인 타입에 대한 타입 클래스의 구현이다.
- 스칼라에서 인스턴스는 타입 클래스를 구체적으로 구현을 정의한 것이고 `implicit` 태그를 붙인다.
  ```scala
  final case class Person(name: String, email: String)
  object JsonWriterInstances {
    implicit val stringWriter: JsonWriter[String] =
      new JsonWriter[String] {
        def write(value: String): Json =
          JsString(value)
      }
    implicit val personWriter: JsonWriter[Person] =
      new JsonWriter[Person] {
        def write(value: Person): Json =
          JsObject(Map(
            "name" -> JsString(value.name),
            "email" -> JsString(value.email)
          ))
  }
  // 그밖에...
  }
```

### 타입 클래스 인터페이스

- 타입 클래스 인터페이스는 사용자에게 제공하는 기능이다.
- 인터페이스는 `implicit` 파라미터로 타입 클래스의 인스턴스를 인자로 받는 메서드다.
- 인터페이스를 만드는 방법은 두가지가 있는데 하나는 `Interface Objects`고 다른 하나는 `Interface Syntax`다.

#### Interface Object

- 인터페이스를 만드는 가장 간단한 방법은 싱글톤 객체에 메서드를 만드는 것이다.
  ```scala
  object Json {
    def toJson[A](value: A)(implicit w: JsonWriter[A]): Json =
        w.write(value)
  }
  ```
- 이 객체를 쓰려면 타입 클래스 인스턴스를 import 해줘야한다.
  ```scala
  import JsonWriterInstances._

  Json.toJson(Person("Dave", "dave@example.com"))
  // res4: Json = JsObject(Map(name -> JsString(Dave), email -> JsString(dave@example.com)))
  ```
- 타입 클래스 인스턴스는 `implicit`로 되어 있기 때문에 컴파일러가 타입에 맞는 적절한 인스턴스를 찾아서
  인자로 넣어준다. 직접 쓰면 아래와 같다.
  ```scala
  Json.toJson(Person("Dave", "dave@example.com"))(personWriter)
  ```

#### Interface Syntax

- 인터페이스를 만드는 두번째 방법은 인터페이스 메서드로 기존에 있는 타입에 기능을 확장하는 확장 메서드를
  쓰는 방법이다. Cats에서는 이것을 타입 클래스의 `syntax`라고 부른다.
  ```scala
  object JsonSyntax {
      implicit class JsonWriterOps[A](value: A) {
        def toJson(implicit w: JsonWriter[A]): Json =
          w.write(value)
    }
  }
  ```
- `Interface Syntax`를 사용하려면 타입 클래스 인스턴스를 import 할 때 함께 import 해줘야한다.
  ```scala
  import JsonWriterInstances._
  import JsonSyntax._

  Person("Dave", "dave@example.com").toJson
  // res6: Json = JsObject(Map(name -> JsString(Dave), email -> JsString(dave@example.com)))
  ```
- 역시 컴파일러가 아래 처럼 적절한 implicit 파라미터를 채워준다.
  ```scala
  Person("Dave", "dave@example.com").toJson(personWriter)
  ```

#### The implicitly Method

- 스칼라 스탠다드 라이브러리는 `implicitly`라고 하는 제너릭 타입 클래스 인터페이스를 제공한다.
  ```scala
  def implicitly[A](implicit value: A): A =
    value
  ```
- `implicitly`는 `implicit` 스코프 안에 있는 어떤 값이든 쓸 수 있다.
  ```scala
  import JsonWriterInstances._
  // import JsonWriterInstances._

  implicitly[JsonWriter[String]]
  // res8: JsonWriter[String] = JsonWriterInstances$$anon$1@20da369
  ```
- Cats에 있는 대부분의 타입 클래스는 인스턴스를 부르는 다른 방법을 제공한다.
- `implicitly`는 디버깅 용으로 적합하다.
- 코드의 일반 흐름에 `implicitly` 호출을 넣어 컴파일러가 타입 클래스의 인스턴스를 찾을 수 있도록 하고
  모호한 `implicit` 에러가 없도록 한다.

## Implicits 사용하기

- 스칼라에서 타입 클래스를 쓴다는 것은 `implicit` 값과 `implicit` 파라미터를 쓴다는 말이다.
- 효과적으로 쓰려면 몇가지 규칙을 알아야 한다.

### Implicits 패키징하기

- implicit는 최상위에 정의하기 보다 객체나 트레이트 안에 정의하는 것이 좋다.
- 앞선 예제에서는 `JsonWriterInstances`라는 객체 안에 타입 클래스 인스턴스를 정의했다.
- `JsonWriter`를 동반 객체에 두는 것도 같다.
- 타입 클래스의 인스턴스를 동반 객체에 두는 것은 스칼라에서 특별하다. 왜냐하면 implicit 스코프라는 곳에서
  동작하기 때문이다.

### Implicit 스코프

- 위에서 본 것처럼 컴파일러가 타입에 따라 대상이 되는 인스턴스 중에서 타입 클래스 인스턴스 찾는다.
- 다음 예제는 `JsonWriter[String]` 타입의 인스턴스를 찾을 것이다.
  ```scala
  Json.toJson("A string!")
  ```
- 컴파일러는 호출하는 곳에 있는 implicit 스코프에서 인스턴스 대상이 되는 것들 중에 찾는다. 대략 다음과
  같이 구성된다:
  - 로컬 또는 상속된 정의
  - import된 정의
  - 타입 클래스의 동반 객체 또는 파라미터 타입(여기서는 JsonWriter 또는 String) 안에 있는 정의
- 정의는 implicit 키워드가 지정된 경우에만 implicit 스코프에 포함한다.
- 컴파일러가 여러개 정의를 발견하면 `ambiguous implicit values error`가 난다.
  ```scala
  implicit val writer1: JsonWriter[String] =
  JsonWriterInstances.stringWriter

  implicit val writer2: JsonWriter[String] =
    JsonWriterInstances.stringWriter

  Json.toJson("A string")
  // <console>:23: error: ambiguous implicit values:
  // both value stringWriter in object JsonWriterInstances of type => JsonWriter[String]
  //  and value writer1 of type => JsonWriter[String]
  // match expected type JsonWriter[String] // Json.toJson("A string")
  //
  ```

- implicit를 해석하는 정확한 규칙인 이것 보다 더 복잡하다. 하지만 이 책에서는 별로 관련이 없는 내용이다.
- 우리의 목적을 위해 타입 클래스 인스턴스를 패키징하는 방법은 대략 4가지가 있다:
  1. `JsonWriterInstances` 처럼 객체 안에 두는 방법
  2. 트레이트 안에 두는 방법
  3. 타입 클래스의 동반 객체에 두는 방법
  4. 타입 파라미터의 동반 객체에 두는 방법
- 1번 방법은 import 해서 인스턴스를 스코프 안으로 가져올 수 있고 2번 방법은 상속을 해서 스코프 안으로
  가져올 수 있고 3번과 4번 방법은 우리가 쓰려고 하지 않아도 항상 스코프 안에 들어와 있다.

### 재귀적인 Implicit 해석

- 타입 클래스와 implicit의 힘은 implicit 정의와 implicit 될 수 있는 인스턴스를 연결하는 컴파일러의
  능력에 달려 있다.
- 앞의 예제에서 모든 타입 클래스 인스턴스는 implicit val로 주입되었다. 단순했고 실제로 두가지 방법으로
  인스턴스를 정의할 수 있다:
  1. 요구되는 타입의 implicit vals로 인스턴스를 구체화해서 정의하기
  2. 다른 타입 클래스 인스턴스로 부터 인스턴스를 생성하기 위해 implicit 메서드로 정의하기
- 왜 다른 타입 클래스 인스턴스로 부터 인스턴스를 생성해야하는지 `Option`의 `JsonWriter`를 정의하는
  예를 들어보자.
- 애플리케이션에서 다루는 모든 A에 대한 `JsonWriter[Option[A]]`가 필요하다.
- `implicit vals`로 된 라이브러리를 만들어서 문제를 풀 수 있다.
  ```scala
  implicit val optionIntWriter: JsonWriter[Option[Int]] = ???

  implicit val optionPersonWriter: JsonWriter[Option[Person]] = ???
  // and so on...
  ```
- 하지만 확장성이 떨어진다. 결국 모든 A에 대해 두개의 `implicit vals`가 필요하다. 하나는 `A`를 위한 것이고
  하나는 `Option[A]`를 위한 것이다:
- 운좋게도 인스턴스 `A`의 일반 생성자를 기반으로 `Option[A]`를 다루기 위한 코드를 추상화할 수 있다.
  - 옵션이 `Some(aValue)`라면 A의 writer로 `aValue`를 write한다.
  - 옵션이 `None`라면 `JsNull`을 리턴한다.
  ```scala
  implicit def optionWriter[A]
    (implicit writer: JsonWriter[A]): JsonWriter[Option[A]] =
      new JsonWriter[Option[A]] {
        def write(option: Option[A]): Json =
          option match {
            case Some(aValue) => writer.write(aValue)
            case None         => JsNull
    }
  }
  ```
- 이 메서드는 `A`의 기능으로 된 implicit 파라미터로 `Option[A]`에 대한 `JsonWriter`를 생성한다.
- 컴파일러가 다음과 같은 코드를 보면:
  ```scala
  Json.toJson(Option("A string"))
  ```
- `JsonWriter[Option[String]]` implicit를 찾는다. `JsonWriter[Option[A]]`에 대한 implicit
  메서드를 찾는다.
  ```scala
  Json.toJson(Option("A string"))(optionWriter[String])
  ```
- 그리고 재귀적으로 `optionWriter` 파라미터에 있는 `JsonWriter[String]`를 찾는다.
  ```scala
  Json.toJson(Option("A string"))(optionWriter(stringWriter))
  ```
- 이런식으로 implicit 해석은 올바른 전체 타입의 타입 클래스 인스턴스를 호출하는 조합을 찾기 위해
  가능한 implicit 정의들의 조합의 공간을 통해 검색 된다.
- `implicit def`로 타입 클래스 인스턴스 생성자를 만들 때 메서드의 파라미터에 implicit 표시해서
  implicit 파라미터로 만드는 것을 잊지 말아야한다. 만약 implicit 표시를 안하면 컴파일러가 implicit를
  해석할 때 채워주지 않는다.
- non-implicit 파라미터를 사용하는 implicit 메서드는 스칼라에서 implicit conversion이라고 다른
  패턴으로 부른다. 이것은 앞에서 본 `Interface Syntax`와 는 다르다. 이 경우는 `JsonWriter`가
  확장 메서드가 있는 implicit 클래스기 때문이다.
- implicit conversion는 요즘 스칼라 코드에서는 꺼려하는 오래된 프로그래밍 패턴이다.
- 다행이 컴파일러가 이 패턴을 쓰면 경고해준다.
- `scala.language.implicitConversions`를 import 해서 수동으로 implicit conversions를
  활성화 할 수 있다.
