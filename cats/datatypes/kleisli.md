# Kleisli

Kleisli로 `Option`, `Either` 같은 이상하고 다루기 어려운 파라미터를 받지 않고 `Option[Int]`이나
`Either[String, List[Double]]` 같은 모나딕 값을 리턴하는 함수를 합성할 수 있다.

어떤 환경에 의존하고 있는 여러 함수를 모두 같은 환경을 받을 수 있도록 합성할 수 있는 좋은 방법이 필요할
수 있다. 또는 로컬 설정에 의존하고 있는 여러 함수의 설정을 모아 전역 설정으로 구성할 수 있다. 그럼 어떻게
함수가 로컬에 필요한 사항만 알고 있으면서 각 기능을 잘 연동 시킬 수 있을까?

이 상황에 `Kleisli`이 도움이 된다.

## 함수

합성(`compose`)은 함수의 가장 유용한 기능이다. `A => B` 함수와 `B => C` 함수가 있을 때 이 둘을
합성해 새로운 `A => C` 함수를 만들 수 있다. 이런 기능으로 작은 함수를 합성해 필요에 맞춰 더 큰 함수를
만들 수 있다.

```scala
val twice: Int => Int =
  x => x * 2

val countCats: Int => String =
  x => if (x == 1) "1 cat" else s"$x cats"

val twiceAsManyCats: Int => String =
  twice andThen countCats // equivalent to: countCats compose twice
```

```scala
twiceAsManyCats(1) // "2 cats"
// res0: String = 2 cats
```

때로는 함수가 모나딕 값을 리턴할 수 있다. 다음 함수를 보자.

```scala
val parse: String => Option[Int] =
  s => if (s.matches("-?[0-9]+")) Some(s.toInt) else None

val reciprocal: Int => Option[Double] =
  i => if (i != 0) Some(1.0 / i) else None
```

위 두 함수는 `Function1.compose`(또는 `Function1.andThen`)으로 합성 할 수 없다. `parse`는
`Option[Int]`을 리턴하고 `reciprocal`는 `Int`을 받는다.

이제 `Kleisli`이 필요하다.

## Kleisli

`Kleisli[F[_], A, B]`는 단순히 `A => F[B]`를 감싸는 타입이다. `F[_]` 속성에 따라 `Kleisli`는
다른 일을 할 수 있다. 예를 들어 `F[_]`가 `FlatMap[F]` 인스턴스(`F[A]` 값의 `flatMap`이라고 부른다)
라면 두 함수를 합성하는 것처럼 두 `Kleisli`를 합성 할 수 있다.

```scala
import cats.FlatMap
import cats.implicits._

final case class Kleisli[F[_], A, B](run: A => F[B]) {
  def compose[Z](k: Kleisli[F, Z, A])(implicit F: FlatMap[F]): Kleisli[F, Z, B] =
    Kleisli[F, Z, B](z => k.run(z).flatMap(run))
}
```

앞에서 나온 예제를 다시 보면,

```scala
import cats.implicits._

val parse: Kleisli[Option,String,Int] =
  Kleisli((s: String) => if (s.matches("-?[0-9]+")) Some(s.toInt) else None)

val reciprocal: Kleisli[Option,Int,Double] =
  Kleisli((i: Int) => if (i != 0) Some(1.0 / i) else None)

val parseAndReciprocal: Kleisli[Option,String,Double] =
  reciprocal.compose(parse)
```

`Kleisli#andThen`도 비슷하게 정의 할 수 있다.

`F[_]`가 `FlatMap`(또는 `Monad`) 인스턴스를 가진다는 것이 중요한 점이고 이것은 어려운 제약 사항이
아니다. 이 작은 제약으로 유용한 것을 할 수 있다. 그러한 예는 `Kleisli#map`로 `F[_]`가 `Functor`
인스턴스(`map: F[A] => (A => B) => F[B]`를 할 수 있는)라는 작은 제약만 필요하다.

```scala
import cats.Functor

final case class Kleisli[F[_], A, B](run: A => F[B]) {
  def map[C](f: B => C)(implicit F: Functor[F]): Kleisli[F, A, C] =
    Kleisli[F, A, C](a => F.map(run(a))(f))
}
```

아래 `F[_]`가 만족해야 할 제약과 함께 더 많은 `Kleisli` 함수가 있다.

```
함수       | `F[_]` 제약
--------- | -------------------
andThen   | FlatMap
compose   | FlatMap
flatMap   | FlatMap
lower     | Monad
map       | Functor
traverse  | Applicative
```

## 타입 클래스 인스턴스

`Kleisli` 타입 클래스 인스턴스는 함수 처럼 입력 타입(과 `F[_]`)은 고정하고 출력 타입은 자유롭게 할 수
있다. 어떤 타입 클래스 인스턴스가 될지는 `F[_]`가 어떤 인스턴스를 가지고 있는지에 따라 달라진다. 예를 들어
`Kleisli[F, A, B]`는 선택된 `F[_]`가 있는 한 `Functor` 인스턴스가 된다. 선택된 `F[_]`에 따라
`Monad` 인스턴스가 된다. Cats에 있는 인스턴스는 implicit가 가능한 가장 구체적인 인스턴스를 선택하는
방법으로 선택된다.

`Kleisli`의 `Monad` 인스턴스의 예제는 아래와 같다.

아래 예제는 `kind-projector compiler plugin`을 사용한다고 가정한다. 만약 프로젝트에서 이 플러그인을
쓰지 않는다면 컴파일이 안된다.

```scala
import cats.implicits._

// We can define a FlatMap instance for Kleisli if the F[_] we chose has a FlatMap instance
// Note the input type and F are fixed, with the output type left free
implicit def kleisliFlatMap[F[_], Z](implicit F: FlatMap[F]): FlatMap[Kleisli[F, Z, ?]] =
  new FlatMap[Kleisli[F, Z, ?]] {
    def flatMap[A, B](fa: Kleisli[F, Z, A])(f: A => Kleisli[F, Z, B]): Kleisli[F, Z, B] =
      Kleisli(z => fa.run(z).flatMap(a => f(a).run(z)))

    def map[A, B](fa: Kleisli[F, Z, A])(f: A => B): Kleisli[F, Z, B] =
      Kleisli(z => fa.run(z).map(f))

    def tailRecM[A, B](a: A)(f: A => Kleisli[F, Z, Either[A, B]]) =
      Kleisli[F, Z, B]({ z => FlatMap[F].tailRecM(a) { f(_).run(z) } })
  }
```

아래는 `F[_]`가 어떤 인스턴스를 가지고 있는지에 따른 `Kleisli` 타입 클래스 표다.

```
타입 클래스       | `F[_]` 제약
-------------- | -------------------
Functor        | Functor
Apply          | Apply
Applicative    | Applicative
FlatMap        | FlatMap
Monad          | Monad
Arrow          | Monad
Split          | FlatMap
Strong         | Functor
SemigroupK*    | FlatMap
MonoidK*       | Monad
```

## 다른 사용 예

### 모나드 트랜스포머

### 설정

함수형 프로그래밍은 작고 단순한 모듈을 조합해 프로그램과 모듈을 만느는 방식을 선호한다. 이 철학은 함수
합성과 닮아 있다. 작은 함수를 많이 만들고 하나의 큰 것을 만들기 위해 합성한다. 어쨌든 프로그램은 함수다.

함수로 유효성 검증이 된 자신만의 설정을 갖고 있는 모듈의 예를 보자. 만약 설정이 유효하다면 `Some` 모듈을
리턴하고 그렇지 않으면 `None`을 리턴한다. 이 예제는 단순히 `Option`을 쓴다. 만약 에러 메시지나 실패에
대한 다른 컨텍스트를 전달하려면 `Either`를 사용하면 된다.

```scala
case class DbConfig(url: String, user: String, pass: String)
trait Db
object Db {
  val fromDbConfig: Kleisli[Option, DbConfig, Db] = ???
}

case class ServiceConfig(addr: String, port: Int)
trait Service
object Service {
  val fromServiceConfig: Kleisli[Option, ServiceConfig, Service] = ???
}
```

`Db`(데이터 접속을 하기 위한)와 `Service`(웹 상에 API를 제공하기 위한) 두 독립적인 모듈이 있다.
둘 다 각각의 설정 파라미터에 의존적이다. 이것은 다른 모듈이 알거나 신경쓰지 않아야 한다. 하지만 애플리케이션이
동작하려면 두 모듈이 필요하다. 이제 더 글로벌 한 설정이 필요하다.

```scala
case class AppConfig(dbConfig: DbConfig, serviceConfig: ServiceConfig)

class App(db: Db, service: Service)
```

하나는 `DbConfig`를 받고 다른 하나는 `ServiceConfig`를 받기 때문에 두 `Kleisli`의 유효성 검사
함수를 잘 쓸 수 없다. 이것은 `FlatMap`(`Monad`를 확장한) 인스턴스가
다르다(입력 타입은 타입 클래스 인스턴스에 의해 고정되어 있다는 것을 기억하라)는 뜻이다. 하지만 `Kleisli`에
`local`이란 좋은 함수가 있다.

```scala
final case class Kleisli[F[_], A, B](run: A => F[B]) {
  def local[AA](f: AA => A): Kleisli[F, AA, B] = Kleisli(f.andThen(run))
}
```

`local`은 근본적으로 입력 타입을 더 일반적인 타입으로 확장한다. 예제에서는 `DbConfig`나 `ServiceConfig`인
`Kleisli`를 받을 수 있고 `AppConfig`가 필요한 것으로 만들 수 있다. `AppConfig`를 어떻게 다른
설정으로 바꿀지에 대해 기술하면 된다.
