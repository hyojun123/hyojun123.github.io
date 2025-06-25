---
layout: post
title: '이펙티브 자바 아이템 1: 생성자 대신 정적 팩터리 메서드를 고려하라'
author : 효준
date: 2024-06-13 11:00
tags: [effectivejava, java, oop]
image: /files/covers/blog.jpg
---

# 이펙티브 자바 아이템 1: 생성자 대신 정적 팩터리 메서드를 고려하라

## 핵심 내용
- 객체를 생성할 때 생성자(constructor) 대신 **정적 팩터리 메서드(static factory method)**를 사용하는 것을 고려하라는 내용입니다.

## 정적 팩터리 메서드란?
- 클래스의 인스턴스를 반환하는 **static 메서드**입니다.
- 예시: `valueOf`, `of`, `getInstance`, `newInstance`, `getType`, `newType` 등

```java
public class Boolean {
    public static Boolean valueOf(boolean b) {
        return b ? Boolean.TRUE : Boolean.FALSE;
    }
}
```

## 장점
1. **이름을 가질 수 있다**
   - 생성자와 달리 메서드 이름을 통해 반환될 객체의 특성을 쉽게 알릴 수 있다.
2. **호출할 때마다 새로운 객체를 생성하지 않아도 된다**
   - 불변 객체의 경우, 미리 만들어둔 객체를 반환할 수 있다(싱글턴, 캐싱 등).
3. **반환 타입의 하위 타입 객체를 반환할 수 있다**
   - 인터페이스를 반환 타입으로 지정하면, 실제 구현체를 숨길 수 있다.
4. **입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다**
5. **메서드 참조를 사용할 수 있다**
   - 예: `Supplier<MyObject> supplier = MyObject::of;`

## 단점
1. **상속을 하려면 public/protected 생성자가 필요하다**
2. **정적 팩터리만 있으면, 개발자가 찾기 어렵다**
   - 문서화가 중요하다.

## 대표적인 네이밍 관례
- `from`: 하나의 매개변수를 받아 해당 타입의 인스턴스를 반환
- `of`: 여러 매개변수를 받아 인스턴스를 반환
- `valueOf`: from과 of의 더 자세한 버전
- `instance`/`getInstance`: 인스턴스를 반환, 매번 같은 객체일 수도 있음
- `create`/`newInstance`: 매번 새로운 객체를 반환
- `getType`/`newType`: 타입이 다른 객체를 반환

## 결론
- 생성자 대신 정적 팩터리 메서드를 사용하면 유연하고 명확한 객체 생성이 가능하다.
- 상황에 따라 적절히 선택해서 사용하자.

---

이 글에서는 이펙티브 자바 3판의 아이템 1번(생성자 대신 정적 팩터리 메서드를 고려하라)의 핵심 내용을 정리했습니다. 