# 모노이드와 세미그룹

- 이번 장에서는 모노이드와 세미그룹에 대해 알아보자. 이건 값을 추가하거나 합칠 수 있게 해준다.
- Ints, Strings, Lists, Options 등 여러 타입에 인스턴스가 있다.
- 간단한 예제로 뭔지 알아보자.

### 숫자 더하기

- Int를 더하는 것은 닫힌 이항 연산이다. 무슨 말이냐면 두 Int는 다른 Int를 결과로 낸다는 말이다.
  ```scala
  2 + 1
  // res0: Int = 3
  ```

- 0 이라는 iden􏰀tity 엘리먼트가 있는데 어떤 Int 값에 대해 `a + 0 == 0 + a == a`라는 성질을 가진다.
  ```scala
  2 + 0
  // res1: Int = 2

  0 + 2
  // res2: Int = 2
  ```
- 더하기에는 또 다른 성질이 있는데 더하는 순서와 상관 없이 같은 값을 가진다. 이것은 associa􏰃vity 라고 한다.
  ```scala
  (1 + 2) + 3
  // res3: Int = 6

  1 + (2 + 3)
  // res4: Int = 6
  ```

### 숫자 곱하기

- 곱하기에도 더하는 성질이 똑같이 적용된다. iden􏰀tity를 0 대신 1을 쓴다.
  ```scala
  1 * 3
  // res5: Int = 3

  3 * 1
  // res6: Int = 3
  ```

- 곱하기도 더하기 처럼 associa􏰃vity가 적용된다.
  ```scala
  (1 * 2) * 3
  // res7: Int = 6

  1 * (2 * 3)
  // res8: Int = 6
  ```

### 문자열과 시퀀스 합치기

- 문자열 이항 연산으로 문자열을 합칠 수 있다.
  ```scala
  "One" ++ "two"
  // res9: String = Onetwo
  ```

- 문자열에서 identity는 빈 문자열이다.
  ```scala
  "" ++ "Hello"
  // res10: String = Hello

  "Hello" ++ ""
  // res11: String = Hello
  ```

- associative도 적용된다.
  ```scala
  ("One" ++ "Two") ++ "Three"
  // res12: String = OneTwoThree

  "One" ++ ("Two" ++ "Three")
  // res13: String = OneTwoThree
  ```

- 시퀀스를 합치기 위해 일반적인 `+` 연산자 대신 `++`를 썼다. 빈 시퀀스와 이항 연산을 사용하면 다른 시퀀스도
  합칠 수 있다.

## 모노이드 정의

- 위에서 몇가지 합치는 것에 대한 시나리오를 봤는데 이게 모노이드다. 타입 A에 대한 모노이드는 다음과 같다.
  - 합치는 연산자인 (A, A) => A
  - 타입 A 에 대한 빈 항목

- 위 정의를 스칼라 코드로 옮기면 다음과 같다. Cats에 간단히 정의 되어 있는 버전이다.
  ```scala
  trait Monoid[A] {
    def combine(x: A, y: A): A
    def empty: A
  }
  ```

- combine과 empty를 제공하려면 모노이드에 몇가지 규칙을 따라야한다. 타입 A에 x,y,z 에 대해 associative
  성질이 유지 되어 야 하고 empty는 indentity 항목이어야 한다.
  ```scala
  def associativeLaw[A](x: A, y: A, z: A)
        (implicit m: Monoid[A]): Boolean = {
    m.combine(x, m.combine(y, z)) ==
      m.combine(m.combine(x, y), z)
  }
  def identityLaw[A](x: A)
        (implicit m: Monoid[A]): Boolean = {
    (m.combine(x, m.empty) == x) &&
      (m.combine(m.empty, x) == x)
  }
  ```

- 예를 들어 빼기 연산은 모노이드가 아니다. 빼기는 associative 가 적용되지 않기 때문이다.
  ```scala
  (1 - 2) - 3
  // res15: Int = -4

  1 - (2 - 3)
  // res16: Int = 2
  ```

- 모노이드를 만들때는 항상 규칙에 맞는지 신경 써야한다. 만약 규칙에 맞지 않는 모노이드를 만들면 Cats를
  사용할 때 어떤 결과가 나올지 예측할 수 없다. 대부분은 Cats에서 제공하는 것을 쓸거고 Cats 라이브러리는
  규칙을 지켰다고 가정한다.

## 세미그룹 정의하기

- 세미그룹은 모노이드에서 combine 만 있는 것이다. 많은 세미그룹이 모노이드다. 어떤 데이터 타입은 empty를
  정의할 수 없다. 에를 들어 앞에서 본 시퀀스 합치기와 숫자 더하기 모노이에서 비어 있지 않은 시퀀스나 0 보다
  큰 수만 합칠 수 있다는 제약을 한다면 empty를 정의할 필요가 없다. Cats에 NonEmptyList는 세미그룹은
  구현되어 있지만 모노이드는 구현되어 있지 않다. 좀 더 정확한 Cats 모노이드의 정의는 다음과 같다.
  ```scala
  trait Semigroup[A] {
    def combine(x: A, y: A): A
  }
  trait Monoid[A] extends Semigroup[A] {
    def empty: A
  }
  ```

- 위와 같은 상속 형태는 앞으로 타입 클래스에 대해 논의 할 때 많이 볼거다. 이건 동작을 재사용할 수 있고
  모듈화가 가능하다. 어떤 타입 A에 대해 모노이드를 정의하면 세미그룹은 자동으로 얻을 수 있고 Semigroup[B]
  인자가 필요한 곳에 Monoid[B]를 사용할 수 있다.

## 연습문제: 모노이드에 대한 진리

```scala
object MonoidInstances {
  implicit val booleanMonoid = new Monoid[Boolean] {
    override def empty: Boolean = true

    override def combine(x: Boolean, y: Boolean): Boolean = x && y
  }
}
```

## 연습문제: Set에 대한 모노이드


## Cat에 모노이드

- Cats에 있는 모노이드를 살펴보자.

### 모노이드 타입 클래스

- 모노이드 타입 클래스는 `cats.kernel.Monoid`에 있고 `cats.Monoid`로 앨리어스가 되어 있다.
  모노이드는 `cats.kernel.Semigroup`를 extends하고 `cats.Semigroup`으로 앨리어스 되어 있다.
  쓸 때는 그냥 아래 처럼 import 해서 쓴다.
  ```scala
  import cats.Monoid
  import cats.Semigroup
  ```

- `cats.kernel`은 Cats 서브 프로젝트로 작은 라이브러리다. Cats에 필수가 아닌 부분들이 들어있고
  코어 타입 클래스들이다. cats 패키지로 앨리어스 되어 있고 이 책에서는 Eq, Semigroup, Monoid를
  다루고 나머지 타입 클래스는 그냥 Cats 프로젝트다.

### 모노이드 인스턴스

- 모노이드 인터페이스는 Cats의 일반적인 패턴을 따르고 있어서 동반 객체에 apply가 특정 타입 클래스 인스턴스를
  리턴한다. 예를 들어 String의 타입 클래스 인스턴스는 아래와 같다.
  ```scala
  import cats.Monoid
  import cats.instances.string._ // for Monoid

  Monoid[String].combine("Hi ", "there")
  // res0: String = Hi there

  Monoid[String].empty
  // res1: String = ""
  ```

- 아래 처럼해도 똑같다.
  ```scala
  Monoid.apply[String].combine("Hi ", "there")
  // res2: String = Hi there

  Monoid.apply[String].empty
  // res3: String = ""
  ```

- 위에서 설명한 것 처럼 Monoid는 Semigroup을 extends 하기 때문에 만약 empty가 필요 없다면 아래는
  위와 같은 코드다.
  ```scala
  import cats.Semigroup

  Semigroup[String].combine("Hi ", "there")
  // res4: String = Hi there
  ```

- 타입 클래스 인스턴스는 `cats.instances` 1장에서 설명한 것 처럼 패키지 아래 잘 정리되어 있다. 만약
  Int 타입에 대한 타입 클래스 인스턴스가 필요하면 `cats.instances.int`를 import 하면 된다.
  ```scala
  import cats.Monoid
  import cats.instances.int._ // for Monoid

  Monoid[Int].combine(32, 10)
  // res5: Int = 42
  ```

- `cats.instances.int`와 `cats.instances.option` 인스턴스로 Monoid[Option[Int]]도
  조합 할 수 있다.
  ```scala
  import cats.Monoid
  import cats.instances.int._    // for Monoid
  import cats.instances.option._ // for Monoid

  val a = Option(22)
  // a: Option[Int] = Some(22)

  val b = Option(20)
  // b: Option[Int] = Some(20)

  Monoid[Option[Int]].combine(a, b)
  // res6: Option[Int] = Some(42)
  ```

### 모노이드 문법

- Cats는 combine 메서드에 대해 `|+|` 연산자를 제공한다. combine은 Semigroup의 성질이기 때문에
  `cats.syntax.semigroup`을 import 해서 쓸 수 있다.
  ```scala
  import cats.instances.string._ // for Monoid
  import cats.syntax.semigroup._ // for |+|

  val stringResult = "Hi " |+| "there" |+| Monoid[String].empty
  // stringResult: String = Hi there

  import cats.instances.int._ // for Monoid

  val intResult = 1 |+| 2 |+| Monoid[Int].empty
  ```

### 연습문제

```scala
import cats._
import cats.implicits._

def add[T: Monoid](items: List[T]): T =
 items.foldLeft(Monoid[T].empty)(_ |+| _)
```

## 모노이드 애플리케이션

- 모노이드는 어디에 쓰일까?

### 빅 데이터

- ... 케이스 스터디 부분에서 자세히 다룸

### 분산 시스템

- CRDT 이것도 뒤에 케이스 스터디에 다룬다.

### 작은 곳에서 모노이드

- 위 두 개는 아키텍처 관점에서 모노이드인데 작은 코드에서도 쓰인다. 이 책에 케이스 스터디 부분에서 다룬다.

## 요약

```scala
import cats.Monoid
import cats.instances.string._ // for Monoid
import cats.syntax.semigroup._ // for |+|

"Scala" |+| " with " |+| "Cats"
// res0: String = Scala with Cats
```

```scala
import cats.instances.int._    // for Monoid
import cats.instances.option._ // for Monoid

Option(1) |+| Option(2)
// res1: Option[Int] = Some(3)

import cats.instances.map._ // for Monoid

val map1 = Map("a" -> 1, "b" -> 2)
val map2 = Map("b" -> 3, "d" -> 4)

map1 |+| map2
// res3: Map[String,Int] = Map(b -> 5, d -> 4, a -> 1) import cats.instances.tuple._ // for Monoid

val tuple1 = ("hello", 123)
val tuple2 = ("world", 321)

tuple1 |+| tuple2
// res6: (String, Int) = (helloworld,444)
```

```scala
def addAll[A](values: List[A])
      (implicit monoid: Monoid[A]): A =
  values.foldRight(monoid.empty)(_ |+| _)

addAll(List(1, 2, 3))
// res7: Int = 6

addAll(List(None, Some(1), Some(2)))
// res8: Option[Int] = Some(3)
```
