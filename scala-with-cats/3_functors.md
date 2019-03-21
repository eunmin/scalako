# Functors

- Functor는 List, Option등과 같이 컨텍스트 안에 연속된 동작을 나타내는 추상이다.
- Functor 자체는 그렇게 쓸모 있지 않지만 모나드와 어플리커티브 Functor와 같은 것들은 Cats 추상에서
  많이 사용된다.

## Functor 예제

- Functor는 map 메서드다. Option, List, Either 같은 것들이 이미 제공하고 있는 것을 알고 있을 거다.
- 먼저 알아 볼 것은 List를 순회할 때 쓰는 map 이다. Functor를 이해하려면 이 메서드를 좀 다른 관점으로
  봐야한다. 리스트를 순회하는 함수로 보기 보다는 리스트 안에 있는 값을 다른 값으로 변환 한다고 보는 것이
  Functor를 이해하는데 도움이 된다. 안에 있는 값은 바뀌지만 감싸고 있는 구조는 변함이 없다.
  ```scala
  List(1, 2, 3).map(n => n + 1)
  // res0: List[Int] = List(2, 3, 4)
  ```
- Option에 있는 map 도 비슷하게 안에 값은 바뀌지만 Option 컨텍스트는 변함이 없다.
- Either도 마찬가지다.
- 컨텍스트 구조는 변하지 않기 때문에 여러 계산을 연속적으로 할 수 있다.
  ```scala
  List(1, 2, 3).
    map(n => n + 1).
    map(n => n * 2).
    map(n => n + "!")
  // res1: List[String] = List(4!, 6!, 8!)
  ```
- map 메서드를 순회하는 패턴으로 생각하지 말고 관계된 타입(컨텍스트)를 무시하면서 순차적으로 계산 할 수 있는
  방법으로 생각해야한다.
  - Option - 값이 있거나 없거나
  - Either - 값이거나 에러거나
  - List - 값이 없거나 여러개 거나

## 더 많은 Functor 예제

- List, Option, Either에서 쓴 map 메서드보다 더 일반적으로 연속된 계산을 보여주는 예제를 더 알아보자.

### Futures

- Futures는 계산을 큐에 넣고 완료된 작업을 적용하는 연속된 비동기 계산을 위한 Functor다.
- Futures map의 타입 시그네처는 앞에서 본것 과 같지만 동작은 조금 다르다.
- Futures를 사용할 때 안에 들어 있는 값의 상태를 보장할 수 없다. Futures가 완료되면 map 함수가 바로 불란다.
  하지만 완료되지 않았다면 함수 호출은 스레드 풀에 있고 나중에 불린다. 그래서 언제 불릴지 모른다. 하지만
  어떤 순서로 불릴지는 안다.그래서 Futures는 Option, List, Either와 같은 연속 된 동작을 할 수 있다.
  ```scala
  import scala.concurrent.{Future, Await}
  import scala.concurrent.ExecutionContext.Implicits.global
  import scala.concurrent.duration._

  val future: Future[String] =
    Future(123).
      map(n => n + 1).
      map(n => n * 2).
      map(n => n + "!")

  Await.result(future, 1.second)
  // res3: String = 248!
  ```

  - Future는 순수함수의 좋은 예가 아니다. 왜냐하면 참조 투명성이 보장되지 않기 때문이다. Future 안에
    사이드 이팩트가 있는 연산이 있다면 Future의 결과는 항상 같지 않을 수 있다. 그리고 Future의 시작을
    사용자가 제어할 수 없는 것도 다른 문제다.
    ```scala
    import scala.util.Random

    val future1 = {
      // Initialize Random with a fixed seed:
      val r = new Random(0L)
      // nextInt has the side-effect of moving to
      // the next random number in the sequence:
      val x = Future(r.nextInt)

      for {
        a <- x
        b <- x
      } yield (a, b)
    }

    val future2 = {
      val r = new Random(0L)

      for {
        a <- Future(r.nextInt)
        b <- Future(r.nextInt)
      } yield (a, b)
    }

    val result1 = Await.result(future1, 1.second)
    // result1: (Int, Int) = (-1155484576,-1155484576)

    val result2 = Await.result(future2, 1.second)
    // result2: (Int, Int) = (-1155484576,-723955400)
    ```

### Functions

- 인자가 하나인 함수는 Functor다. 함수 A => B는 두 타입 파라미터가 있다. 타입 A는 파라미터고 타입 B는
  결과다. 이것을 바른 모양으로 억지로 만들어 보면 아래 처럼 만들 수 있다.
  - X => A 로 시작
  - A => B 적용
  - X => B 를 얻음

- X => A를 MyFunc[A]라고 하면 위에서 본 Functor형식과 같다.
  - MyFunc[A]로 시작
  - A => B 적용
  - MyFunc[B]를 얻음

- 즉 Function1에 매핑은 함수 조합이다.
  ```scala
  import cats.instances.function._ // for Functor
  import cats.syntax.functor._     // for map

  val func1: Int => Double =
    (x: Int) => x.toDouble

  val func2: Double => Double =
    (y: Double) => y * 2

  (func1 map func2)(1) // composition using map
  // res7: Double = 2.0

  (func1 andThen func2)(1) // composition using andThen
  // res8: Double = 2.0

  func2(func1(1))          // composition written out by hand
  // res9: Double = 2.0
  ```

- 이것이 일반적인 순차적인 연산과 무슨 관계가 있을까? 함수 조합은 순차적이다. 하나의 인자로 시작한 함수가
  다른 연산들을 얼마든지 붙여서 연결할 수 있다.
- map을 부르는 것은 실제 아무것도 하지 않지만 인자를 넘기면 순차적으로 계산이된다.
- 이것은 Future와 비슷하게 작업을 지연 큐에 넣은 것으로 볼 수 있다.
  ```scala
  val func =
    ((x: Int) => x.toDouble).
      map(x => x + 1).
      map(x => x * 2).
      map(x => x + "!")

  func(123)
  // res10: String = 248.0!
  ```

- 위 코드가 동작하려면 `build.sbt`에 `scalacOptions += "-Ypartial-unification"`를 추가해야한다.

## Functor 정의

- 지금까지 살펴본 예가 모두 Functor이다: 연속된 계산 속에 있는 클래스.
- Functor는 (A => B) => F[B] 형식의 map 연산을 가진 F[A] 타입이다.
- 일반 타입표는 아래서 보겠다.
- Cats에서 Fuctor는 cats.Functor 타입 클래스로 되있다. 메서드를 보면 조금 다르다.
- 변환 함수 한쪽에 F[A] 타입을 받는다. 간단한 버전은 아래와 같다.
  ```scala
  package cats

  import scala.language.higherKinds

  trait Functor[F[_]] {
    def map[A, B](fa: F[A])(f: A => B): F[B]
  }
  ```
- `F[_]` 이런 문법을 처음 본다면 타입 생성자와 higher kinded 타입에 대해 배울 차례다.
- `scala.language`를 import 하는 것도 설명하겠다.

### Functor 법칙
- Functor는 많은 작은 동작을 하나씩 순회하거나 매핑하기 전에 큰 함수로 조합하거나 상관 없이 같은 의미를
  보장한다.
- 이게 되려면 다음 법칙을 따라야한다.
- Identity: identity 함수로 map을 하면 같은 값이어야 한다.
  ```scala
  fa.map(a => a) == fa
  ```
- Composition: f와 g 두함수를 매핑한것과 f를 매핑하고 g를 매핑한것이 같아야한다.
  ```scala
  fa.map(g(f(_))) == fa.map(f).map(g)
  ```

## 부록: Higher Kinds와 타입 생성자

- Kind는 타입에 대한 타입이다. 이건 타입에 몇개 빈칸이 있는 것과 같다.
- 빈칸이 없는 일반 타입과 타입을 만들 때 채워 넣을 수 있는 타입 생성자와 구분한다.
- 예를 들어 List는 빈칸 하나가 있는 타입 생성자다. 일반 타입을 만들 때 List[Int]나 List[A] 같이
  빈 칸에 특정 타입을 넣는다.
- 이것은 타입 생성자와 제너릭 타입을 혼동하는 것이 아니다. List는 타입 생성자고 List[A]는 타입이다.
  ```scala
  List    // 인자 하나를 받는 타입 생성자
  List[A] // 타입 인자를 써서 만든 타입
  ```
- 이건 함수랑 값과 유사하다. 함수는 인자를 넣으면 값이 만들어지는 값 생성자다.
  ```scala
  math.abs    // 인자 하나를 받은 함수
  math.abs(x) // 값 인자를 써서 만든 값
  ```
- 스칼라에서는 언더스코어로 타입 생성자를 선언한다. 위에서 봤지만 간단히 다시 보자.
  ```scala
  // 언더스코어를 써서 F 선언:
  def myMethod[F[_]] = {
    // 언더스코어 없이 F를 참조:
    val functor = Functor.apply[F]
    // ...
  }
  ```
- 이건 함수 정의에 있는 파라미터와 함수 자체를 참조하는 것과 비슷하다:
  ```scala
  // 인자를 써서 f 선언:
  val f = (x: Int) => x * 2
  // 인자 없이 f를 참조:
  val f2 = f andThen f
  ```
- Cats에서 Functor는 List나 Option, Future 또는 MyFunc 같은 타입 앨리어스 처럼 어떤 타입이든
  하나를 인자로 받아 인스턴스를 생성할 수 있도록 정의 되있다.

### Language Feature import

- Higher Kinds는 스칼라에서 고급 언어로 되어 있다. 그래서 `A[_]` 같은 타입 생성자 문법을 쓰려면
  컴파일러 경고를 막기 위해 Higher Kinded type 언어 기능을 활성화 해야한다.
- `language import`를 하거나
  ```scala
  import scala.language.higherKinds
  ```
- `build.sbt`에 `scalacOptions`에 아래와 같이 추가한다.
  ```scala
  scalacOptions += "-language:higherKinds"
  ```
- 이 책에서는 가능한 `language import`를 하겠지만 진짜 쓰려면 `scalacOptions`에 설정하고 쓰는
  것이 더 간편하다.

## Cats에서 Functor

- Cats에서 구현된 Functor를 보자. 모노이드에서 봤던 것처럼 살펴보자: 타입 클래스, 인스턴스, 문법

### Functor 타입 클래스

- Cats의 Functor 타입 클래스는 cats.Functor에 있다. 인스턴스는 동반 객체에 정의된 Functor의 기본
  apply 메서드로 만들 수 있다. 일반적으로 기본 인스턴스는 cats.intances에 있다.
  ```scala
  import scala.language.higherKinds
  import cats.Functor
  import cats.instances.list._   // Functor를 쓰려고
  import cats.instances.option._ // Functor를 쓰려고

  val list1 = List(1, 2, 3)
  // list1: List[Int] = List(1, 2, 3)

  val list2 = Functor[List].map(list1)(_ * 2)
  // list2: List[Int] = List(2, 4, 6)

  val option1 = Option(123)
  // option1: Option[Int] = Some(123)

  val option2 = Functor[Option].map(option1)(_.toString)
  // option2: Option[String] = Some(123)
  ```
- Functor에는 A => B 함수를 F[A] => F[B] 함수로 변환하는 `lift` 함수가 있다.
  ```scala
  val func = (x: Int) => x + 1
  // func: Int => Int = <function1>

  val liftedFunc = Functor[Option].lift(func)
  // liftedFunc: Option[Int] => Option[Int] = cats.Functor$$Lambda$11699/1790382668@37dabefe

  liftedFunc(Option(1))
  // res0: Option[Int] = Some(2)
  ```

### Functor 문법

- Functor에서 가장 중심이 되는 문법은 map 메서드다.
- List나 Option은 기본으로 map 메서드가 있는데 스칼라 컴파일러는 기본 메서드가 확장 메서드 보다
  우선하기 때문에 이걸 쓰기가 복잡하다.
- 예제 두개로 살펴보자.
- 먼저 함수를 매핑하는 것을 보자. 스칼라 Function1 타입은 map 메서드가 없다. (대신 andThen을 쓴다.)
  그래서 이름이 겹치지 않는다.
  ```scala
  import cats.instances.function._ // for Functor
  import cats.syntax.functor._     // for map

  val func1 = (a: Int) => a + 1
  val func2 = (a: Int) => a * 2
  val func3 = (a: Int) => a + "!"
  val func4 = func1.map(func2).map(func3)

  func4(123)
  // res1: String = 248!
  ```

- 다른 예제를 보자.
- 특정 구체 타입이 동작하지 않도록 functor로 추상화한다. 어떤 functor 컨텍스트에 관계 없이 숫자 계산을
  하기 위해 메서드를 만듭니다.
  ```scala
  def doMath[F[_]](start: F[Int])(implicit functor: Functor[F]): F[Int] =
    start.map(n => n + 1 * 2)

  import cats.instances.option._ // for Functor
  import cats.instances.list._   // for Functor

  doMath(Option(20))
  // res3: Option[Int] = Some(22)
  doMath(List(1, 2, 3))
  // res4: List[Int] = List(3, 4, 5)
  ```
- 이게 어떻게 동작하는지 알아보기 위해 cats.syntax.functor에 map 메서드가 어떻게 정의되어 있는지
  봅시다.
- 간단한 코드는 이러합니다.
  ```scala
  implicit class FunctorOps[F[_], A](src: F[A]) {
    def map[B](func: A => B)(implicit functor: Functor[F]): F[B] =
      functor.map(src)(func)
  }
  ```
- 컴파일러는 빌트인 map 메서드가 없다면 이 확장 메서드를 씁니다.
  ```scala
   foo.map(value => value + 1)
  ```
- `foo`는 빌트는 map 메서드가 없다고 가정하면 컴파일러는 잠재적 에러를 발견하거나 FunctorOps 표현으로
  아래 코드 처럼 감싼다.
  ```scala
  new FunctorOps(foo).map(value => value + 1)
  ```
- Functor에 있는 map 메서드는 implicit Functor 인자가 필요하다. 스코프 안에 expr1에 대한 Functor
  가 없다면 에러가 발생할 것이다.
  ```scala
  final case class Box[A](value: A)

  val box = Box[Int](123)

  box.map(value => value + 1)
  // <console>:34: error: value map is not a member of Box[Int]

  //        box.map(value => value + 1)
  //            ^
  ```

### 커스텀 타입 인스턴스

- 간단히 map 메서드를 정의하면 functor를 정의 할 수 있다. 이미 cats.instances에 있지만 다음은
  Option에 대한 Functor 예제다. 구현은 특별하지 않다. 단순히 Option에 원래 있는 map 메서드를 부른다.
  ```scala
  implicit val optionFunctor: Functor[Option] =
      new Functor[Option] {
        def map[A, B](value: Option[A])(func: A => B): Option[B] = value.map(func)
      }
  ```
- 인스턴스를 만들다보면 의존성이 필요할 때도 있다. 예를 들어 Future에 대한 Functor가 있다면 future.map에
  implicit ExecutionContext 가 필요하다. Functor map에 추가 파라미터를 받을 수 없기 때문에
  의존성으로 봐야한다.
  ```scala
  import scala.concurrent.{Future, ExecutionContext}

  implicit def futureFunctor(implicit ec: ExecutionContext): Functor[Future] =
    new Functor[Future] {
      def map[A, B](value: Future[A])(func: A => B): Future[B] =
        value.map(func)
    }
  ```
- Functor.apply를 부르거나 아니면 map 확장 메서드를 부르거나 해서 Future의 Functor가 필요할때는
  컴파일러가 먼저 futureFunctor를 찾고 그 다음에 재귀적으로 ExecutionContex를 해석한다. 어떻게
  확장되는지는 다음과 같다.
  ```scala
  // 이렇게 코드를 쓰면:
  Functor[Future]

  // 컴파일러는 첫번째로 아래와 같이 확장한다:
  Functor[Future](futureFunctor)

  // 다음은 아래와 같이 확장한다:
  Functor[Future](futureFunctor(executionContext))
  ```

### 연습문제

- 다음 이진 트리에 대한 Functor를 작성하시오. 예상하대로 동작하는지는 Branch와 Leaf 인스턴스로 확인해
  보시오.
  ```scala
  sealed trait Tree[+A]

  final case class Branch[A](left: Tree[A], right: Tree[A])
    extends Tree[A]

  final case class Leaf[A](value: A) extends Tree[A]
  ```

```scala
implicit val treeFunctor = new Functor[Tree] {
  override def map[A, B](tree: Tree[A])(f: A => B): Tree[B] =
    tree match {
      case Branch(left, right) => Branch(map(left)(f), map(right)(f))
      case Leaf(value) => Leaf(f(value))
    }
}
```

## Either

- 이제 다른 유용한 모나드인 스칼라 기본 라이브러리에 있는 `Either`타입 에 대해 살펴보자. 스칼라 2.11
  이전에는 많은 사람들이 `Either` 모나드를 신경쓰지 않았다. 왜냐면 `Either`에 `map`이나
  `flatMap` 메서드가 없었기 때문이다. 하지만 스칼라 2.12 부터 `Either`는 right로 편향 되었다.

### Left와 Right 편향

- 스칼라 2.11에는 `map`과 `flatMap`이 없었다. 그래서 2.11 버전에서는 `Either`를 for 구문에 쓰기
  불편했다. for 구문의 매 생성절에 `.right`를 붙었었다.
  ```scala
  val either1: Either[String, Int] = Right(10)
  val either2: Either[String, Int] = Right(32)

  for {
    a <- either1.right
    b <- either2.right
  } yield a + b
  // res0: scala.util.Either[String,Int] = Right(42)
  ```
- 스칼라 2.12에서 `Either`가 재설계되었다. 새로운 `Either`는 `map`과 `flatMap`이 직접 right가
  성공으로 판단하도록 해서 for 구문이 더 좋아졌다.
  ```scala
  for {
    a <- either1
    b <- either2
  } yield a + b
  // res1: scala.util.Either[String,Int] = Right(42)
  ```
- Cats는 스칼라 2.11에서 `cats.syntax.either`를 import 해서 right 편향을 지원하기 때문에 모든
  스칼라 버전에서 동일한 구문을 쓸 수 있다. 2.12에 써도 문제가 없다.
  ```scala
  import cats.syntax.either._ // for map and flatMap

  for {
    a <- either1
    b <- either2
  } yield a + b
  ```

### 인스턴스 만들기

- `Left`나 `Right` 인스턴스를 직접 만들기 위해 `cats.syntax.either`에 있는 `asLeft`나
  `asRight`를 쓸 수 있다.
  ```scala
  import cats.syntax.either._ // for asRight

  val a = 3.asRight[String]
  // a: Either[String,Int] = Right(3)

  val b = 4.asRight[String]
  // b: Either[String,Int] = Right(4)

  for {
    x <- a
    y <- b
  } yield x*x + y*y
  // res4: scala.util.Either[String,Int] = Right(25)
  ```
- 이 똑똑한 생성자는 `Left.apply`나 `Right.apply`에 비해 장점이 있다. 왜냐햐면 이 생성자는
  `Either` 타입을 리턴하지 않고 `Left`나 `Right`타입을 리턴하기 때문이다. 그래서 타입 추론 버그를
  막는데 도움을 준다. 버그가 발생하는 예제는 다음과 같다.
  ```scala
  def countPositive(nums: List[Int]) =
    nums.foldLeft(Right(0)) { (accumulator, num) =>
      if(num > 0) {
        accumulator.map(_ + 1)
      } else {
        Left("Negative. Stopping!")
      }
    }
  // <console>:21: error: type mismatch;
  //  found   : scala.util.Either[Nothing,Int]
  //  required: scala.util.Right[Nothing,Int]
  //              accumulator.map(_ + 1)
  //                             ^
  // <console>:23: error: type mismatch;
  //  found   : scala.util.Left[String,Nothing]
  //  required: scala.util.Right[Nothing,Int]
  //              Left("Negative. Stopping!")
  //                  ^
  ```
