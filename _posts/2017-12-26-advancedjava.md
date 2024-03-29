---
layout: post
title: 'Advanced 자바'
author : 효준
date: 2017-12-26 21:36
tags: [java]
image: /files/covers/blog.jpg
---
### advanced 자바
이미 자바를 처음부터 끝까지 한번이상 보았지만 그래도 좀더 단단하게 다지기 위해
자바 문법을 알고있는 사람에게 추천한다.

### 컴퓨터 프로그래밍
 * 하나 이상의 관련된 추상 알고리즘을 특정한 프로그래밍 언어를 이용해 구체적인 컴퓨터 프로그램으로 구현하는 기술

### 추상화
 * 추상화는 복잡한 자료,모듈,시스템등으로부터 핵심적인 개념 또는 기능을 간추려 내는 것

### 객체지향 vs 절차지향

 * 객체지향 프로그래밍은 컴퓨터 프로그램을 명령어의 목록으로 보는 시각에서 벗어나 여러개의 독립된 단위, 즉 객체 들의 모임으로 파악하고자 하는 것이다. 각각의 객체는 메시지를 주고받고, 데이터를 처리할 수 있다.

 * 절차지향 프로그래밍은 때때로 명령형 프로그래밍과 동의어로 쓰이기도 하지만, 프로시저 호출의 개념을 바탕으로 하고 있는 프로그래밍 패러다임을 의미하기도 한다. 프로시저는 루틴,하위프로그램,서브루틴,메서드,함수라고도 하는데, 간단히 말하여 수행되어야 할 연속적인 계산과정을 포함하고 있다. 프로그램의 아무 위치에서나 프로시저를 호출될 수 있는데, 다른 프로시저에서도 호출 가능하고 심지어는 자기자신에서도 호출 가능하다.

컵에 물을따라 마시는 과정
ex) 절차지향 : 컵에 물을 따른다. 컵을 든다. 컵을 입에 대고 마신다.

ex) 객체지향 : 컵에 물을 따라마셔야지. --> 나라는 객체와 컵이라는 객체사이에 관계를 맺어줌.

객체지향과 절차지향중 뭐가 더좋은지는 사용하는 용도에 따라 다름.
예를 들면 과학계산용 계산기를만들려면 객체지향으로만들 경우 오버헤드가 심함.
절차지향으로 만들어서해야 좋음.
하지만 사용자간의 인터랙션이 필요한 어플일 경우 객체지향이 좋다.

절차지향 프로그래밍

```
shape s[100];
int sum = 0;
for(i=0 ; i < 100 ; i++){
	if(s[i] typeof triangle)
	sum += ~~~
	if(s[i] typeof Rectangle)
	sum += ~~~
	if(s[i] typeof circle)
	sum += ~~~
}
```

이런식으로 절차적으로 프로그래밍 하는 것이 절차지향 프로그래밍이다.

반면에 객체지향은

```
abstract class Shape{
	abstaract float calArea();
}

class Triangle extends Shape {
	float calArea(){
	~~~
	}
}

class Rectangle extends Shape {
	float calArea(){
	~~~
	}
}

class Circle extends Shape {
	float calArea(){
	~~~
	}
}

Shape s[100];

for(i = 0 ; i < 100 ; i++)
	sum += s[i].calArea();
```

처럼 객체를 이용해 s의 객체들마다 이름이 같은 메소드이지만 각각 객체마다 다른 기능을하는 메소드를 부르는 것이 객체지향프로그래밍이다.

간단히 말해서 절차지향은 내가 구하고자 하는 값을 구하기위한 과정을 기술하는 것이다.
반면에 객체지향언어라함은 어떤문제를 해결하기위해서 그 문제를 구성하고있는 각각의 객체를 설계하고 , 그 관계를 비교해줌으로써 문제를 해결하는 것이다.

### 객체지향 언어의 특성
1. Abstraction*객체와 프로시저들의 공통의 특질을 골라내는 과정*속성과 메소드를 그룹 짓는다.*기능
2. Information Hiding* Birthday 라는 클래스에 day 라는 속성이 있다. 만약 Information Hiding을 할필요없다면, Birthday bd = new Birthday(); 로 bd라는 객체를 생성할 때 bd.day = 32처럼 올바르지않은값이 들어갈 수 있는데 이걸 방지하기 위해서 한다.
3. Encapsulation* 객체와 기능, 즉 메소드와 데이터등을 위해 필요한 모든 자원을 프로그램 객체내에 포함시키는 것.* 구현된 클래스의 세부사항을 숨긴다.* 사용자가 데이터에 접근하기 위해서는 반드시 인터페이스를 통하도록 한다.* 코드 관리를 쉽게 만들어준다.장점 : 그 클래스의 내부를 몰라도됨. 단점 : 그 기능을 변경하고자할 때 어려움.


### Object vs. Class

현실에 있는 책상,의자,컴퓨터 등등을 Object라 한다.
그 Object들은 그룹핑을 할 수 있다. 
그 그룹핑한것들의 설계도는 같음. 그 설계도를 Class라 한다.
Class는 메모리에 로딩되고 Object는 메모리에 생성된다.(Heap)
Class는 Object의 청사진이다.
제조분야에서 청사진이란 많은 물리적 디바이스들로부터 만들어진 하나의 디바이스에 대한 자세한 설명을 뜻한다.

소프트웨어에서 클래스는 객체에 대한 설명이다.
 *클래스는 객체가 포함하고 있는 Data를 나타낸다.
 *클래스는 객체가 보여주는 기능을 나타낸다.

자바에서 클래스는 OOP의 세가지 특성을 지원한다.
 *캡슐화(encapsulation),상속(inheritance),다형성(polymorphism)

### Field and Method

필드란 클래스에 정의된 값이다. 예를들면 나이, 키, 성별 등등

메서드란 행동을 기술해놓은 것이다. 예를들면 더한다.뺀다 등등

객체라는 것은 혼자 독립적으로 존재하지 않음.

객체는 Is-a vs Has-a 라는 두가지 관계가 있음.

Is-a 관계는 삼각형도 도형, 사각형도 도형

Has-a 관계는 삼각형은 점이라는 객체를 가지고 있다.

객체의 멤버에 접근하는 방법은 
객체를 가리키는 변수.멤버 로 접근가능하다.

상속이란 새로운클래스가 기존의 클래스의 자료와 연산을 이용할 수 있게 하는 기능.
상속을 받는클래스를 자식클래스, 상속을 한 클래스를 부모클래스라고 한다.

다형성이란 많은 다른형태를 가질 수 있는 성질을 말한다.
예를들어 Manager 클래스는 Employee클래스의 성질을 가지고 있다.

오버라이딩 메소드
 상위클래스로부터 상속받은 메소드를 하위 클래스에서 수정할 수 있다.
 조건은 메소드명이 같아야하고, 리턴타입이 같아야하고, 파라미터가 같아야한다.




