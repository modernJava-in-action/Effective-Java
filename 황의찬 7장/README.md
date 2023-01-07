## 스트림은 주의해서 사용하라
---
#### 스트림을 과용하면 프로그램이 읽거나 유지보수하기 어려워진다.

#### 스트림으로 할 수 없는일

 - 코드 블록에서의 범위 안의 지역변수를 수정할 경우, 수정하는 건 불가능하다.(Atomic 종류(AtomicInteger, AtomicBoolean..)는 가능)
 - break, continue 문으로 종료하거나, skip하는 작업.
 - 내부 로직에서 예외를 던지는 작업.
 

#### 스트림으로 하기 좋은 작업 
 - 원소들의 시퀀스를 일관되게 변환한다.
 - 원소들의 시퀀스를 필터링한다.
 - 원소들의 시퀀스를 하나의 연산을 사용해 결합한다.(더하기, 연결하기, 최소값 구하기 등.)
 - 원소들의 시퀀스를 컬렉션에 모은다.
 - 원소들의 시퀀스에서 특정 조건을 만족하는 원소를 찾는다. 

#### 스트림으로 하기 어려운 작업
 - 파이프라인의 여러 단계를 통과할 때 이 데이터의 각 단계에서의 값들에 동시에 접근하기는 어려운
 경우다. 스트림 파이프라인은 일단 한 값을 다른 값에 매핑하고 나면 원래의 값은 잃기 때문이다.
 
---
### 요약
`
스트림을 사용해야 더 알맞거나, 반복 방식을 해야 더 알맞을 경우가 있다.
어느 쪽이 나은지가 확연히 드러나는 경우가 많겠지만, 확신하기 어려운 경우라면 
둘 다 해보고 더 나은 쪽을 선택하라.
`
## 스트림에서는 부작용 없는 함수를 사용하라

스트림의 핵심은 계산을 일련의 변횐으로 재구성 하는 부분이다. 이대 각 변환 단계는 이전 단계의 결과를 받아 처리하는 순수 함수여야 한다. 순수 함수란 오직 입력만이 결과에 영향을 주는 함수를 말한다. 그러려면 스트림 연산에 건네는 함수 객체는 모두 부작용이 없어야 한다.

아래는 텍스트 파일에서 단어별 수를 세어 빈도표를 만드는 코드이다:
```java
        Map<String, Long> freq = new HashMap<>();
        try (Stream<String> words = new Scanner(file).tokens()) {
            words.forEach(word -> {
                freq.merge(word.toLowerCase(), 1L, Long::sum);
            });
        }
```
이 코드는 스트림 코드를 가장한 반복적 코드다, 스트림의 이점을 살리지 못해서 길고, 읽기 어렵고, 유지보수에도 좋지 않다.

올바른 스트림 사용법을 알아보자:
```java
        Map<String, Long> freq;
        try (Stream<String> words = new Scanner(file).tokens()) {
            freq = words
                    .collect(groupingBy(String::toLowerCase, counting()));
        }
```
이전 코드에 비해 훨씬 가독성이 좋다. 스트림 메서드중에 forEach 연산은 종단 연산 중 기능이 가장 적고 가장 스트림답지 않다 (병렬화도 불가능하다). **forEach 연산은 스트림 계산 결과를 보고할 때만 사용하고, 계산할때는 사용하지 말자**. 위 코드는 collector를 사용하는데 스트림을 사용하려면 꼭 배워야 하는 새로운 개념이다. 
  수집기를 사용하면 스트림의 원소를 손쉽게 컬렉션으로 모을 수 있다. 수집기는 총 세 가지로 toList(), toSet(), toCollection(collectionFactory)가 있다. 이들은 차례대로 리스트, 집합, 프로그래머가 지정한 컬렉션 타입을 반환한다. 

  아래는 빈도표에서 가장 흔한 단어 10개를 뽑아내는 코드이다:
```java
        List<String> topTen = freq.keySet().stream()
                .sorted(comparing(freq::get).reversed())
                .limit(10)
                .collect(toList());
```
sorted(comparing(freq::get).reversed()) 에서 comparing 메서드는 키 추출 함수를 받는 비교자 생성 메서드다. 여기서 키 추출 함수로 쓰인 freq::get은 입력받은 단어를 빈도표에서 찾아 그 빈도를 반환한다. 그 후에 가장 흔한 단어가 위로 오도록 역순으로 정렬했다.

  다른 수집기중 간단한 맵 수집기는 **toMap(keyMapper, valueMapper)로 스트림 원소를 키에 매핑하는 함수와 값에 매핑하는 함수를 인수로 받는다. 아래 코드는 열거 타입 상수의 문자열 표현을 열거 타입 자체에 매핑하는 fromString을 구현하는 데 사용했다.
```java
  private static final Map<String, Operation> stringToEnum =
      Stream.of(values()).collect(
        toMap(Object::toString, e -> e)
      );
```

  더 복잡한 형태의 toMap은 키 매퍼와 값 매퍼는 물론 병합 함수까지 제공할 수 있다. 병합 함수의 형태는 BinaryOperator<U>이며 여기서 U는 해당 맵의 값 타입이다. 아래 코드는 각 키와 해당 키의 특정 원소를 연관 짓는 맵을 생성하는 수집기다:
```java
  Map<Artist, Album> topHits = albums.collect(
    toMap(Album::artist, a->a, maxBy(comparing(Album::sales)))
  );
```
여기서 비교자로는 BinaryOperator에서 정적 임포트한 maxBy라는 정적 팩터리 메서드를 사용했다. 비교자 생성 메서드인 comparing이 maxBy에 넘겨줄 비교자를 반환하는데, 자신의 키 추출 함수로는 Album::sales를 받았다. 이 코드를 말로 풀어보면 "앨범 스트림을 맵으로 바꾸는데, 이 맵은 각 음악가와 그 음악가의 베스트 앨범을 짝지은 것이다" 라고 할수있다. 