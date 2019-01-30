# 타입 클래스

- 타입 클래스는 일반적으로 오버로딩으로 알려져 있는 ad-hoc 다형성을 함수형 프로그래밍에서 사용하고 있는 강력한 도구다.
- 많은 객체지향 언어에서 다형성 코드를 위해 하위 타입을 만들어 쓰는 것처럼 함수형 프로그래밍 언어에서는 파라미터 다형성(타입 파라미터, 자바의 제너릭)과 ad-hoc 다형성을 조합한다.

## 예제: 리스트 접기

아래 코드는 숫자 리스트의 합을 구하고, 문자열을 합치고, 리스트 셑을 합치는 예제다.

```scala
def sumInts(list: List[Int]): Int = list.foldRight(0)(_ + _)

def concatStrings(list: List[String]): String = list.foldRight("")(_ ++ _)

def unionSets[A](list: List[Set[A]]): Set[A] = list.foldRight(Set.empty[A])(_ union _)
```

위 예제는 비슷한 패턴을 가진다. 초기 값(0, 빈 문자열, 빈 셑)과 합치는 함수(`+`,`++`,`union`). 
