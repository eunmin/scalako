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
  final case class JsObject(get: Map[String, Json]) extends Json
  final case class JsString(get: String) extends Json
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
- 다음 예제는 `JsonWriter[String]` 타입의 인스턴스를 찾을 것이다.
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

## 연습 문제

```scala

final case class Cat(name: String, age: Int, color: String)

trait Printable[A] {
  def format(value: A): String
  def print(value: A): Unit = println(format(value))
}

object PrintableInstances {
  implicit val intInstance = new Printable[Int] {
    override def format(value: Int): String = s"$value"
  }

  implicit val stringInstance = new Printable[String] {
    override def format(value: String): String = value
  }

  implicit val catInstance = new Printable[Cat] {
    override def format(value: Cat): String = s"${value.name} is a ${value.age} year-old ${value.color} cat."
  }
}

object Printable {
  def format[A](value: A)(implicit printable: Printable[A]): String =
    printable.format(value)

  def print[A](value: A)(implicit printable: Printable[A]): Unit =
    println(format(value))
}

object PrintableSyntax {
  implicit class PrintableOps[A](value: A) {
    def format(implicit printable: Printable[A]): String = printable.format(value)

    def print(implicit printable: Printable[A]): Unit = printable.print(value)
  }
}

object Main extends App {
  import PrintableInstances._
  import PrintableSyntax._

  "Test".print
  123.print
  Cat("Jerry", 10, "White").print
}
```

## Cats 써보기

- Cats에 있는 타입 클래스와 인스턴스, 인터페이스 메서드를 사용해보자. 먼저 Cats에 있는 `cats.Show`를
  살펴보자. `cats.Show`는 위 예제에서 만든 `Printable` 타입 클래스와 같은 것이다.
  ```scala
  package cats

  trait Show[A] {
    def show(value: A): String
  }
  ```

### 타입 클래스 import 하기

- `Cats` 타입 클래스는 `cats` 패키지에 있다. `Show`는 아래와 같이 import 할 수 있다.
  ```scala
  import cats.Show
  ```

- `Cats` 타입 클래스의 동반 객체에는 여러 타입에 대한 인스턴스가 정의된 apply 메서드를 가지고 있다.
  ```scala
  val showInt = Show.apply[Int]
  // <console>:13: error: could not find implicit value for parameter
  // instance: cats.Show[Int]
  // val showInt = Show.apply[Int]
  ```

- 에러가 나는 이유는 implicit 값을 찾을 수 없기 때문이다. 이 스코프에 적절한 인스턴스를 사용할 수 있도록
  해야한다.

### 기본 인스턴스 import 하기

- `cats.instances` 패키지에는 다양한 타입에 대한 기본 인스턴스가 있다. 아래 패키지에는 `Cats` 타입
  클래스의 특정 파라미터에 대한 모든 인스턴스가 정의되어 있다.

  - `cats.instances.int`는 Int 타입의 인스턴스
  - `cats.instances.string`은 String 타입의 인스턴스
  - `cats.instances.list`에는 List 타입의 인스턴스
  - `cats.instances.option`에는 Option 타입의 인스턴스
  - `cats.instances.all`에는 모든 타입의 인스턴스

- 인스턴스 전체 목록을 보려면 `cats.instances` 문서를 봐라.
- String과 Int의 `Show` 인스턴스를 import 하고 써보자.
  ```scala
  import cats.instances.int._    // for Show
  import cats.instances.string._ // for Show

  val showInt: Show[Int] = Show.apply[Int]
  val showString: Show[String] = Show.apply[String]

  val intAsString: String =
    showInt.show(123)
  // intAsString: String = 123
  val stringAsString: String =
    showString.show("abc")
  // stringAsString: String = abc
  ```

### 인터페이스 문법(Syntax) import 하기

- `cats.syntax.show`에 있는 인터페이스 문법을 import 하면 더 쉽게 쓸 수 있다.
  ```scala
  import cats.syntax.show._ // for show

  val shownInt = 123.show
  // shownInt: String = 123
  val shownString = "abc".show
  // shownString: String = abc
  ```

### 다 import 하기

- `import cats._` 하면 Cats의 모든 타입 클래스를 import 한다.
- `import cats.instances.all._` 하면 Cats의 모든 타입 클래스 인스턴스를 import 한다.
- `import cats.syntax.all._` 하면 Cats의 모든 인터페이스 문법을 import 한다.
- `import cats.implicits._` 하면 모든 타입 클래스 인스턴스와 인터페이스 문법을 import 한다.
- 그래서 보통 아래 처럼 하면 Cats의 모든 기능을 쓸 수 있다. (충돌이 나는 경우는 뒤에서 살펴본다.)
  ```scala
  import cats._
  import cats.implicits._
  ```

### 커스텀 인스턴스 정의하기

- 어떤 타입에 대한 Show 인스턴스는 다음과 같이 만들 수 있다.
  ```scala
  import java.util.Date

  implicit val dateShow: Show[Date] =
    new Show[Date] {
      def show(date: Date): String =
        s"${date.getTime}ms since the epoch."
    }
  ```

- 하지만 Cats에서 좀 더 편하게 만들 수 있는 메서드를 Show 동반 객체에서 제공한다.
  ```scala
  object Show {
    // Convert a function to a `Show` instance:
    def show[A](f: A => String): Show[A] =
      ???
    // Create a `Show` instance from a `toString` method:
    def fromToString[A]: Show[A] =
      ???
  }
  ```

- 위 메서드를 이용해 `Date` 클래스의 Show 타입 인스턴스를 만들어보자.
  ```scala
  implicit val dateShow: Show[Date] =
    Show.show(date => s"${date.getTime}ms since the epoch.")
  ```

### 연습 문제

```scala
import cats._
import cats.implicits._

implicit val catShow: Show[Cat] =
  Show.show(value => s"${value.name} is a ${value.age} year-old ${value.color} cat.")

"Test".show
123.show
Cat("Jerry", 10, "White").show
```

## Eq 예제

- `cats.Eq`는 스칼라 빌트인 연산자인 `==`로 타입 안정적인 동등성 비교와 주소값에 신경쓰지 않아도 되는 비교를
  할 수 있게 해준다.
  ```scala
  List(1, 2, 3).map(Option(_)).filter(item => item == 1)
  // res0: List[Option[Int]] = List()
  ```

- 위 예제에서 `filter`는 item 값이 Option[Int] 타입이게 때문에 항상 false가 나온다. 1이 아니고
  Some(1)과 비교를 해야한다.
- 타입 에러가 나지 않는 이유는 `==`는 어떤 객체도 비교할 수 있기 때문이다.
- Eq는 이런 문제를 해결하기 위해 타입 안정적인 비교를 한다.

### Equality, Liberty, and Fraternity

- Eq는 주어진 타입을 가지고 타입 안정적 비교를 한다.
  ```scala
  package cats

  trait Eq[A] {
    def eqv(a: A, b: A): Boolean
    // other concrete methods based on eqv...
  }
  ```

- `cats.syntax.eq`에 다음과 같은 인터페이스 문법이 정의 정의 되어 있다.
  - `===` 두 객체의 동등 비교
  - `=!=` 두 객체의 다름 비교

### Int 비교하기

- 예제를 해보기 위해서 타입 클래스를 import 한다.
  ```scala
  import cats.Eq
  ```

- Int 타입에 대한 Eq 타입 인스턴르를 만든다.
  ```scala
  import cats.instances.int._ // for Eq

  val eqInt = Eq[Int]
  ```

- 이제 `eqInt` 인스턴스를 써서 `eqv`로 비교할 수 있다.
  ```scala
  eqInt.eqv(123, 123)
  // res2: Boolean = true

  eqInt.eqv(123, 234)
  // res3: Boolean = false
  ```

- 스칼라 기본 동등 연산자(`==`)와 다르게 타입이 다르면 컴파일 에러가 난다.
  ```scala
  eqInt.eqv(123, "234")
  // <console>:18: error: type mismatch; // found : String("234")
  // required: Int
  // eqInt.eqv(123, "234")
  //
  ```

- 인터페이스 문법을 사용하려면 `cats.syntax.eq`를 import 하면 된다.
  ```scala
  import cats.syntax.eq._ // for === and =!=

  123 === 123
  // res5: Boolean = true
  123 =!= 234
  // res6: Boolean = true
  ```

- 역시 다른 타입은 에러가 난다.
  ```scala
  123 === "123"
  // <console>:20: error: type mismatch;
  //  found   : String("123")
  //  required: Int
  //        123 === "123"
  //                ^
  ```

### Option 비교하기

- Option[Int]를 비교해보기 위해 다음 패키지를 import 하자.
  ```scala
  import cats.instances.int._    // for Eq
  import cats.instances.option._ // for Eq
  ```

- 타입이 모호하기 때문에 잘 안된다.
  ```scala
  Some(1) === None
  // <console>:26: error: value === is not a member of Some[Int] // Some(1) === None
  // ^
  ```

- 타입을 지정하고 다시해보자.
  ```scala
  (Some(1) : Option[Int]) === (None : Option[Int])
  // res9: Boolean = false
  ```

- 좀 더 편하게 쓰기 위해 Option.apply와 Option.empty 메서드를 이용해보자.
  ```scala
  Option(1) === Option.empty[Int]
  // res10: Boolean = false
  ```

- 좀 특별한 문법을 써서 더 간단히 비교할 수 있다.
  ```scala
  import cats.syntax.option._ // for some and none

  1.some === none[Int]
  // res11: Boolean = false

  1.some =!= none[Int]
  // res12: Boolean = true
  ```

### 커스텀 타입 비교하기

- Eq 타입 클래스의 인스턴스를 만들어서 커스텀 타입 비교를 할 수 있다.
  ```scala
  import java.util.Date
  import cats.instances.long._ // for Eq

  implicit val dateEq: Eq[Date] =
    Eq.instance[Date] { (date1, date2) =>
      date1.getTime === date2.getTime
    }

  val x = new Date() // now
  val y = new Date() // a bit later than now

  x === x
  // res13: Boolean = true
  x === y
  // res14: Boolean = false
  ```

### 연습문제

```scala
import cats._
import cats.implicits._

implicit val catEq: Eq[Cat] =
  Eq.instance((a, b) => a.name == b.name && a.age == b.age && a.color == b.color)

val cat1 = Cat("Garfield",   38, "orange and black")
val cat2 = Cat("Heathcliff", 33, "orange and black")

val optionCat1 = Option(cat1)
val optionCat2 = Option.empty[Cat]

cat1 === cat1
cat2 === cat2
cat1 === cat2
cat1 =!= cat1
optionCat1 === optionCat2
```  

## 인스턴스 선택을 조정하기

- 타입과 그 서브 타입이 있는 경우 인스턴스 선택은 어떻게 될까?

### Variance

- 스칼라에서 타입 변수에 대한 하위 타입 설정은 다음과 같다.
  ```scala
  trait F[+A] // the "+" means "covariant"
  ```

#### Covariance

- B가 A의 하위 타입이라면 F[B]는 F[A]를 쓰는 곳에 쓸 수 있다.
- 다음 예를 보자.
  ```scala
  trait List[+A]
  trait Option[+A]

  sealed trait Shape
  case class Circle(radius: Double) extends Shape
  val circles: List[Circle] = ???
  val shapes: List[Shape] = circles
  ```
- Circle은 Shape의 하위 타입이기 때문에 List[Circle]는 List[Shape]에 쓸 수 있다.


#### Contravariance
