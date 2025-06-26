---
layout: post
title: '이펙티브 자바 아이템 2: 생성자에 매개변수가 많다면 빌더를 고려하라'
author : 효준
date: 2025-06-27 10:00
tags: [effectivejava, java, oop, design-pattern]
image: /files/covers/blog.jpg
---

# 이펙티브 자바 아이템 2: 생성자에 매개변수가 많다면 빌더를 고려하라

## 핵심 정리
> 생성자나 정적 팩터리가 처리해야 할 매개변수가 많을 때, **빌더 패턴(Builder Pattern)**은 최고의 선택지입니다.

## 1. 매개변수가 많을 때의 문제점

객체를 생성할 때 설정할 필드가 많은 경우, 우리는 보통 두 가지 방법을 사용합니다.

### 가. 점층적 생성자 패턴 (Telescoping Constructor Pattern)

필수 매개변수만 받는 생성자부터 시작해서, 선택적 매개변수를 하나씩 추가한 생성자를 계속해서 만드는 방식입니다.

```java
// 점층적 생성자 패턴
public class NutritionFacts {
    private final int servingSize;  // (mL, 1회 제공량)     필수
    private final int servings;     // (회, 총 n회 제공량)  필수
    private final int calories;     // (1회 제공량당)      선택
    private final int fat;          // (g/1회 제공량)      선택
    private final int sodium;       // (mg/1회 제공량)     선택
    private final int carbohydrate; // (g/1회 제공량)      선택

    public NutritionFacts(int servingSize, int servings) {
        this(servingSize, servings, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories) {
        this(servingSize, servings, calories, 0);
    }
    
    public NutritionFacts(int servingSize, int servings, int calories, int fat) {
        this(servingSize, servings, calories, fat, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium) {
        this(servingSize, servings, calories, fat, sodium, 0);
    }

    public NutritionFacts(int servingSize, int servings, int calories, int fat, int sodium, int carbohydrate) {
        this.servingSize  = servingSize;
        this.servings     = servings;
        this.calories     = calories;
        this.fat          = fat;
        this.sodium       = sodium;
        this.carbohydrate = carbohydrate;
    }
}
```

- **단점:** 매개변수가 많아지면 코드를 작성하거나 읽기 어렵고, 매개변수의 순서를 헷갈려 버그를 유발하기 쉽습니다.

### 나. 자바빈즈 패턴 (JavaBeans Pattern)

매개변수가 없는 생성자로 객체를 만든 후, `setter` 메서드를 호출해 원하는 매개변수의 값을 설정하는 방식입니다.

```java
// 자바빈즈 패턴
public class NutritionFacts {
    private int servingSize = -1; // 필수; 기본값 없음
    private int servings = -1;    // 필수; 기본값 없음
    private int calories = 0;
    private int fat = 0;
    private int sodium = 0;
    private int carbohydrate = 0;

    public NutritionFacts() { }

    // Setters
    public void setServingSize(int val)  { servingSize = val; }
    public void setServings(int val)     { servings = val; }
    public void setCalories(int val)     { calories = val; }
    public void setFat(int val)          { fat = val; }
    public void setSodium(int val)       { sodium = val; }
    public void setCarbohydrate(int val) { carbohydrate = val; }
}
```

- **단점:**
    1.  객체 하나를 만들려면 여러 메서드를 호출해야 합니다.
    2.  객체가 완전히 생성되기 전까지 **일관성(consistency)이 무너진 상태**에 놓입니다.
    3.  **불변(immutable) 클래스**를 만들 수 없습니다.

## 2. 빌더 패턴 (Builder Pattern)

> 점층적 생성자 패턴의 **안전성**과 자바빈즈 패턴의 **가독성**을 겸비한 해결책입니다.

- **작동 방식:**
    1.  필요한 객체를 직접 만드는 대신, 필수 매개변수만으로 생성자를 호출해 **빌더(builder) 객체**를 얻습니다.
    2.  빌더 객체가 제공하는 `setter`와 비슷한 메서드들로 원하는 선택 매개변수들을 설정합니다.
    3.  마지막으로 `build()` 메서드를 호출해 필요한 객체를 얻습니다.

```java
// 빌더 패턴
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    private final int carbohydrate;

    public static class Builder {
        // 필수 매개변수
        private final int servingSize;
        private final int servings;

        // 선택 매개변수 - 기본값으로 초기화
        private int calories      = 0;
        private int fat           = 0;
        private int sodium        = 0;
        private int carbohydrate  = 0;

        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }

        public Builder calories(int val) {
            calories = val;
            return this; // 빌더 자신을 반환하여 연쇄적으로 호출 가능
        }
        public Builder fat(int val) {
            fat = val;
            return this;
        }
        public Builder sodium(int val) {
            sodium = val;
            return this;
        }
        public Builder carbohydrate(int val) {
            carbohydrate = val;
            return this;
        }

        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }

    private NutritionFacts(Builder builder) {
        servingSize  = builder.servingSize;
        servings     = builder.servings;
        calories     = builder.calories;
        fat          = builder.fat;
        sodium       = builder.sodium;
        carbohydrate = builder.carbohydrate;
    }
}
```

- **사용 예시:**
```java
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
    .calories(100)
    .sodium(35)
    .carbohydrate(27)
    .build();
```

### 빌더 패턴의 장점
1.  **가독성이 뛰어나다:** 각 매개변수의 의미를 명확히 알 수 있습니다.
2.  **불변 객체를 만들 수 있다:** `setter`가 없으므로 객체의 일관성이 유지됩니다.
3.  **유연하다:** 여러 매개변수를 유연하게 설정할 수 있으며, 빌더 하나로 여러 객체를 만들 수 있습니다.

## 결론
- 생성자나 정적 팩터리로 처리해야 할 매개변수가 많다면 빌더 패턴을 사용하는 것이 좋습니다.
- 특히 매개변수 4개 이상부터는 빌더 패턴을 도입할 가치가 있습니다.

---

다음 글에서는 아이템 3번(private 생성자나 열거 타입으로 싱글턴임을 보증하라)에 대해 알아보겠습니다.
