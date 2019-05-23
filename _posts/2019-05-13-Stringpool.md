---
layout: post
title: 'Java에서 String 클래스'
author : 효준
date: 2019-05-23 13:00
tags: [java]
image: /files/covers/blog.jpg
---

# String

우리가 일반적으로 쓰는 문자열객체이다.

# String pool

Java 디자이너는 설꼐시 모든 종류의 Java 어플리케이션에서 가장 많이 사용 되는 데이터타입이  String이 될 것이라 예측했었고,<br> 그리고 그것이 처음부터
최적화가 필요한 이유라는 것을 알았다.
첫번째 아이디어는 String pool에 String 리터럴을 포함하는 것이었다.<br>
목표는 String 객체를 공유해, 템프러리하게 생성된 String 객체를 줄여주는 것이었다. 공유를 하기 위해서는 String 클래스는 Immutable Class 여야 한다.
서로 알 수 없는 두 영역에서 mutable 객체의 공유는 불가능하다. 예를 가정해보면, 두 참조 변수가 동일한 String 객체를 참조한다.

``` java
String s1 = "Java";
String s2 = "Java";
```

지금 s1을 "Java"에서 "C++"객체로 변경하면 참조 변수 s2="C++"이 된다.
String을 immutable하게 하는 것으로, String 리터러의 공유가 가능하게 된다.<br>
즉, String pool의 주요한 아이디어는 String을 final 또는 immutable하게 해야만 Java에서 String pool을 구현할 수 있다.
