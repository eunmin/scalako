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
