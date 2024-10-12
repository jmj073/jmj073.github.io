---
layout: post
title:  "Continuation과 call/cc"
date:   2024-10-12 15:35:15 +0900
categories: language concept
typora-root-url: ./..
on_hold: false
use_math: false
---

{% comment %}

[toc]

{% endcomment %}

이 글은 continuation 시리즈의 첫 글이다. 다음과 같은 순서로 이 시리즈를 진행해볼 생각이다.

1. continuation과 `call/cc`
2. delimited continuation
3. continuation passing style(CPS)
4. continuation의 응용

그럼 continuation과 `call/cc`. 시작하겠다.

# 들어가기에 앞서...

이 글에서는 의사 코드가 나온다. scheme 문법을 사용할까 했지만 대부분의 사람들이 이에 익숙하지 않을 것이므로 필자가 임의로 만든 의사 코드?를 사용하겠다.

이 글에서 나오는 의사 코드의 특징:
+ program은 0개 이상의 statement의 나열로 이루어진다. 예시는 다음과 같다.

  ```javascript
  let a = 3 + 4; // statement1
  for (let i = 0; i < 10; ++i) { // statement2
      println(i);
  }
  let b = a + 6; // statement3
  ```

+ rust 언어와 같이 compound expression(복합 표현식)을 사용한다. compound expression의 예시(rust 코드)는 다음과 같다. 자주 등장하므로 꼭 이해하고 넘어가자.

  ```rust
  // rust에서는 중괄호안의 statement 나열에서 마지막에 ;를 붙이지 않으면 해당 표현식이 중괄호 전체 표현식의 값이 된다.
  // 이는 함수에도 적용된다. 함수에서는 return 값으로 간주된다.
  let x = {
      let y = 10;
      let z = 20;
      y + z  // 복합 표현식의 마지막 줄이 전체 표현식의 값이 됨
  };
  println!("{}", x);  // 출력: 30
  let a = if x != 30 { 1 } else { let b = 6; b + 4 }; // a는 10
  fn foo() -> i32 { 3 * 11 } // 33을 반환하는 함수
  ```

# Continuation이란?

continuation을 단어로서 보자면 "이어지는 무언가"란 의미를 갖는다고 볼 수 있겠다. 이 글의 주제가 되는 "continuation"이란 것도 비슷한 맥락을 가지는 용어라고 볼 수 있다.

예를 들어 보자. 아래의 예에서 `instruction2 instruction3 ...`는 현재 명령인 `instruction1`의 뒤로 이어지는(continuation) 흐름으로 볼 수 있다. 이를 continuation이라 부른다. 즉, `instruction1`을 기준으로 `instruction2 instruction3 ...`가 continuation이 되는 것이다.

``` 
instruction0
instruction1 <- 현재 명령
instruction2
instruction3
...
```

이게 끝이다.

> [!info]
> 참고로, continuation을 "후속문"이라고 번역한 글도 있다. [여기](https://guruma.github.io/posts/2018-11-18-Continuation-Concept/)에서 볼 수 있다.

# Continuation의 사용 예

구체적으로 continuation이 어떤 기능을 가져다 줄지 알아보자.

프로그래밍 언어에서는 실행 흐름에 대한 제어(즉, 이어서 실행될 것(continuation)에 대한 제어)를 제공하기 위해 if, for, while 등의 제어문을 제공한다. 이런 제어문들은 언어 차원에서 기본적으로 제공해 주는 것이지만, 만약 continuation이 지원된다면 여러가지 제어 흐름을 프로그래머가 직접 만들어 사용할 수 있다.

예를 들어 이중 반복문을 실행하다가 빠져나가는데 사용될 수도 있다. 즉, 다음에 실행될 것(continuation)을 반복문의 다음 반복이 아닌 반복문 후에 실행될 것(continuation)으로 바꾸면 된다. 이 뿐만 아니라 예외(exception)나, 코루틴(coroutine) 같은 제어 요소를 구현하는 데에도 사용할 수 있다.

# Scheme에서의 Continuation

[first-class](https://ko.wikipedia.org/wiki/%EC%9D%BC%EA%B8%89_%EA%B0%9D%EC%B2%B4) continuation을 처음으로 지원한 언어는 scheme이다. [The Scheme Programming Language](https://www.scheme.com/tspl4/further.html#./further:h3)라는 책은 continuation에 대해 어떻게 설명했는지 한번 살펴보자.

## Scheme의 문법

일단 본격적으로 설명하기 전에 scheme의 문법에 대해 짚고 넘어가 보자.

scheme은 lisp(list processing) 프로그래밍 언어의 방언이다. lisp 언어는 괄호쌍으로 list의 원소들을 묶어서 [s-expression](https://en.wikipedia.org/wiki/S-expression)을 표현하는 방식을 사용한다. 아래는 그 예시이다:

+ **일반적인? 프로그래밍 언어**: `print(4 * 3)`
+ **lisp**: `(print (* 4 3))`

참고로 scheme의 문법은 다음과 같다.

```
<program>               -> <form>*
<form>                  -> <definition> | <expression>
<definition>            -> <variable definition> | (begin <definition>*)
<variable definition>   -> (define <variable> <expression>)
<expression>            -> <constant>
                        |   <variable>
                        |   (quote <datum>)
                        |   (lambda <formals> <expression> <expression>*)
                        |   (if <expression> <expression> <expression>)
                        |   (set! <variable> <expression>)
                        |   <application>
<constant>              -> <boolean> | <number> | <character> | <string>
<formals>               -> <variable>
                        |   (<variable>*)
                        |   (<variable> <variable>* . <variable>)
<application>           -> (<expression> <expression>*)
```

> [!note]
> 혹시 위의 문법이랍시고 적어놓은게 뭔지 잘 모르겠다면 [여기](https://ko.wikipedia.org/wiki/%EB%B0%B0%EC%BB%A4%EC%8A%A4-%EB%82%98%EC%9A%B0%EB%A5%B4_%ED%91%9C%EA%B8%B0%EB%B2%95)를 참고하자.

> [!note]
> scheme은 function 대신 procedure를, call(함수 호출) 대신 apply(프로시저 적용)라는 용어를 사용하는 경향이 있다. 위의 문법에서 `<application>`은 함수 호출이라 보면 된다.

> [!info]
> 기본 문법, 즉,  core syntactic form(예: lambda, if)은 몇개 없지만, scheme은 syntactic extension을 정의할 수 있으며, syntactic extension은 한번 정의되고 나면, core form과 정확히 동일한 상태를 갖는다.

> [!info]
> core syntactic form에 반복에 관한 것이 없다는 것을 알 수 있다. scheme은 반복을 표현하기 위해 tail recursion(꼬리 재귀)를 사용하며, scheme의 구현체는 tail-call optimization을 수행해야 한다.

## 설명!

scheme expression을 evaluate[^eval]하는 동안, 구현체(scheme의 구현체)는 두 가지를 추적해야 한다.

1. what to evaluate(무엇을 평가할지, 평가할 대상)
2. what to do with the value(그 값으로 무엇을 할지, 값으로 할 행위)

아래의 expression에서 `(null? x)`의 evaluation을 고려해 보자.

```scheme
; (if condition-expr then-expr else-expr)라고 보면 됨.
; 예를 들어, (if true 3 4)는 3이 된다.
(if (null? x) (quote ()) (cdr x))
```

구현체는 첫번째로 `(null? x)`를 evaluate해야 한다. 그리고, 그 값에 따라 `(quote ())` 또는 `(cdr x)`를 evaluate해야 한다. 여기서 "what to evaluate"는 `(null? x)`이며, "what to do with value"는 `(quote ())`와 `(cdr x)` 중 어느 것을 evaluate할지 결정하고 실제로 evaluate하는 것이다. "what to do with the value"를 "계산(computation)의 continuation"이라 부른다. 즉, 이 예에서는 "**`(null? x)`의 continuation**"과 같은 말을 사용할 수 있을 것이다.

expression을 evaluate하는 동안 어느 시점에서든, 해당 지점에서 계산을 완료하거나 적어도 계속할(continue) 준비가 된 continuation이 있다.

`x`가 `(a b c)` 값을 가지고 있다고 가정하자. 그러면 `(if (null? x) (quote ()) (cdr x))`의 evaluate 도중 다음 여섯 가지 continuation을 분리할 수 있다. 각 continuation들은 다음을 기다린다.

1. `(if (null? x) (quote ()) (cdr x))`의 값
2. `(null? x)`의 값
3. `null?`의 값
4. `x`의 값
5. `cdr`의 값
6. `x`의 값

> [!note]
> `(cdr x)`의 continuation은 `(if (null? x) (quote ()) (cdr x))`의 continuation과 같기 때문에 위에 나열되지 않았다.

# 언어에서의 Continuation 지원

## `call-with-current-continuation`(`call/cc`)

scheme은 first-class continuation의 지원을 위해 `call-with-current-continuation`(대부분의 구현체에서 `call/cc`라는 별칭을 가짐)라는 함수를 제공한다. 참고로 scheme은 함수 이름에 `-`나 `/`를 사용할 수 있다.

`call-with-current-continuation`이라는 이름에서도 알 수 있듯이, `call/cc`는 함수를 인수로 받아 해당 함수에 continuation을 인수로 넘겨서 호출한다. 예를 들어서 설명해 보겠다.

+ `call-with-current-continuation`의 시그니쳐(`call_cc`라는 이름을 사용하겠다):
  ```javascript
  call_cc(func)
  ```
  
+ `func`의 시그니쳐: 아래에서 `k`는 continuation이며, `k`는 `call_cc(func)`의 continuation이다.
  ```javascript
  func(k)
  ```

+ `k`의 시그니쳐: continuation `k`는 함수로서 표현된다. `k`에 인수로 `value`를 넘겨서 호출하면 `value`의 continuation이 `k`로 교체된다. `k`가 호출되지 않고 `func`가 반환되면 `call_cc(func)`의 값은 `func(k)`의 값이 된다.
  
  ```javascript
  k(value)
  ```

예시:

```javascript
call_cc(function(k) { 5 * 4 })
=> 20
call_cc(function(k) { 5 * k(4) })
=> 4
2 + call_cc(function(k) { 5 * k(4) })
=> 6
```

+ `cc` & `identity`: 앞으로의 설명에서 `cc`와 `identity`라는 함수를 사용하겠다. `cc` 함수는 이름 그대로 `current-continuation`을 반환하는 함수이다. 함수의 정의는 다음과 같다.
  
  ```javascript
  let identity = function(v) { v };
  let cc = function() { call_cc(identity) };
  ```

## `call/cc` Continuation의 범위

continuation의 범위에 대해 먼저 설명해야 할지, 아니면 `call/cc`의 사용 예에 대해 먼저 설명해야 할지 고민이었지만, 범위먼저 설명하기로 한 이유는 범위에 대해 이해해야지 예시의 동작을 명확하게 할 수 있을것이라 생각했기 때문이다. 하지만 예시에 대해 알아보는 것도 범위에 대해 이해하는데에 도움이 될 것이라 생각하기 때문에, 아직 `call/cc`의 사용 방법에 대해 감이 잘 잡히지 않는다면 예시를 먼저 보고오는 것도 좋을 것이다.

위에서 아래와 같은 언급을 했었다.
```
expression을 evaluate하는 동안 어느 시점에서든, 해당 지점에서 계산을 완료하거나 적어도 계속할(continue) 준비가 된 continuation이 있다.
```

이 문장에서 "계산을 완료할 continuation"이 어느 부분인지를 살펴볼 것이다. "계산을 완료할 continuation"이 어느 부분인지 안다는 것은 어떤 continuation의 끝이 어디인지 안다는 것이다. 재귀 함수의 base case와 비슷한 맥락이라고 생각하면 된다.

필자의 생각으론 scheme에서 continuation의 범위는 `<form>`이다. `<form>`에는 `<definition>`도 포함되지만, 그부분은 신경쓰지 말자... 어쨌든 `<form>`의 continuation이 "계산을 완료할 continuation"이 되는 것이다.

그래서, 아래의 예시에서는:

```scheme
; 참고로 이 예시는 scheme의 구현중 하나인 chicken scheme에서 실행되었다
(define k (cc)); 변수 k를 정의하고 (cc)의 값으로 초기화. cc(current-continuation)는 위에서 언급된 함수이다.
(print k); 변수 k를 출력.
(k 3); 인수를 3으로 k를 호출.
(print "hi " k); "hi "와 변수 k를 출력.
```

다음과 같은 결과가 나온다.

```scheme
#<procedure (continuation . results)>; 2번 줄
hi 3; 4번 줄
```

즉, continuation의 범위는 `<program>`이 아니기 때문에 다음과 같은 결과는 나오지 않는다.

```scheme
#<procedure (continuation . results)>; 2번 줄
3; 2번 줄
Error: call of non-procedure: 3; 3번 줄
```

continuation의 범위를 `<program>`으로 할 수 있을까?라는 의문이 들지도 모르겠지만, 일단 그전에 타임머신을 만들 수 있을지를 먼저 따져봐야 할 것이다. 왜냐하면 lisp는 [read-eval-print loop](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93print_loop)(REPL)를 제공하기 때문이다. continuation의 범위가 `<program>`이라면 특정 continuation 오브젝트의 호출 시점에 전체 프로그램을 알아야 하는데 타임머신을 사용하는 것 외에는 알길이 있을 것 같지는 않다.

아무튼 scheme의 continuation의 범위는 `<form>`이고, 필자는 의사 코드의 continuation의 범위를 정할 필요가 있다. 앞서 의사코드 program은 0개 이상의 statement의 나열로 이루어진다고 했으므로, 각각의 최상위 statement(다른 statement에 포함되지 않는 statement)를 continuation의 범위로 정하겠다.

## `call/cc`를 사용해보자!

여기서는 `call/cc`의 사용 예에 대해 좀더 다루겠다. 간단한 예시들을 다룰 것이며, coroutine 같은 것은 "continuation의 응용"에서 다룰 것이다. 참고로 예시들중에 몇몇은 [The Scheme Programming Language](https://www.scheme.com/tspl3/further.html#./further:h3)와 [여기](https://guruma.github.io/posts/2018-11-18-Continuation-Concept/#_%ED%9B%84%EC%86%8D%EB%AC%B8_%EC%9D%B4%EB%94%94%EC%97%84)를 참고하였다. 의사 코드는 실행이 불가능하니 직접 실행해 보고 싶다면 두 글의 scheme 코드를 실행해 보자.

+ **탈출 만들기**: 다음은 `it` iterator의 각 숫자를 곱하여 결과를 반환하는 함수이다. 원소에 `0`이 있으면 항상 결과가 `0`이 되므로, 그 뒤를 계산할 필요가 없기 때문에 continuation을 사용하여 `0`을 바로 반환한다.
  
    ```javascript
    let product = function(it) {
        call_cc(function(k) {
            let f = function(it) {
                if (it.end()) return 1;
                let cur = it.next();
                if (cur == 0) k(0);
                cur * f(it)
            };
            f(it)
        })
    };
    ```
    
+ **"hi"가 두개**:
    ```javascript
    {
        let x = cc();
        x(function(ignore) { "hi" })
    }
    => "hi"
    ```
    해석:
    ```javascript
    let x = cc();
    x(function(ignore) { "hi" })
    // ===================================================
    let x = function(ignore) { "hi" };
    x(function(ignore) { "hi" })
    // ===================================================
    (function(ignore) { "hi" })(function(ignore) { "hi" })
    // ===================================================
    "hi"
    ```
    
+ **ii**: (scheme으로 볼 때는 복잡해 보였는데 `cc`와 `identity`로 인해 간단해보인다...)
    ```javascript
    cc()(identity)("HEY!")
    => "HEY!"
    ```
    해석:
    ```javascript
    cc()(identity)("HEY!")
    identity(identity)("HEY!")
    identity("HEY!")
    "HEY!"
    ```
    
+ **retry**:
    ```javascript
    let retry;
    let factorial = function(n) {
        if (n == 0) {
            call_cc(function(k) { retry = k; 1 })
        } else {
            n * factorial(n - 1)
        }
    };
                     n:   4    3    2    1    0
    factorial(4); => 24  (4 * (3 * (2 * (1 * (1)))))
    retry(1);     => 24  (4 * (3 * (2 * (1 * (1)))))
    retry(2);     => 48  (4 * (3 * (2 * (1 * (2)))))
    retry(5);     => 120 (4 * (3 * (2 * (1 * (5)))))
    ```

+ **변수 참조 1**: "변수 참조 1"과 "변수 참조 2"의 출력 결과에 차이가 있는 이유는 1은 변수가 다른 것을 가리키게 바꾼 것이고, 2는 변수가 가리키는 오브젝트 자체를 수정한 것이기 때문이다.
  
    ```javascript
    {
        let x = true;
        let k = cc();
        
        if (x) {
            x = false;
            println("true");
            k(function(ignore) { println("hi"); });
        } else {
            println("false");
        };
    };
    ```
    output:
    ```
    true
    true
    hi
    ```
    
+ **변수 참조 2**:
    ```javascript
    {
        let x = [true];
        let k = cc();
        
        if (x[0]) {
            x[0] = false;
            println("true");
            k(function(ignore) { println("hi"); });
        } else {
            println("false");
        };
    };
    ```
    output:
    ```
    true
    false
    ```
    
+ **evaluation 순서**: `k`를 호출했음에도 불구하고 `hi`가 한번 출력되는 이유는 `cc()`보다 `println("hi")`의 evaluation이 먼저 일어나기 때문이다.
  
    ```javascript
    let k = { println("hi"); identity }(cc());
    k("dummy");
    ```
    output:
    ```
    hi
    ```

### 음양

예시중 하나인 음양에 대한 설명은 길어질 것 같으므로 섹션을 따로 만들었다. 일단 음양(yin-yang)의 코드는 다음과 같다.

```javascript
{
    let yin  = (function(k) { print("@"); k })(cc());
    let yang = (function(k) { print("*"); k })(cc());
    yin(yang);
};
```

위 코드는 scheme으로 되어 있던 것을 의사 코드로 옮긴 것이다. 사실 아래 코드와 출력 결과가 다를게 없지만, 위 코드가 더 있어보인다.

```javascript
{
    let yin = cc();
    print("@");
    let yang = cc();
    print("*");
    yin(yang);
};
```

코드의 출력은 아래와 같다. `*`이 하나씩 늘어나면서 출력이 계속된다.

```
@*@**@***@****@*****@******@*******@********@*********@**********@ ...
```

`cc`의 호출이 두 군데이므로, 둘을 구분하기 위해 설명에서는 아래와 같은 코드를 사용하겠다.

```javascript
let a = cc;
let b = cc;
{
    let yin  = (function(k) { print("@"); k })(a());
    let yang = (function(k) { print("*"); k })(b());
    yin(yang);
};
```

일단 다음과 같은 정의를 해보자. TODO

```
A = `a()`의 continuation
B(0) = `yin`의 값이        A인 `b()`의 continuation
B(n) = `yin`의 값이 B(n - 1)인 `b()`의 continuation (n > 0)
YIN(n)    = `yin`의 호출 횟수가 n일때 `yin`의 값 (n >= 0)
YANG(n)   = `yin`의 호출 횟수가 n일때 `yang`의 값 (n >= 0)
OUTPUT(n) = `yin`의 호출 횟수가 n일때 출력 (n >= 0)
```

위의 정의에 의하면 다음과 같은 표를 만들 수 있다.

| N    | YIN(N) | YANG(N) | OUTPUT(N) |
| ---- | ------ | ------- | --------- |
| 0    | A      | B(0)    | `@*`      |

이제 N > 0일 때를 채워나가야 한다. 이를 위해서 다음과 같은 규칙을 세울 수 있다.

```
YIN(0) = A
YIN(n) =
    if YIN(n - 1) == A: YANG(n - 1)
    else: YIN(n - 1) continuation의 `yin` 값
    (n > 0)

YANG(0) = B(0)
YANG(n) =
    if YIN(n - 1) == A: `yin`의 값이 YANG(n - 1)(= YIN(n))인 continuation
    else: YANG(n - 1)
    (n > 0)

OUTPUT(0) = "@*"
OUTPUT(n) =
    if yin(n - 1) == A: "@*"
    else: "*"
    (n > 0)
```

위의 규칙에 따라 표를 채워 나가보자.

| N    | YIN(N) | YANG(N) | OUTPUT(N) |
| ---- | ------ | ------- | --------- |
| 0    | A      | B(0)    | `@*`      |
| 1    | B(0)   | B(1)    | `@*`      |
| 2    | A      | B(1)    | `*`       |
| 3    | B(1)   | B(2)    | `@*`      |
| 4    | B(0)   | B(2)    | `*`       |
| 5    | A      | B(2)    | `*`       |
| 6    | B(2)   | B(3)    | `@*`      |
| 7    | B(1)   | B(3)    | `*`       |
| 8    | B(0)   | B(3)    | `*`       |
| 9    | A      | B(3)    | `*`       |
| 10   | B(3)   | B(4)    | `@*`      |

`YIN(-1)`은 정의되지 않았지만 `OUTPUT(0)`이 `@*`라는 것은 유추할 수 있다. 하지만 필자는 예외를 좋아하지 않기 때문에, `YIN(0)`과 `YANG(0)`에 충족되는 `YIN(-1)`과 `YANG(-1)`를 정의해 보겠다.

| N    | YIN(N) | YANG(N) | OUTPUT(N) |
| ---- | ------ | ------- | --------- |
| -1   | A      | A       |           |
| -1   | B(0)   | B(0)    |           |
| 0    | A      | B(0)    | ???       |

예상치 않게 두가지 경우가 나왔지만 `OUTPUT(0)`이 `@*`이 되게 하는 것은 `YIN(-1)` = `A`, `YANG(-1)` = `A`이므로, 이를 `N`이 -1일 때로 간주하자.

음양은 더 다뤄볼까 했으나 더하다간 음양오행(陰陽五行)까지 나올 것 같으므로, 음양은 언젠가 글을 한번 따로 써보도록 하겠다.

# 나오기에 앞서...

사실 글을 시리즈로 나누지 않고 하나로 적으려 했지만 생각보다 글이 길어지고, 자료조사와 글을 작성하는 일이 매우 귀찮은 일이라는 것을 깨달았기 때문에 다음으로 미루기로 했다. continuation을 "계속할 준비가 된 continuation"에서, "계산을 완료할 continuation"으로 바꿨을 뿐이다. 이 `<program>`은 `<form>`이 하나이지는 않을 것이다. 하지만 필자는 타임머신이 없기 때문에 확답은 못한다.

글을 적으면서 약간 아쉽기도 했다. 이론에 대해 아는게 적다보니 글의 깊이가 얕아지는 듯 하다. 그래도 공부하면서 점차 글을 보완해나갈 생각이다. 어쨌든 다음은 "delimited continuation"이다. 사실 필자는 이 글을 적는 현 시점에 delimited continuation에 대해 아는 것이 없으므로 공부한다음에 다음 편을 내보도록 하겠다.

# Reference

+ [후속문(Continuation) : 제1부. 개념과 call/cc](https://guruma.github.io/posts/2018-11-18-Continuation-Concept/)
+ [the scheme programming language: continuation](https://www.scheme.com/tspl3/further.html#./further:h3)[^TSPL]
+ **scheme wiki**:
  + [call-with-current-continuation](http://community.schemewiki.org/?call-with-current-continuation)
+ **wikipedia**:
  + [Continuation](https://en.wikipedia.org/wiki/Continuation)
  + [call-with-current-continuation](https://en.wikipedia.org/wiki/Call-with-current-continuation)
  + [Scheme (programming language)](https://en.wikipedia.org/wiki/Scheme_(programming_language))
  + [S-expression](https://en.wikipedia.org/wiki/S-expression)

---

[^eval]: expression을 계산하여 값(value)으로 만드는 행위
[^TSPL]: 참고로 이 링크의 책은 제3판이며, 읽고 싶다면 3판 대신 최신판(현재 제4판)을 읽는것을 추천한다.
