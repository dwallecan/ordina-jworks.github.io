---
layout: post
authors: [tim_schmitte]
title: 'Building the applications of tomorrow with Kotlin coroutines, R2DBC and Spring webflux'
image: /img/todo.jpg
tags: [Spring,Spring Boot,R2DBC,Webflux,Reactive,Coroutines,Kotlin]
category: Reactive
comments: true
---

## Introduction

Want to create an application using a comfortable imperative style yet still efficient with your resources?

Maybe Kotlin's coroutines are the answer. 

Coroutines are essentially light weight processes which can run in the same thread and 
'yield' to eachother when they have to wait(for example when doing an expensive REST or database call). 
Thread creation is expensive since each thread requires memory for it's stack and reserving it costs time and space.
Coroutines don't have this issue, they can reuse the same thread.


##  Why coroutines ?

As you may have noticed in the last few years there's been increased attention for efficient resource usage; 
there's multiple reasons for this.
- Smartphones and their apps increased the amount of clients for applications
- Many more small devices running on limited battery life and increasing expectations from users about their responsiveness.
- Processor speeds are reaching their limits
- Applications are often being run in the cloud and the cost is based on the amount of resources used.
- The microservices architecture gained a lot of momentum in recent years=> many small applications need to communicate a lot with eachother
=> many (expensive) network calls
- ...

Here are some solutions
* Putting expensive calls on separate threads(background threads) 
    - Threads are expensive, creating on the fly creates performance overhead, keeping them in a pool consumes resources unnecessarily
* Callbacks
    - Google callback hell
    - error handling tends to be a pain
* Futures/Promises/...
    + These kind of api's help avoid callback hell, by returning resultwrappers on which you can chain methods 
    to do something with the eventually returned results.
    - Different programming model
    - Not standardized, many different apis
* Functional Reactive programming:
    + Very efficient resource usage
    + Modern frameworks (Spring in the java world) are quickly adopting it and its reaching maturity
    + Ported to many different environments, different implementations have similar api's
    + Improved error handling
    - Different programming model
    - Transparency 
* Coroutines!
    + Very efficient
    + No dramatic shift in programming style from good old fashioned imperative programming
    + Transparency
    - Still in its infancy in the java world
    
    
## How do they work?
Coroutines work very differently than Java threads, threads are preemptively scheduled, 
meaning the jvm's scheduler, a central entity, schedules execution of the different threads to assure that every thread gets an appropriate time slice,
this means the jvm's scheduler will interrupt them and switch to advancing other threads regularly.
Coroutines however keep running until they yield, then some kind of dispatcher is responsible for allowing another coroutine to do it's work.
This means the application needs to know which coroutines are available:  enter the CoroutineScope

This also means that you have to be careful when you have a heavy process running in a coroutine because 
it will block the other coroutines running on the same thread until it yields.

Threads also have their own scope, if an exception is thrown and propagated to the top of the stack it kills the thread(TODO: correct wording?).

## Coroutines in Kotlin

The Kotlin language itself only has support for coroutines, in the philosophy of the language the actual implementation is done by libraries.
The 1 they offer themselves is called Kotlinx-coroutines.
## Examples: 

### setup

### a simple test 
Let's try testing the build in delay function using junit 5


```kotlin
     public suspend fun delay(timeMillis: Long) { /* */ }
```

```kotlin
    
     @Test
     internal fun testDelay() {
        val startTime = System.currentTimeMillis()
        delay(1000)
        val endTime = System.currentTimeMillis()
        assertTrue(endTime - startTime >= 1000)
     }
```

Hey wait, this doesn't compile! Coroutines need a scope to run in.
Obviously since we're trying to test async code we will need to block here.
Luckily Kotlinx provides us with a function to do this for is.
Ideally there should be no need to use runBlocking in your production code,
but if you need to call coroutines from existing blocking code you may need to.

```kotlin
    @Test
    internal fun testDelay() {
        runBlocking {
            val startTime = System.currentTimeMillis()
            delay(1000)
            val endTime = System.currentTimeMillis()
            assertTrue(endTime - startTime >= 1000)
        }    
    }
```

This seems very similar to good old Thread.sleep right? 
Lets try a more advanced example with 2 coroutines

```kotlin
   runBlocking {
            launch {
                println("sub cr1: starting")
                println("sub cr1: yielding sub coroutine 1")
                delay(1000)
                println("sub cr1: resuming sub coroutine 1")

            }

            launch{
                println("sub cr2: starting sub coroutine 2")
                println("sub cr2: ending sub coroutine 2")
            }
            println("ending main coroutine" )
        }
```

output 
```text 
sub cr1: starting
sub cr1: yielding sub coroutine 1
sub cr2: starting sub coroutine 2
sub cr2: ending sub coroutine 2
sub cr1: resuming sub coroutine 1
```
As you can see first coroutine does not block!
The delay function suspends the current coroutine and allows the dispatcher to instead run coroutine 2!
Which after finishing automatically yields, allowing the dispatcher to execute coroutine 1.

