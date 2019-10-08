---
layout: post
authors: [tim_schmitte]
title: 'Should you adopt Kotlin coroutines in your Spring Boot Application? '
image: /img/todo.jpg
tags: [Spring,Spring Boot,R2DBC,Webflux,Reactive,Coroutines,Kotlin]
category: Reactive
comments: true
---

## Introduction

Want to create an application using a comfortable imperative style yet still efficient with your resources?

Maybe Kotlin's coroutines are the answer. 

Functional reactive programming has been all the hype in recent years but let's face it;
these libraries completely change the way we program. 
They require a dramatic shift in thinking, and getting your whole team on board might be a pain. 
On top of that, in my experience, it is overkill for many use cases.

As we will see with Kotlin's coroutines we are going to try to keep a comfortable imperative style while retaining many of the benefits of asynchronous programming. 

TODO add an index to the subtitles

## What are Coroutines?

Coroutines are essentially lightweight processes which can run in the same thread and 'yield' to one another when they have to wait, for example when doing a network or database call. 
Thread creation is expensive since each thread requires memory for it's stack and reserving it costs time and space.
Coroutines don't have this issue, as they can reuse the same thread.

Coroutines work very different from Java threads, threads are scheduled preemptively. 
The jvm's scheduler, a central entity, knows which threads are available and schedules their execution to assure that every thread gets an appropriate time slice, it can interrupt them and switch to advancing other threads.

Coroutines are tasks which keep running until they either finish or yield.
After which some kind of dispatcher is responsible for allowing another coroutine to do it's work.
This also means that the programmer is responsible for letting the coroutines yield in time so other coroutines can do their work.
The advantage of this system is that you know that the operations in a single coroutine are executed sequentially until a yield happens or the task finishes!

Moreover, coroutines do not end until all sub coroutines end, they run in a hierarchy.
This makes coroutines a lot easier to reason about because this way no runaway side processes are spawned.

## Coroutines in Kotlin

The Kotlin language itself only has support for coroutines.
In the philosophy of the language the actual implementation is done by libraries.
The one they offer themselves is called [Kotlinx-coroutines](https://github.com/Kotlin/kotlinx.coroutines).

### A simple example
Let's try testing the built in delay function using junit 5


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

Hey wait, this doesn't compile! 
This is where Kotlin's language features come in.
As I explained earlier coroutines need to be run in a scope!
Notice the 'suspend' keyword on the delay function, this tells kotlin that this function can yield and needs to be run in a coroutine scope. 
'suspend' functions can only be run from inside a coroutine scope, meaning another suspending function or a function providing a coroutine scope.
Luckily kotlinx provides us with a few nice functions to easily setup a coroutine scope and get started.
In this case we will be using [runBlocking](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/run-blocking.html) since we don't want our test to finish until all coroutines finished execution. 
Ideally there should be no need to use runBlocking in your production code, but if you need to call coroutines from existing blocking code you may need to.

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

This seems very similar to good old 'Thread.sleep()' right? 
Lets try a more advanced example with 2 coroutines:

```kotlin
   runBlocking {
            launch {
                println("cr1: starting")
                println("cr1: yielding sub coroutine 1")
                delay(1000)
                println("cr1: resuming sub coroutine 1")

            }

            launch{
                println("cr2: starting sub coroutine 2")
                println("cr2: ending sub coroutine 2")
            }
            println("ending main coroutine" )
        }
```

output 
```text 
cr1: starting
cr1: yielding sub coroutine 1
cr2: starting sub coroutine 2
cr2: ending sub coroutine 2
cr1: resuming sub coroutine 1
```
As you can see; the first coroutine does not block other coroutines from executing!
The delay function yields and the current coroutine is suspended, which allows the dispatcher to instead run coroutine 2!
Which after finishing automatically yields, allowing the dispatcher to continue coroutine 1.

### Writing your own suspending functions
Let's figure out how delay achieves this by writing our own suspending function.

```kotlin
    private suspend fun suspendingFunction(): String = withContext(Dispatchers.Default) {
            println("coroutine 1 dispatched starting execution")

            Thread.sleep(5000)
            println("coroutine 1 dispatched executed")

            return@withContext "result of subcoroutine"
    }
    
    @Test
    internal fun selfImplementedSuspendingFunction() {
        runBlocking {
            launch {
                println("coroutine 1 starting execution")
                println("coroutine 1 calling suspending function")
                val suspendedCoroutineResult = suspendingFunction()
                println("coroutine 1 executed, result was : " + suspendedCoroutineResult)
            }
            launch{
                println("coroutine 2 executed")
            }
        }
    }
```

Output: 

```text
coroutine 1 starting execution
coroutine 1 calling suspending function
coroutine 2 executed
coroutine 1 dispatched starting execution
coroutine 1 dispatched executed
coroutine 1 executed, result was : result of subcoroutine
```

What actually happens here is kotlin switching the 'expensive' call implemented in #suspendingFunction() 
to a different Context which usually means a different Thread.
The key to this is the withContext() function. 
Obviously if your concern is resource usage this is not solving anything, as instead of blocking the main thread you will block another thread. 
Although it allows other coroutines to be run in the current context and gives you a degree of flexibility on where this blocking code should be run. 

To spice things up I also added a return type in this function, as you can see, you can seemingly call this function as if it was a normal method and seemingly immediately use the return value!
TODO lots of seeminglys

### Exception handling

When using other approaches like Promises , callbacks or reactive we usually pass a sort of on error function somewhere, highly dependant on the API we're using. 

Let's see what this looks like with Kotlin coroutines.

```kotlin
    @Test
    internal fun exceptionHandling() {
        runBlocking {
            try {
                throwingFunction()
            } catch (e: Exception) {
                println("Caught exception")
            }
        }
    }

    private suspend fun throwingFunction() = withContext(Dispatchers.Default){
        if(1==2){
            return@withContext "impossible"
        }
        throw RuntimeException("Totally unexpected exception")
    }

```

It's just plain old try catch!

### Parallelism

By default coroutines run sequentially, though concurrently. 
Until a yield happens all code is executed sequentially. 
Concurrency just means that multiple different coroutines can be run intermittently.
If you do want to run through processes in parallel you need to use the async function. 

```kotlin
    val startTime = System.currentTimeMillis()
    runBlocking {
        val asyncCoroutine1Result: Deferred<String> = async {
            delay(1500)
            println("async coroutine 1 returning after ${System.currentTimeMillis() - startTime} millis " )

            return@async "result of async coroutine 1"
        }

        val asyncCoroutine2Result: Deferred<String> = async {
            delay(2000)
            println("async coroutine 2 returning after ${System.currentTimeMillis() - startTime} millis " )
            return@async "result of async coroutine 2"
        }

        println("Awaiting results of both coroutines in top coroutine after ${System.currentTimeMillis() - startTime} millis")
        println("result of both functions was ${asyncCoroutine1Result.await() + ' ' + asyncCoroutine2Result.await()} after ${System.currentTimeMillis() - startTime} millis")
    }
```

Output:
```text
Awaiting results of both coroutines in top coroutine after 1 millis
async coroutine 1 returning after 1507 millis 
async coroutine 2 returning after 2005 millis 
result of both functions was result of async coroutine 1 result of async coroutine 2 after 2005 millis
``` 
The async function creates and immediately launches a new asynchronous coroutine, by default, there's a parameter to make it lazy.
This is pretty comparable to promise type libraries. TODO: like ...
A key difference is that, even though the sub coroutines are run in parallel, the outer coroutine will never finish before all 
inner coroutine are finished, even if the result is not used! 
This is called structured concurrency.

### Structured concurrency

We programmers write horrendously big applications which are hard to fit in our mammal brains.
One thing what makes applications, even more so asynchronous applications, hard to understand and reason about are side effects.
If I am examining a piece of code and I need to inspect the code of every function it calls (and the functions they call) to understand what it does I will be wasting a lot of time and brainpower. 
It is very easy, especially with simple promise like libraries, to create seemingly synchronous functions which spawn a task somewhere doing stuff you have no control on anymore while the caller is unaware. 
This can lead to unwanted side effects and a waste of resource usage.

Coroutines tackle it like this: once you are in a coroutine scope every coroutine needs to finish before the parent finishes.

```kotlin
    val startTime = System.currentTimeMillis()
    runBlocking {
        val topCoroutineJob: Job = launch {
            launch {
                delay(1000)
                println("sub cr 1 returning after ${System.currentTimeMillis() - startTime} millis")
            }
 
 
            launch {
                delay(1500)
                println("sub cr returning after ${System.currentTimeMillis() - startTime} millis")
            }
 
            println("last line in top coroutine reached at ${System.currentTimeMillis() - startTime} millis")
        }
 
        topCoroutineJob.invokeOnCompletion(true, true) {
            println("completed top coroutine ")
        }
    }
```

TODO Reference to dijkstra article (Trio)
output: 
```text
async coroutine 2 returning after 3006 millis 
completed top coroutine
```
(Disclaimer: I'm using GlobalScope.launch here + a thread.sleep on purpose because i found runBlocking confusing in this context)
Even though the async coroutine is run on a completely different thread and nothing is awaiting it's result the top coroutine 
is not completed until the inner (async) coroutine is finished! 

TODO

##  Why and when should you consider using coroutines ?

## Increasing demand for efficient resource usage
As you may have noticed in the last few years there has been increased attention for efficient resource usage. 
There are multiple reasons for this:
- Smartphones and their apps increased the amount of clients for applications
- Many more small devices running on limited battery life and increasing expectations from users about their responsiveness.
- Processor speeds are reaching their limits
- Applications are often being run in the cloud and part of the cost is often based on the amount of resources used.
- The microservices architecture gained a lot of momentum in recent years=> many small applications need to communicate a lot with eachother
=> many (expensive) network calls
- ...
Older solutions like manual thread management and callbacks quickly turn your application in to a big ball of mud in these environments.

The most popular solutions these days are either promise/future like libraries or fully reactive applications.
A lot has been written about callbacks and manual thread management and their limitations so I will mainly focus on comparing with these 
and why coroutines is a worthy competitor.

### Programming model
Promise/future like libraries help avoid callback hell by providing DSLs to handle asynchronous calls.
These libraries aren't standardized and require you to learn a new API which usually impacts the resulting code quite a bit.
The advantage is they are not invasive at all, you can use them in 1 place without affecting the rest of your code base.

FRP libraries(Reactor/Rxjava/...)  TODO add some links, with maybe a single sentence explanation of the libs
These libraries are a very good fit for applications that are very performance/resource critical.
But they are very invasive; to fully leverage them you need to make your applications reactive end to end. 
Learning and mastering them requires quite some involvement, also from the rest of your team and any future developer that might have to maintain your system. 

Coroutines fit somewhere in between, in my opinion. 
They are more invasive than Promise/future libraries.
Imagine changing an existing application method to a suspend function; you will need to make all callers run in a coroutine scope!
But once you're there, you have an application that can be reasoned about more easily like a regular application.

### Resource management
Promise/future like libraries don't really do much here except for unblocking a certain thread, often to give faster results to
a user. 
Somewhere there's still a thread running doing the job instead.

FRP really makes a difference here; by making the source of the data responsible for pushing the response back the application itself consumes less resources.

With coroutines you can take both approaches, luckily coroutines and reactive programming are quite compatible and there's 
already some effort going on in the spring environment to bridge them. 
As you can see at: [https://spring.io/blog/2019/04/12/going-reactive-with-spring-coroutines-and-kotlin-flow](https://spring.io/blog/2019/04/12/going-reactive-with-spring-coroutines-and-kotlin-flow)

## Performance

TODO

## Conclusion

I dully recommend using Kotlin Coroutines





 
