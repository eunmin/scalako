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
