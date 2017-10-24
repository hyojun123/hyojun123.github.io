---
layout: post
title: "JavaScript 기본문법"
date: 2017-10-21 16:25:06
tags: javascript
description: 기본 문법
---

<br>

<b>식별자의 규칙</b><br/>
식별자는 자바스크립트에서 이름을 붙일때 사용하는 단어로 <b>변수명,함수명</b> 이 있다.<br/>
식별자의 규칙으로는
<li>키워드를 사용하면 안된다.</li>
<li>숫자로 시작하면 안된다.</li>
<li>특수문자는 언더바와 달러만 허용된다.</li>
<li>공백문자를 포함할 수 없다.</li>
<br/><br/><br/>
예를들면 아래의 4개는 모두 사용 가능하다.


~~~html
alpha
alpha10
_alpha
$alpha
~~~

반면에 숫자로 시작하거나 키워드 또는 공백 문자가 있으면 안된다.
다음은 식별자로 사용할 수 없는 단어이다.

~~~html
break
273alpha
has space
~~~

식별자를 생성할 때에는 몇가지 약속이 있다.
<ul>
<li>생성자 함수의 이름은 항상 대문자로 시작한다.</li>
<li>변수와 인스턴스,함수,메소드의 이름은 항상 소문자로 시작한다.</li>
<li>여러 단어로 이루어진 식별자는 각 단어의 첫글자를 대문자로 한다.</li> </ul>
<br/><br/>


| 구분 | 단독으로 사용 | 다른 식별자와 사용 |
| :--- | :---: | :---: |
| 식별자 뒤에 괄호 없음 | 변수 | 속성 |
| 식별자 뒤에 괄호 있음 | 함수 | 메서드 |
<br/><br/>
자바스크립트를 아직 배우지 않았어도 이러한 특징으로 네가지를 구분할 수 있다.
<br/><br/>
<ul>
<li>alert('Hello World') -> 함수</li>
<li>Array.length -> 속성</li>
<li>input -> 변수</li>
<li>prompt('Message','Defstr') -> 함수</li>
<li>Math.PI -> 속성</li>
<li>Math.abs(-273) -> 메서드</li>

</ul>
<br/><br/>
<b>주석</b>
주석은 프로그램 코드를 설명하는데 사용하며, 프로그램을 진행하는데 전혀 영향을 주지 않는다. 주석을 사용하는 습관을 기르자.
HTML페이지에는 크게 HTML 태그 주석과 자바스크립트 주석이 있다.
다음은 HTML 페이지에 주석을 사용한 예제이다.<br/><br/>

~~~HTML
<html>
<head>
  <!-- 주석입니다 -->
  <script>
  </script>
</head>
<body>
  <!-- <h1>주석입니다</h1> -->
</body>
<html>  
~~~

자바스크립트는 두 가지 방법으로 주석을 만든다. 첫번째 방법은 // 를 입력하는 것으로 한줄 주석을 표현한다. // 뒤에 문장은 실행되지 않는다.<br/>
그리고 두번째 방법은 /\*  \*/를 이용하여 이 블록내의 문장은 실행되지 않는다.<br/><br/>


~~~javascript
//주석문
/*
  주석문
  주석문
*/
~~~


<b>출력</b>
이제 본격적으로 자바스크립트를 공부해보자.
다른태그를 다루지 않는이상 script 태그만 다루도록 하겠다.
자바스크립트의 가장 기본적인 출력 방법은 <b>alert()</b> 함수를 사용하는 것이다.<br/>
함수의 괄호안에는 문자열을 입력한다.<br/><br/>

~~~html
<script>
  alert("Hello World!");
</script>
~~~

함수 안에 들어가는 것은 매개변수라고 부른다.<br/>
이것을 저장하고 실행하면 경고창에 Hello World!가 뜨는것을 볼 수 있다.<br/><br/>

<img src="{{ site.baseurl }}/images/171021/1.png" alt="Post Sample Image">

자바와는 다르게 자바스크립트에서의 변수선언은 var 로 하면 된다.<br/><br/>


~~~html
<script>
  var a = "Hello world";
  var add = '+';
  var trueVal = 273>53;
  var falseVal = 273<53;
  var num = 23333;
</script>
~~~


<b>undefined 자료형</b><br/><br/>
선언하지 않은 변수 또는 변수를 선언했지만 초기화 하지 않았을 경우 해당 변수의 자료형은<br/>
undefined이다.

~~~html

<br/><br/>
  alert(typeof(variable));
//위 코드를 실행하면 undefined가 뜬다. 이 자료형은 굉장히 많이 활용 된다.
//지금은 모르겠지만 중반부터 중요한 역할을 차지하므로 기억하자.

  var input = prompt('Message','DefStr');
  alert(input);
  //실행결과는 Message가 고정되있고 DefStr이 적혀있는 prompt창이 열린다. prompt창은 사용자에게 입력받는 창이다.
  //문자열 자료형을 입력할 때 사용한다.

  var input = confirm('수락하시겠습니까?');
  alert(input);
  //실행결과는 수락하시겠습니까? 라는 confirm 창이 열린다. confirm창은 예,아니오 버튼이 있다.
  // 예를 누르면 true, 아니오 누르면 false를 리턴한다.

  변수 재선언을 하게되면 기존에 함수 이름으로 사용하던 식별자인 alert를 변수로 재선언 했으므로 코드에 문제가 생긴다.
  <script>
    var alert='red Alert';
    alert(alert);

  </script>
  //위 코드를 실행하면 경고창이 뜨지 않는다. alert함수를 'red Alert'로 재선언 했기 때문이다.

~~~

<br/><br/>
<b>자료형 검사</b>
자바스크립트는 숫자,문자열,불 같은 자료형을 확인할 때 다음 코드처럼 typeof연산자를 사용한다.
<br/><br/>

~~~html
  <script>
    alert(typeof('String'));
    alert(typeof(273));
  </script>
~~~


<br/><br/>
변수 선언
ECMAScript6 에서 추가된 템플릿문자열, let키워드와 const 키워드를 보자.
이것은 인터넷 익스플로어에서는 동작하지 않는다. 다른 웹 브라우저를 사용해 결과를 확인하자.

~~~html
<script>
  alert('표현식 273+53의 값은 '+(273+52)+'입니다.');

  alert('표현식 273+53의 값은 +${52 + 273} 입니다.');
  //위 두개의 코드의 결과값은 정확하게 일치함.

  var variable = 273;
  alert('변수 variable의 값은 ${variable}입니다.');

  // 위와 같이 변수를 넣어서 사용할 수 도 있다.
  //굉장히 편리한 기능이지만 인터넷익스플로어에서는 사용이 불가능 하다.
</script>
~~~

<br/><br/>
<b>let 키워드와 const 키워드</b>
이번 내용은 초보자에게 어려울 수 있다. 이해가 안갈경우 그냥 넘어간 후 다시 보면된다.
<br/><br/>

~~~html
  var variableA = 52;
  let variableB = 237;
  const constantC = 103;
  alert(variableA);
  alert(variableB);
  alert(constantC);

//이것들은 다음과 같은 차이가 있다.
~~~

<table style="text-align:center;">
<tr>
  <td>키워드</td>
  <td>구분</td>
  <td>선언위치</td>
  <td>재선언</td>
</tr>
<tr>
  <td>var</td>
  <td>변수</td>
  <td>전역 스코프</td>
  <td>가능</td>
</tr>
<tr>
  <td>let</td>
  <td>변수</td>
  <td>해당 스코프</td>
  <td>불가능</td>
</tr>
<tr>
  <td>const</td>
  <td>상수</td>
  <td>해당 스코프</td>
  <td>불가능</td>
</tr>

</table>
<br/><br/>

~~~html
  let variable = 273;
  variable = 52; // 가능함

  const constant = 273;
  constant = 52; // 불가능함.




  const constant = 273; //가능
  const constant; //불가능
  //const 는 선언할때 값을 반드시 넣어줘야 한다. 안넣으면 오류생김.



~~~

<br/><br/>
<b>var키워드로 선언한 변수와 let키워드로 선언한 변수의 차이</b>
<br/><br/>
특정 변수를 사용할 수 있는 유효 범위를 '스코프'라고 한다. 조건문,반복문,대괄호 등으로<br/>
만들어지는데,다음 코드에서 대괄호로 감싼 부분이 스코프이다. 스코프를 찾아보면 <br/>
'가장 외곽에 있는 코드를 입력 할 수 있는 부분'이라는 전역스코프를 포함해 3개의 스코프가 있다.<br/>

~~~html
//전역스코프
  {
    //스코프A
    {
      //스코프B
    }

  }
~~~

<br/><br/>
일반적인 프로그래밍 언어에서는 "특정 스코프 안에서 선언한 변수는 해당 스코프안에서만 사용 할 수있다" 라는 <br> 규칙이 있다. 따라서 다른 프로그래밍 언어에서 다음코드처럼 작성하면 오류가 발생한다.

~~~html
  {
    //스코프A
    var variable = 273;
  }

  {
    //스코프B
    alert(variable);
  }
  //전역 스코프
  alert(variable);
~~~


<br/><br/>
하지만 자바스크립트에서 코드를 실행하면 에러가 발생하지 않는다. 자바스크립트의 var키워드는 <br/>
전역스코프 위치에 변수를 선언하는 키워드이기 때문이다.

let으로 선언한 변수는 위의 코드처럼 하면 오류가 난다.<br/>
그럼에도 let을 쓰는 이유는 var 키워드는 그 특성때문에 다양한 실수를 유발할 수 있다.
