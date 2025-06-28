---
layout: post
title: Effective Java Item 6: 불필요한 객체 생성을 피하라
author: 효준
date: 2025-06-29 09:30:00 +0900
categories: [Java, Effective Java]
tags: [Effective Java, 객체 생성, 성능 최적화, 메모리 관리, 재사용, 불변 객체]
---

# Effective Java Item 6: 불필요한 객체 생성을 피하라

## 서론

자바 프로그래밍에서 객체 생성은 매우 자연스러운 일이지만, 때로는 불필요한 객체 생성이 애플리케이션의 성능과 메모리 사용에 부정적인 영향을 미칠 수 있습니다. **Effective Java**의 **아이템 6**은 "불필요한 객체 생성을 피하라"고 조언하며, 효율적인 자바 코드를 작성하기 위한 중요한 지침을 제공합니다. 이 아이템은 객체 재사용의 중요성을 강조하며, 어떻게 하면 불필요한 객체 생성을 줄일 수 있는지에 대한 구체적인 방법을 제시합니다.

이 글에서는 Effective Java Item 6의 핵심 원칙과 함께, 실제 코드에서 불필요한 객체 생성을 피하고 성능을 최적화하는 다양한 전략을 살펴보겠습니다.

## 본론

### 1. 객체 재사용의 중요성

새로운 객체를 생성하는 것은 비용이 드는 작업입니다. 객체 생성에는 메모리 할당, 생성자 호출, 가비지 컬렉션에 의한 회수 등 여러 오버헤드가 발생합니다. 따라서 동일한 기능을 수행하는 객체가 여러 번 필요할 경우, 매번 새로운 객체를 생성하기보다는 기존 객체를 재사용하는 것이 효율적입니다.

### 2. 불필요한 객체 생성을 피하는 방법

#### 2.1. 불변 객체(Immutable Objects) 재사용

불변 객체는 한 번 생성되면 그 상태를 변경할 수 없는 객체입니다. 이러한 객체는 항상 안전하게 재사용할 수 있습니다.

*   **`String` 리터럴 사용:**
    `new String("bikini")`와 같이 `new` 연산자를 사용하여 `String` 객체를 생성하면 매번 새로운 객체가 생성됩니다. 반면, `String s = "bikini";`와 같이 문자열 리터럴을 사용하면 JVM의 문자열 상수 풀(String Pool)에서 기존 객체를 재사용합니다.

    ```java
    String s1 = new String("hello"); // 새로운 객체 생성
    String s2 = "hello";             // 기존 객체 재사용 (상수 풀)
    ```

*   **`Boolean.valueOf()`와 같은 정적 팩토리 메서드 사용:**
    `Boolean`과 같은 래퍼 클래스는 `new Boolean(true)` 대신 `Boolean.valueOf(true)`와 같은 정적 팩토리 메서드를 제공합니다. `valueOf` 메서드는 내부적으로 미리 생성된 인스턴스를 반환하여 불필요한 객체 생성을 방지합니다.

    ```java
    Boolean b1 = new Boolean(true); // 새로운 객체 생성 (권장하지 않음)
    Boolean b2 = Boolean.valueOf(true); // 기존 객체 재사용
    ```

#### 2.2. 오토박싱(Autoboxing) 주의

기본 타입(primitive type)과 박싱된 기본 타입(boxed primitive type) 간의 자동 변환인 오토박싱은 편리하지만, 불필요한 객체 생성을 유발할 수 있습니다. 특히 반복문 내에서 박싱된 기본 타입을 사용하면 성능 저하의 원인이 될 수 있습니다.

```java
// 나쁜 예: 불필요한 Long 객체 수백만 개 생성
Long sum = 0L;
for (long i = 0; i < Integer.MAX_VALUE; i++) {
    sum += i; // 매번 Long 객체 생성
}

// 좋은 예: 기본 타입 사용
long sum = 0L;
for (long i = 0; i < Integer.MAX_VALUE; i++) {
    sum += i;
}
```

#### 2.3. 비용이 비싼 객체 캐싱 및 재사용

생성 비용이 많이 드는 객체(예: `SimpleDateFormat`, `Pattern` 객체)는 한 번 생성한 후 재사용하는 것이 좋습니다.

*   **`Pattern` 객체:** 정규 표현식 컴파일은 비용이 많이 드는 작업이므로, `Pattern` 객체는 한 번만 생성하여 재사용해야 합니다.

    ```java
    // 나쁜 예: 매번 Pattern 객체 생성
    public boolean isRomanNumeralBad(String s) {
        return s.matches("^(?=.)M*(C[MD]|D?C{0,3})(X[LC]|L?X{0,3})(I[XV]|V?I{0,3})$");
    }

    // 좋은 예: Pattern 객체 재사용
    private static final Pattern ROMAN = Pattern.compile(
        "^(?=.)M*(C[MD]|D?C{0,3})(X[LC]|L?X{0,3})(I[XV]|V?I{0,3})$");

    public boolean isRomanNumeralGood(String s) {
        return ROMAN.matcher(s).matches();
    }
    ```

*   **`SimpleDateFormat` 객체:** `SimpleDateFormat`은 스레드 안전(thread-safe)하지 않으므로, 스레드마다 별도의 인스턴스를 사용하거나 `ThreadLocal`을 활용하여 재사용해야 합니다.

#### 2.4. `StringBuilder`를 이용한 문자열 연결

반복문 내에서 `+` 연산자를 사용하여 문자열을 연결하면 매번 새로운 `String` 객체가 생성되어 성능 저하를 일으킵니다. 대신 `StringBuilder`를 사용하면 효율적으로 문자열을 조작할 수 있습니다.

```java
// 나쁜 예: 불필요한 String 객체 다수 생성
String result = "";
for (int i = 0; i < 1000; i++) {
    result += i;
}

// 좋은 예: StringBuilder 사용
StringBuilder sb = new StringBuilder();
for (int i = 0; i < 1000; i++) {
    sb.append(i);
}
String result = sb.toString();
```

#### 2.5. 빈 컬렉션 상수 활용

빈 리스트, 세트, 맵을 반환할 때는 `Collections.emptyList()`, `Collections.emptySet()`, `Collections.emptyMap()`과 같은 상수를 사용하는 것이 좋습니다. 이는 새로운 빈 컬렉션 인스턴스를 생성하는 것을 방지합니다.

```java
// 나쁜 예: 매번 새로운 빈 리스트 생성
public List<String> getEmptyListBad() {
    return new ArrayList<>();
}

// 좋은 예: 빈 리스트 상수 재사용
public List<String> getEmptyListGood() {
    return Collections.emptyList();
}
```

## 결론

Effective Java Item 6은 자바 개발자가 성능과 메모리 효율성을 고려하여 코드를 작성하는 데 필수적인 지침을 제공합니다. 불필요한 객체 생성을 피하고 객체를 재사용하는 습관을 들이는 것은 애플리케이션의 전반적인 품질을 향상시키는 데 크게 기여합니다.

코드 리뷰 시 불필요한 객체 생성이 없는지 확인하고, 위에서 제시된 방법들을 적극적으로 활용하여 더 효율적이고 견고한 자바 애플리케이션을 만들어나가시길 바랍니다.

---
