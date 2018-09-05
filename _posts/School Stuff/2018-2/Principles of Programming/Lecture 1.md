---

layout : single
title : "Principle Of Programming - Call by value vs. Call by name"
categories : [School Stuff - 2018/2]
tags : [Principles Of Programming]
comment : true

---

### '프로그래밍 원리' 수업에서 유용한 포인트들을 발췌해서 기록하는 글입니다.

---

There is two strategy to evaluate values.

#### Call-by-value

- Evaluate the arguments first, then apply the function to them

#### Call-by-name

- Just apply the function to its arguments, without evaluating them

Let's see the example.

~~~
def square (x: Int) = x * x
~~~

[cbv] square(1+1) ~ square(2) ~ 2 * 2 ~ 4

[cbn] square(1+1) ~ (1+1) * (1+1) ~ 2 * (1+1) ~ 2 * 2 ~ 4

So what's the difference?

**Call-by-value** evaluates arguments once, on the other hand, **Call-by-name** DO NOT evaluate unused arguments.

So if there is an argument could not be used depending on conditions, then Call-by-name could be more efficient.

~~~
def f(x : Double, y: Double) = 
	if x 
		0
	else 
		y
~~~

In this case, Call-by-value compute both x and y, but Call-by-name may not compute y!

---

Scala's evaluation strategy

#### Call-by-value

- By default

#### Call-by-name

- Use "=>"

~~~
def loop:Int = loop	// Just a normal, boring loop :)

def one(x: Int, y: => Int) = 1

one(1+2, loop) 	// does not compute loop, therefore result is 1

one(loop, 1+2)	// it tries to compute loop, falls into divergence and never ends.
~~~

---

Scala's name binding strategy

#### Call-by-value

- Use "val" (also called "field")

- Evaluate the expressiong first, then bind the name to it

#### Call-by-name

- Use "def" (also called "method")

- Just bind the name to the expression, without evaluating

~~~
def a = 1 + 2 + 3
val a = 1 + 2 + 3 	// a is now just 6
def b = loop
def b = loop		// it will loop forever, because it tries to compute b. Remember, functions are not meant to be evaluated when it is defined!

def f(a: Int, b: Int): Int = a * b -2
~~~

---


#### Exercise: square root calculation

- Calculate square roots with Newton's method

~~~
def isGoodEnough(guess: Double, x: Double) =
	??? // guess*guess is 99.9% close to x

def improve(guess: Double, x: Double) =
	(guess + x/guess) / 2

def sqrtIter(guess: Double, x: Double): Double =
	??? // repeat improving guess until it is good enough

def sqrt(x: Double) =
	sqrtIter(1, x)

sqrt(2)
~~~

<br/>

- Answer : 

~~~
object HelloWorld {

  def main(args: Array[String]): Unit = {

    def isGoodEnough(guess: Double, x: Double) =
      guess * guess / x > 0.999 && guess * guess / x < 1.001
     // guess * guess is 99.9% close to x

    def improve(guess: Double, x: Double): Double =
      (guess + x / guess) / 2

    def sqrtIter(guess: Double, x: Double): Double =
      if (isGoodEnough(guess, x))
        guess
      else
        sqrtIter(improve(guess, x), x)

      // repeat improving guess until it is good enough

    def sqrt(x: Double) =
      sqrtIter(1, x)

    sqrt(2) 	// For check, println(sqrt(2))

  }

}
~~~
