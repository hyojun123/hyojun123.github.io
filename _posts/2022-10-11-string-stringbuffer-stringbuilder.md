---
layout: post
title: '[백엔드-면접] String, StringBuffer, StringBuilder의 차이'
author : 효준
date: 2022-10-11 20:00
tags: [spring]
image: /files/covers/blog.jpg
---

# 과연 String, StringBuffer, StringBuilder의 차이가 무엇일까?



> 현재 재직하고 있는 회사에서도 3개다 사용하는 코드가있는데 나는 별 상관없이 사용했다.
항상 개발할때 어떻게하면 자원을 덜 사용할까? 라는 고민을 하면서 개발을 진행하는데
혹시 이것들의 차이가 있을까라는 고민을 시작으로 한번 정리해보려고 한다.


결론부터 말하자만 차이가 당연히 존재한다. 각자 환경에 따라 골라서 쓰길 바란다.
---

# 위 3개 클래스 모두 문자열을 다루는 클래스이다.

-  String
   -  String 의 경우 불변의 속성(Immutable)을 갖는 클래스이다. 
  ```java
  String a = "Hello"; // Hello
  a += " world"; // Hello world
  ```

  - 단순하게 보면 참조변수 a에 처음선언했던 Hello에 world를 합친걸로 착각할 수 있다.
    하지만 실제로는 "Hello World"를 가진 영역을 새로 생성하고 기존 Hello를 가진 영역은 GC대상이 된다.
    이와같이 String은 문자열을 자주 변경할 경우 계속해서 가비지가 생성되고 힙메모리 부족으로 성능에 문제가 생길 수 있는 가능성이 생기게된다.


- StringBuilder, StringBuffer
  - 위 두개의 클래스는 기본적으로 가변성(mutable)을 갖는 클래스이다.
  - 따라서 문자열의 변경이 자주 일어날 경우 위 두 클래스를 고려해야한다.
  - 위 두개의 차이점은 동기화의 유무로써 StringBuffer의 경우 멀티스레딩환경에서 Thread-safe하다.
  - 반면 StringBuilder는 동기화를 고려하지않는다. 멀티스레딩환경이 아니라면 StringBuilder의 성능이 더 좋다.

> 정리

  - String : 문자열 연산이 적고 멀티스레드 환경일 경우
  - StringBuffer : 문자열 연산이 많고 멀티스레드 환경일 경우
  - StringBuilder : 문자열 연산이 많고 단일쓰레드이거나 동기화를 고려하지 않아도 되는 경우

위의 사항을 고려하여 적절하게 사용하면 좋다.


  





