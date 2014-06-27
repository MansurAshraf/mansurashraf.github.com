---
layout: post
title: "A fistful of monoids"
description: ""
category: Scala
tags: [Scala,fp,monoid]
---
{% include JB/setup %}

I first ran into monoid while searching for monads on google. Monoids are ubiquitous in programming and chances are that you have already used them without being aware of them.  Wikipedia describes monoid as an algebraic structure with a single associative binary operation and an identity element. Now that definition is a mouthful and does not make much sense unless you are not rusty in college algebra. It is much easier to understand them by looking into some examples.
Before we delve into code, lets get some math out of the way. Lets look at the addition operation on integers

   1) Addition is associative i.e 1+(2+3)=(1+2)+3
   2) There is an identity element (e) for addition such that (e+i = i+e = i) i.e 5+0  =  0+5 = 5

Any data structure that obeys these two laws is a monoid and there are handful of them. Lets look at some more examples

  * Multiplication operation on integer.

    *  associative => (1*5)*2 = 1*(2*5)
    * Identity => 5*1= 1*5 =5

  * Concatenation operation on String
    * associative =>  "monoids" +("are"+"cool") = ("monoids"+"are)+"cool" ="monoids are cool"
    * Identity => "monoids"+""  =  ""+"monoids" = "monoids" , where "" is empty string

There are more examples of monoids such as append operation on list etc but I think we have gotten the picture.
Now at this point you may be wondering how they are useful in day to day programming. Before I show you some real world examples, let me introduce you to another operation that goes hand in gloves with Monoids.
Originating from the functional world, this function has already been incorporated in most main stream languages.
It is called Aggregate in C#, Inject in Ruby and Fold in Scala with variations such as FoldLeft and FoldRight.
Even Java is finally getting it in Java 8 as a part of project Lambda. Regardless of what it is called in your language, this function applies a binary operator to a start value and all elements of the collection.
Not very helpful definition but lets look at a example in Scala which calculates sum of all the elements in a list.

```
  val ints=List(1,2,3)
  ints.foldLeft(0){(a,b)=>a+b}
  6
``` 
 Basically foldLeft takes a seed value, 0 in our case, and a lambda that applies some binary operation on consecutive elements on the list.
 If you understand fold and if you squint slightly you will notice that the seed value in this example is nothing more than our identity element from our first monoid example and lambda we passed is simply the associative operation 'addition' on integer.
 I could have written the above example as follow

```
class AdditionMonoid {
def identity = 0
def op(a1: Int, a2: Int) = a1 + a2
}
val monoid = new AdditionMonoid
val ints = List(1, 2, 3)
ints.foldLeft(monoid.identity)(monoid.op)
```
 If you have ever looped over a collection to aggregate a value, chances are that you were dealing with a monoid and since they are associative, they play very well with parallelism. You can chunk the data into small pieces and fold over them in parallel. In addition product of two monoids is also a monoid which lets you compose more complex monoids. You can calculate both sum and length of a list in one Fold for example.

Now lets look at a semi real example of monoid. Lets say you are calling a web service that returns user information including social security number, credit card etc as JSON response. Lets say you want to scrub sensitive information before logging the response. One way you could do it using a monoid and foldLeft is as follow

JSON response
```
{
"first": "John",
"last": "Doe",
"credit_card": 5105105105105100,
"ssn": "123-45-6789",
"salary": 70000,
"registered": true }
A JSON obfuscation monoid
trait Monoid[A] {
def identity: A
def op(a1: A, a2: A): A
}

class JsonObfsucator extends Monoid[String] {
 val CREDIT_CARD_REGEX = "\\b\\d{13,16}\\b"
 val SSN_REGEX = "\\b[0-9]{3}-[0-9]{2}-[0-9]{4}\\b"
 val BLOCK_TEXT = "*********"

def identity = ""
def op(a1: String, a2: String) =  a1+a2.replaceAll(MASTER_CARD_REGEX, BLOCK_TEXT).replaceAll(SSN_REGEX, BLOCK_TEXT)+","
}
```
and the application

```
val monoid = new JsonObfsucator
val result = json.split(',').foldLeft(monoid.identity)(monoid.op)
println(result)
finally the result

{
"first": "John",
"last": "Doe",
"credit_card": *********,
"ssn": "*********",
"salary": 70000,
"registered": true
}
```
 a very trivial example showing the power of monoids and fold. No need to explicitly loop and mutate state. Monoids are very simple abstraction and if you like monoids you would love monads which bring even higher order of abstraction over various data types. Finally, If you are getting into functional programming, I would highly encourage reading Functional Programming in Scala which was the main inspiration for this post.
