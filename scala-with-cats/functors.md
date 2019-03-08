# Functors

- Functor는 List, Option등과 같이 컨텍스트 안에 연속된 동작을 나타내는 추상이다.
- Functor 자체는 그렇게 쓸모 있지 않지만 모나드와 어플리커티브 Functor와 같은 것들은 Cats 추상에서
  많이 사용되는 것들이다.

## Functor 예제

- Functor는 map 메서드다. Option, List, Either 같은 것들이 이미 제공하고 있는 것을 알고 있을 거다.
- 먼저 알아 볼 거는 List를 순회할 때 쓰는 map 이다. Functor를 이해하려면 이 메서드를 좀 다른 관점으로
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
