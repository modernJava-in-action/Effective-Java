## 스트림은 주의해서 사용하라

스트림 API가 제공하는 추상 개념 중 핵심은 두 가지다:
  - 스트림은 데이터 원소의 유한 혹은 무한 시퀀스를 뜻한다.
  - 스트림 파이프라인은 이 원소들로 수행하는 연산 단계를 표현하는 개념이다.
스트림 파이프라인은 소스 스트림에서 시작해, 하나 이상의 중간 연산, 그리고 종단 연산으로 끝난다. 스트림 파이프라인은 **지연 평가 (lazy evaluation)** 되기 때문에 종단 연산이 호출되지 않으면 아무 일도 하지 않는 명령어인 *no-op*과 같게된다. 한마디로 스트림의 평가는 종단 연산이 호출될 때 이뤄진다는 뜻이다. 기본적으로 스트림 파이프라인은 순차적으로 수행된다, parallel 메서드를 사용해서 병렬로 실행할 수 있지만 이득을 볼수있는 상황은 그리 많지 않다. 스트림을 제대로 사용하면 프로그램이 짧고 간결해진다, 하지만 잘못 사용하면 읽기 어렵고 유지보수가 어려워지기 때문에 신중하게 사용해야 한다. 

아래 코드는 사전 파일에서 단어를 읽어 사용자가 지정한 값보다 원소 수가 많은 아나그램(anagram) 그룹을 출력한다. 아나그램이랑 철자를 구성하는 알파벳이 같고 순서만 다른 단어이다. 
```java
public class IterativeAnagrams {
    public static void main(String[] args) throws IOException {
        File dictionary = new File(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        Map<String, Set<String>> groups = new HashMap<>();
        try (Scanner s = new Scanner(dictionary)) {
            while (s.hasNext()) {
                String word = s.next();
                groups.computeIfAbsent(alphabetize(word),
                        (unused) -> new TreeSet<>()).add(word);
            }
        }

        for (Set<String> group : groups.values())
            if (group.size() >= minGroupSize)
                System.out.println(group.size() + ": " + group);
    }

    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```
이 프로그램은 맵에 각 단어를 삽입할 때 computeIfAbest 메서드를 활용한다. 이 메서드는 맵 안에 키가 있는지 찾은 다음, 있으면 그 키에 매핑 된 값을 반환한다. 키가 없으면 건네진 함수 객체를 키에 적용하여 값을 계산해낸 다음 그 키와 값을 매핑해놓고, 계산된 값을 반환한다.

이제 다음 코드를 살펴보자, 이전 코드와 같은 일을 하지만 스트림을 과하게 사용한 예이다.
```java
public class StreamAnagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(
                    groupingBy(word -> word.chars().sorted()
                            .collect(StringBuilder::new,
                                    (sb, c) -> sb.append((char) c),
                                    StringBuilder::append).toString()))
                    .values().stream()
                    .filter(group -> group.size() >= minGroupSize)
                    .map(group -> group.size() + ": " + group)
                    .forEach(System.out::println);
        }
    }
}
```
위 코드는 사전을 여는 부부만 제외하면 하나의 표현식으로 처리된다. 이 코드는 이전 코드보다는 짧지만 가독성이 떨어진다. 특히 스트림에 익숙하지 않은 프로그래머라면 더욱더 그럴것이다. 이처럼 스트림을 너무 과하게 사용하면 코드의 가독성이 떨어지고 유지보수하기도 어려워진다.

아래 코드는 스트림을 적절히 사용한 예이다:
```java
public class HybridAnagrams {
    public static void main(String[] args) throws IOException {
        Path dictionary = Paths.get(args[0]);
        int minGroupSize = Integer.parseInt(args[1]);

        try (Stream<String> words = Files.lines(dictionary)) {
            words.collect(groupingBy(word -> alphabetize(word)))
                    .values().stream()
                    .filter(group -> group.size() >= minGroupSize)
                    .forEach(g -> System.out.println(g.size() + ": " + g));
        }
    }

    private static String alphabetize(String s) {
        char[] a = s.toCharArray();
        Arrays.sort(a);
        return new String(a);
    }
}
```
이전에 스트림으로 모든걸 해결한 코드보다 훨씬 간결하고 이해하기 쉬워졌다. try-with-resources 블록에서 사전 파일을 열고, 파일의 모든 라인으로 구성된 스트림을 얻는다. 스트림 변수의 이름을 words로 지어 스트림 안의 각 원소가 단어임을 명확히 하고, 모든 단어를 수집해 맵으로 모은다. 이 맵은 단어들을 아나그램끼리 묶어놓은 것으로, 위 두 코드와 같은 역할을 수행한다. 그 후 이 맵의 values()가 반환한 값에서 새로운 Stream<List<String>> 스트림을 연다. 이 스트림의 원소는 아마그램 리스트이고 이 중 minGroupSize보다 작은 원소들은 필터링된다. 마지막으로 foreach에서 살아남은 원소들을 출력한다. alphabetize 메서드도 스트림을 사용할 수 있지만 그렇게하면 명확서이 떨어지고, 잘못 구현할수 있고, 심지어 속도도 느려질수 있다 (자바가 char용 스트림을 지원하지 않기 때문이다). 

아래는 char 값들을 스트림으로 처리하는 코드이다.
```java
"Hello world!".chars().forEach(System.out::print);
```
위 코드의 출력값은 721011081081113211911111410810033 이다. .chars()가 반환하는 스트림의 원소가 int 값이기 때문이다. 아래와 같이 명시적으로 형변환을 해줘야 우리가 원하는대로 출력이 된다:
```java
"Hello world!".chars().forEach(x -> System.out.print((char) x));
```
원하는 출력값을 얻었지만 char 값을 처리할 때는 스트림을 사용하지 않는것이 좋다.

처음 스트림을 사용하기 시작하면 모든 반복문을 스트림으로 바꾸고 싶겠지만 너무 서두르지 않는것이 좋다. 위의 예제들처럼 스트림으로 변환했을때의 단점도 명확히 존재하기 때문이다. HybridAnagrams의 예처럼 스트림과 반복문을 적절히 조합하여 사용하는것이 최선의 사용법이다.

**함수 객체로는 할 수 없지만 코드 블록으로는 할 수 있는 것들**
  - 코드 블록에서는 범위 안의 지역변수를 읽고 수정할 수 있다. 하지만 람다에서는 지역변수를 수정할 수 없다.
  - 코드 블록에서는 return 문을 사용해 메서드에서 빠져나가거나, break나 continue 문으로 블록 바깥의 반복문을 종료하거나 반복을 한 번 건너뛸 수 있다. 또한 메서드 선언에 명시된 검사 예외를 던질 수 있다. 람다로는 모두 불가능한 일이다.

**스트림을 활용하기 좋은 경우**
  - 원소들의 시퀀스를 일관되게 변환한다.
  - 원소들의 시퀀스를 필터링한다.
  - 원소들의 시퀀스를 하나의 연산을 사용해 결합한다 (더하기, 연결하기, 최솟값 구하기 등).
  - 원소들의 시퀀스를 컬렉션에 모은다.
  - 원소들의 시퀀스에서 특정 조건을 만족하는 원소를 찾는다.

-------------------------------------------------

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