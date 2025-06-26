---
layout: post
title: '이펙티브 자바 아이템 3: private 생성자나 열거 타입으로 싱글턴임을 보증하라'
author : 효준
date: 2025-06-27 11:00
tags: [effectivejava, java, oop, design-pattern, singleton]
image: /files/covers/blog.jpg
---

# 이펙티브 자바 아이템 3: private 생성자나 열거 타입으로 싱글턴임을 보증하라

## 핵심 정리
> 싱글턴(Singleton)이란 인스턴스를 오직 하나만 생성할 수 있는 클래스를 말합니다. 이 글에서는 싱글턴을 만드는 세 가지 방법을 알아보고, 어떤 방법이 가장 좋은지 살펴봅니다.

## 1. 싱글턴을 만드는 방법

### 가. public static final 필드 방식

가장 간단한 방법입니다. `public static final` 필드로 유일한 인스턴스를 노출합니다.

```java
// public static final 필드 방식의 싱글턴
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }

    public void leaveTheBuilding() { ... }
}
```

- **장점:**
    1.  클래스가 싱글턴임이 API에 명확히 드러납니다.
    2.  코드가 간결합니다.
- **단점:**
    - 리플렉션(Reflection)을 사용하면 private 생성자를 호출하여 두 번째 인스턴스를 만들 수 있습니다.

### 나. 정적 팩터리 방식

`getInstance` 정적 팩터리 메서드를 통해 유일한 인스턴스를 반환합니다.

```java
// 정적 팩터리 방식의 싱글턴
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }

    public void leaveTheBuilding() { ... }
}
```

- **장점:**
    1.  API를 바꾸지 않고도 싱글턴이 아니게 변경할 수 있습니다. (예: 스레드별로 다른 인스턴스를 반환)
    2.  정적 팩터리를 제네릭 싱글턴 팩터리로 만들 수 있습니다.
    3.  정적 팩터리의 메서드 참조를 `Supplier`로 사용할 수 있습니다. (예: `Elvis::getInstance`)

### 다. 열거 타입(Enum) 방식

> **가장 좋은 싱글턴 구현 방식입니다.**

원소가 하나인 열거 타입을 선언하여 싱글턴을 구현합니다.

```java
// 열거 타입 방식의 싱글턴 - 바람직한 방법
public enum Elvis {
    INSTANCE;

    public void leaveTheBuilding() { ... }
}
```

- **장점:**
    1.  **더 간결하다:** public 필드 방식과 비슷하지만 코드가 더 짧습니다.
    2.  **직렬화가 자동으로 처리된다:** 복잡한 직렬화 상황이나 리플렉션 공격에서도 여러 인스턴스가 생기는 것을 완벽하게 막아줍니다.

## 2. 직렬화 문제

- `public static final` 필드 방식이나 정적 팩터리 방식은 직렬화/역직렬화 과정에서 새로운 인스턴스가 생길 수 있습니다.
- 이를 방지하려면 모든 인스턴스 필드를 `transient`로 선언하고 `readResolve` 메서드를 제공해야 하는 번거로움이 있습니다.
- **열거 타입 방식은 이런 걱정을 할 필요가 없습니다.**

## 결론
- 대부분의 상황에서는 원소가 하나인 **열거 타입(Enum)**이 싱글턴을 만드는 가장 좋은 방법입니다.
- 단, 만들려는 싱글턴이 `Enum` 외의 클래스를 상속해야 한다면 이 방법은 사용할 수 없습니다.

---

다음 글에서는 아이템 4번(인스턴스화를 막으려거든 private 생성자를 사용하라)에 대해 알아보겠습니다.
