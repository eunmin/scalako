# 모나드

- 모나드는 스칼라에서 가장 일반적인 추상이다. 많은 스칼라 개발자가 모나드에 빨리 익숙해지려고 한다.
- 비공식적으로 어떤 생성자와 flatMap 메서드가 있으면 모나드다. Option, List, Future를 포함해
  지난 장에서 본 모든 functor도 모나드다.
- 이미 모나드를 위한 특별한 문법도 있다: for 구문
- 이 개념이 어디에나 쓸 수 있지만 스칼라 기본 라이브러리에는 flatMap 할 수 있는 것에 대한 구체적인 타입이
  없다. 그래서 Cats에 있는 Monad 타입 클래스는 유용하다.
- 이 장에서 모나드에 대해 깊이 알아본다. 동기에 대해 몇 몇 예제로 알아보자. 그러면서 Cats에 구현된 공식적인
  정의에 대해 다가가보자. 마지막으로 본적 없는 재미난 몇 몇 모나드에 대해 소개와 예제로 살펴보자.

## 모나드란?

- 이 질문에 대해 쓴 글은 많이 있다.
- 일단 이 질문에 간단히 답해보자
  - 모나드는 연속된 계산을 위한 방법이다.
- 쉽다. 문제가 해결되었다. 지난 장에서 Functor는 정확히 같은 것을 컨트롤하는 방법이라고 했다. 좀 더 논의 해보자.
- 이전 장에서 Functor는 좀 복잡한 것을 무시하고 연속된 계산을 할 수 있도록 해준다고 했다.
- 그러나 Functor는 그 복잡한 것이 계산의 시작에서 딱 한번 만 허용된다는 제약이 있다. 계산 각 단계 마다
  복잡한 것이 있으면 처리할 수 없다.
- 이것이 모나드가 필요한 이유다. 모나드에 flatMap 메서드는 계산 과정 중 어떤 문제를 고려해서 단계를 진행할지
  지정할 수 있다.
- Option의 flatMap은 중간에 Option을 고려한다. List에 flatMap은 List의 중간을 다룬다. 등등..
- 각각의 경우에 따라 flatMap에 전달되는 함수는 애플리케이션 부분을 구성하고 flatMap 자체는 다음 flatMap을
  실행할지 결정한다.
- 예제를 보자.

### Option

```scala
def parseInt(str: String): Option[Int] =
  scala.util.Try(str.toInt).toOption

def divide(a: Int, b: Int): Option[Int] =
  if(b == 0) None else Some(a / b)
```

- 각 함수는 None을 리턴해 실패할 수 있다. flatMap 메서드는 연속된 계산을 할 때 이것을 무시할 수 있게
  해준다.  
  ```scala
  def stringDivideBy(aStr: String, bStr: String): Option[Int] =
    parseInt(aStr).flatMap { aNum =>
      parseInt(bStr).flatMap { bNum =>
        divide(aNum, bNum)
      }
    }
  ```
- 이 의미를 잘 알고 있다.
  - 첫번째 parseInt 호출은 None이거나 Some이다.
  - 만약 Some이면 flatMap은 aNum에 값을 넘겨준다.
  - 두번째 parseInt 호출은 None이거나 Some이다.
  - 만약 Some이면 flatMap은 bNum에 값을 넘겨준다.
  - divide 호출은 None이거나 Some이다. 이것이 전체 결과다.
- flatMap은 함수를 호출할 지 선택하고 함수는 연속된 계산의 다음 계산을 만들어낸다.
- 계산의 결과는 Option이기 때문에 다음 계산 단계에서 다시 flatMap을 부를 수 있다.
- 이 결과는 잘 알고 있는 fail-fast 에러 핸들링 동작을 하고 어느 단계에서 발생한 None은 전체적으로
  None으로 볼 수 있다.
  ```scala
  stringDivideBy("6", "2")
  // res1: Option[Int] = Some(3)

  stringDivideBy("6", "0")
  // res2: Option[Int] = None

  stringDivideBy("6", "foo")
  // res3: Option[Int] = None

  stringDivideBy("bar", "2")
  // res4: Option[Int] = None
  ```
- 모든 모나드는 Functor다(아래서 증명 예정). 그래서 새로운 모나드를 정의하던지 아니던지 연속된 게산에서
  map이나 flatMap을 쓸 수 있다. 그리고 동작을 명확하게 하기 위해서 map이나 flatMap을 for 구문에서
  사용할 수 있다.
  ```scala
  def stringDivideBy(aStr: String, bStr: String): Option[Int] = for {
      aNum <- parseInt(aStr)
      bNum <- parseInt(bStr)
      ans  <- divide(aNum, bNum)
  } yield ans
  ```

### List

- 다음은 `flatMap`을 List에 적용해보자. for 문법을 쓰면 절차형 프로그래밍 언어의 반복문과 많이 비슷
  하다.
  ```scala
  for {
    x <- (1 to 3).toList
    y <- (4 to 5).toList
  } yield (x, y)
  // res5: List[(Int, Int)] = List((1,4), (1,5), (2,4), (2,5), (3,4), (3,5))
  ```
- 리스트를 중간 결과에 Set으로 보면 `flatMap`은 순서를 바꾸고 조합하는 생성자가 된다.
- 위 예제에서 `x`는 가능한 값이 3개 있고 `y`는 가능한 값이 2개 있다.
- 이것은 6개의 `(x, y)` 가능한 값이 있다. `flatMap`이 코드에서 이런 코드를 생성한다.
  - `x` 가져오기
  - `y` 가져오기
  - `(x, y)` 튜플 만들기

### Future

- Future는 비동기를 신경쓰지 않고 연속적인 계산을 할 수 있게 해주는 모나드다.
  ```scala
  import scala.concurrent.Future
  import scala.concurrent.ExecutionContext.Implicits.global
  import scala.concurrent.duration._

  def doSomethingLongRunning: Future[Int] = ???
  def doSomethingElseLongRunning: Future[Int] = ???

  def doSomethingVeryLongRunning: Future[Int] =
    for {
      result1 <- doSomethingLongRunning
      result2 <- doSomethingElseLongRunning
    } yield result1 + result2
  ```
- 코드의 각 단계가 실행될 때, `flatMap`은 복잡한 스래드풀, 스케쥴일에 대한 처리를 담당한다.
- Future를 많이 써봤다면 위에 코드가 순서대로 실행된다는 것을 알 수 있다.
- for 구문을 쓴다면 중첩된 `flatMap` 호출을 더 명확하게 만들 수 있다.
  ```scala
  def doSomethingVeryLongRunning: Future[Int] =
    doSomethingLongRunning.flatMap { result1 =>
      doSomethingElseLongRunning.map { result2 =>
        result1 + result2
      }
    }
  ```
- 연속된 계산 속에 있는 각각의 Future는 이전 Future의 결과를 받는 함수를 생성한다. 다시 말해 계산의
  각 단계는 이전 작업이 끝나야만 시작할 수 있다.
- 물론 Future를 동시에 실행할 수 도 있지만 모나드는 연속된 계산에 대한 내용이기 때문에 그건 딴 얘기다.

## 모나드 정의

- 위에서 `flatMap`에 대해서만 이야기 했지만 원래 모나드는 연산이 두개다.
  - `A => F[A]` 타입의 `pure`
  - `(F[A], A => F[B]) => F[B]` 타입의 `flatMap`
- `pure`는 생성자를 추상화 한다. 일반 값을 받아서 모나드 값을 만든다.
- `flatMap`은 위에서 이야기 한 것 처럼 컨택스트에서 값을 꺼내서 다음 계산의 컨택스트를 만든다.
- 아래는 간단한 버전의 Cats 모나드 타입 클래스다.
  ```scala
  import scala.language.higherKinds

  trait Monad[F[_]] {
    def pure[A](value: A): F[A]

    def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]
  }
  ```
### 모나드 법칙
- `pure`와 `flatMap`은 의도하지 않은 결과가 생기지 않도록 몇 가지 규칙을 따라야한다.
- 왼쪽 항등원 법칙: `pure` 결과에 `func`로 변환을 하면 `func`를 그냥 쓴것과 같아야 한다:
  ```scala
  pure(a).flatMap(func) == func(a)
  ```
- 오른쪽 항등원 법칙: `flatMap`에 `pure`를 넘기면 아무것도 하지 않은 것과 같아야한다:
  ```scala
  m.flatMap(pure) == m
  ```
- 결합 법칙: `f`와 `g`를 `flatMap`한 것은 `f`와 `g`를 `flatMap`한 것과 같아야한다:
  ```scala
  m.flatMap(f).flatMap(g) == m.flatMap(x => f(x).flatMap(g))
  ```

## 연습문제

- 모든 모나드는 functor기 때문에 모나드로 map을 구현해봐라.

```scala
import scala.language.higherKinds

trait Monad[F[_]] {
  def pure[A](a: A): F[A]

  def flatMap[A, B](value: F[A])(func: A => F[B]): F[B]

  def map[A, B](value: F[A])(func: A => B): F[B] =
    flatMap(value)(x => pure(func(x)))
}
```

## Cats 모나드

- 이제 Cats에 있는 모나드를 살펴보자. 역시 타입 클래스, 인스턴스, 구문을 살펴볼 거다.

### 모나드 타입 클래스

- 모나드 타입 클래스는 `cats.Monad`다. `Monad`는 두개 타입 클래스를 확장한다: `flatMap` 메서드를
  쓸 수 있는 `FlatMap`과 `pure`를 쓸 수 있는 `Applicative`다. `Applicative`는 `Functor`를
  확장하기 때문에 모든 `Monad`는 위에서 살펴본 `map` 메서드가 있다. 6장에서 `Applicative`에 대해
  다룰 거다.
- 아래는 `pure`와 `flatMap`과 `map`을 직접 쓰는 예제다.
  ```scala
  import cats.Monad
  import cats.instances.option._ // for Monad
  import cats.instances.list._   // for Monad

  val opt1 = Monad[Option].pure(3)
  // opt1: Option[Int] = Some(3)

  val opt2 = Monad[Option].flatMap(opt1)(a => Some(a + 2))
  // opt2: Option[Int] = Some(5)

  val opt3 = Monad[Option].map(opt2)(a => 100 * a)
  // opt3: Option[Int] = Some(500)

  val list1 = Monad[List].pure(3)
  // list1: List[Int] = List(3)

  val list2 = Monad[List].
    flatMap(List(1, 2, 3))(a => List(a, a*10))
  // list2: List[Int] = List(1, 10, 2, 20, 3, 30)

  val list3 = Monad[List].map(list2)(a => a + 123)
  // list3: List[Int] = List(124, 133, 125, 143, 126, 153)
  ```
- `Monad`에는 `Functor`에 있는 메서드를 포함해 다양한 메서드가 있다. 자세한 내용은 문서를 참고하라.

### 기본 인스턴스

- Cats는 `cats.instances`에서 기본 라이브러리(Option, List, Vector 등등)에 있는 타입에 대한
  모나드 인스턴스를 제공한다.
  ```scala
  import cats.instances.option._ // for Monad

  Monad[Option].flatMap(Option(1))(a => Option(a*2))
  // res0: Option[Int] = Some(2)

  import cats.instances.list._ // for Monad

  Monad[List].flatMap(List(1, 2, 3))(a => List(a, a*10))
  // res1: List[Int] = List(1, 10, 2, 20, 3, 30)

  import cats.instances.vector._ // for Monad

  Monad[Vector].flatMap(Vector(1, 2, 3))(a => Vector(a, a*10))
  // res2: Vector[Int] = Vector(1, 10, 2, 20, 3, 30)
  ```
- Cats는 `Future`에 대한 모나드도 제공한다. `Future`에 원래 있는 `pure`와 `flatMap` 메서드와
  다르게 Cats에 있는 메서드는 `ExecutionContext`를 받지 않는다. (`Monad` 트레잇에 정의되지 않은
  파라미터기 때문에). 이문제를 해결하기 위해 Cats는 `Future` 모나드를 부를 때 `ExecutionContext`를
  요구한다.
  ```scala
  import cats.instances.future._ // for Monad
  import scala.concurrent._
  import scala.concurrent.duration._

  val fm = Monad[Future]
  // <console>:37: error: could not find implicit value for parameter instance: cats.Monad[scala.concurrent.Future]
  //        val fm = Monad[Future]
  //                 
  ```
- `ExecutionContext`를 쓰기 위해 implicit 해석하는 부분을 수정한다.
  ```scala
  import scala.concurrent.ExecutionContext.Implicits.global

  val fm = Monad[Future]
  // fm: cats.Monad[scala.concurrent.Future] = cats.instances.FutureInstances$$anon$1@71738c4a
  ```
- `pure`를 `flatMap` 쓸 때 모나드 인스턴스는 `ExecutionContext`를 쓴다.
  ```scala
  val future = fm.flatMap(fm.pure(1))(x => fm.pure(x + 2))

  Await.result(future, 1.second)
  // res3: Int = 3
  ```
- Cats에는 기본 라이브러리에 없는 새로운 모나드도 제공한다. 이것에 익숙해질 거다.

### 모나드 구문

- 모나드 구문은 세 곳에 있다.
  - `flatMap` 구문은 `cats.syntax.flatMap`
  - `map` 구문은 `cats.syntax.functor`
  - `pure` 구문은 `cats.syntax.applicative`
- 쓸 때는 `cats.implicits`로 한번에 불러오는 것이 좋지만 여기서는 명확하게 하기 위해 각각 import
  하겠다.
- 모나드 값을 만들 때 `pure`를 쓸 수 있다. `pure`를 쓸 때는 모나드 값을 명확하게 하기 위해 타입 인자가
  필요할 수 있다.
  ```scala
  import cats.instances.option._   // for Monad
  import cats.instances.list._     // for Monad
  import cats.syntax.applicative._ // for pure

  1.pure[Option]
  // res4: Option[Int] = Some(1)

  1.pure[List]
  // res5: List[Int] = List(1)
  ```
- Option이나 List 같은 스칼라 기본 라이브러리에 대한 `flatMap`과 `map`을 직접 쓰는 것은 힘들다.
  왜냐하면, 원래 기본적으로 `flatMap`과 `map` 메서드가 있기 때문이다. 대신 사용자가 선택한 모나드로
  감싼 파라미터로 받아서 실행하는 일반 함수를 만들거다.
  ```scala
  import cats.Monad
  import cats.syntax.functor._ // for map
  import cats.syntax.flatMap._ // for flatMap
  import scala.language.higherKinds

  def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
    a.flatMap(x => b.map(y => x*x + y*y))

  import cats.instances.option._ // for Monad
  import cats.instances.list._   // for Monad

  sumSquare(Option(3), Option(4))
  // res8: Option[Int] = Some(25)

  sumSquare(List(1, 2, 3), List(4, 5))
  // res9: List[Int] = List(17, 26, 20, 29, 25, 34)
  ```
- for 구문을 써서 만들 수 있다. 컴파일러는 구문에 `flatMap`과 `map`와 알맞는 모나드의 implict를
  넣어서 잘 동작할 수 있게 만들거다.
  ```scala
  def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
    for {
      x <- a
      y <- b
    } yield x*x + y*y

  sumSquare(Option(3), Option(4))
  // res10: Option[Int] = Some(25)

  sumSquare(List(1, 2, 3), List(4, 5))
  // res11: List[Int] = List(17, 26, 20, 29, 25, 34)
  ```
- 이것이 Cats 모나드에 대해 알아야할 모든 것이다. 다음은 스칼라 라이브러리에 없는 모나드를 살펴보자.

## Identity 모나드

- 앞에서 `flatMap`과 `map` 데모를 위해 여러 모나드가 쓸 수 있는 메서드를 하나 만들었다.
  ```scala
  import scala.language.higherKinds
  import cats.Monad
  import cats.syntax.functor._ // for map
  import cats.syntax.flatMap._ // for flatMap

  def sumSquare[F[_]: Monad](a: F[Int], b: F[Int]): F[Int] =
    for {
      x <- a
      y <- b
    } yield x*x + y*y
  ```
- 이건 Option이나 List에 대해 잘 동작한다. 하지만 그냥 값을 전달하려면 어떻게 해야할까?
  ```scala
  sumSquare(3, 4)
  // <console>:22: error: no type parameters for method sumSquare: (a: F[Int],
  // b: F[Int])(implicit evidence$1: cats.Monad[F])F[Int] exist so that it can
  // be applied to arguments (Int, Int)
  ```
- 만약 `sumSquare` 파라미터에 모나드 또는 일반 값을 함께 쓸 수 있다면 정말 유용할 거다. 그렇다면
  모나드 값과 일반 값을 함께 추상화 할 수 있다. 다행이 Cats에 이 간극을 매워줄 `Id` 타입이 있다.
  ```scala
  import cats.Id

  sumSquare(3 : Id[Int], 4 : Id[Int])
  // res2: cats.Id[Int] = 25
  ```
- `Id`로 모나드 메서드에 일반 값을 전달 할 수 있다. 하지만 정확한 의미는 이해하기 어렵다.
- `sumSquare` 메서드에 `Id[Int]`로 변환 해서 넘기고 `Id[Int]`를 받았다.
- 무슨일인가? 아래는 `Id`에 대한 정의다:
  ```scala
  package cats

  type Id[A] = A
  ```
- `Id`는 실제로 단일 타입을 받아 그 타입을 리턴하는 타입 앨리어스다. 그래서 어떤 타입의 값도 `Id`에
  해당하는 타입으로 바꿀 수 있다.
  ```scala
  "Dave" : Id[String]
  // res3: cats.Id[String] = Dave

  123 : Id[Int]
  // res4: cats.Id[Int] = 123

  List(1, 2, 3) : Id[List[Int]]
  // res5: cats.Id[List[Int]] = List(1, 2, 3)
  ```
- Cats는 `Id` 타입에 대해 `Functor`나 `Monad`를 포함한 여러 타입 클래스의 인스턴스를 제공한다.
- 이걸로 `map`, `flatMap`, `pure` 같은 값을 전달 할 수 있도록 해준다.
  ```scala
  val a = Monad[Id].pure(3)
  // a: cats.Id[Int] = 3

  val b = Monad[Id].flatMap(a)(_ + 1)
  // b: cats.Id[Int] = 4

  import cats.syntax.functor._ // for map
  import cats.syntax.flatMap._ // for flatMap

  for {
    x <- a
    y <- b
  } yield x + y
  // res6: cats.Id[Int] = 7
  ```
- 모나드와 일반 값을 함께 추상화하는 것은 매우 강력하다. 예를들어 실제 환경에서는 Future를 이용해 비동기로
  돌리고 테스트 환경에서는 Id로 동기화해서 돌릴 수 도 있다. 이건 8장에서 살펴볼 예정이다.

### 연습문제

- `Id`에 대한 `pure`, `map`, `flatMap`을 구현하라. 구현하다가 발견한 흥미로운 점은 무었인가?
  ```scala
  def pure[A](value: A): Id[A] =
    value

  def map[A, B](initial: Id[A])(func: A => B): Id[B] =
    func(initial)

  def flatMap[A, B](initial: Id[A])(func: A => Id[B]): Id[B] =
    func(initial)
  ```
- `map`과 `flatMap`이 같다.
