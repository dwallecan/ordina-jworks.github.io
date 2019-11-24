---
layout: post
authors: [tim_schmitte, jeroen_meys]
title: 'Should you adopt Kotlin coroutines in your Spring Boot Application? '
image: /img/cypress/cypress-logo.png?TODO
tags: [Spring,Spring Boot,R2DBC,Webflux,Reactive,Coroutines,Kotlin]
category: Reactive
comments: true
---

TODO images

## Introduction

Want to create an application using a comfortable imperative style yet still efficient on resources?

Maybe Kotlin's coroutines are the answer. 

Functional reactive programming has been all the hype in recent years but let's face it;
these libraries completely change the way we program. 
They require a dramatic shift in thinking, and getting your whole team on board might be a pain. 
On top of that, in our opinion, it is overkill for many use cases.

As we will see with Kotlin's coroutines we are going to try to keep a comfortable imperative style while retaining many of the benefits of asynchronous programming. 

TODO add an index to the subtitles

## What are Coroutines?

Coroutines are essentially lightweight processes which can run in the same thread and 'yield' to one another when they have to wait, for example when doing a network or database call. 
Thread creation is expensive since each thread requires memory for its stack and reserving it costs time and space.
Coroutines don't have this issue, as they can reuse the same thread.
Coroutines are very different from Java threads, threads are scheduled preemptively. 
The jvm's scheduler, a central entity, knows which threads need to run and schedules their execution to assure that every thread gets an appropriate time slice, 
it can interrupt them and switch to advancing other threads.

Coroutines are tasks which keep running until they either finish or yield.
After which some kind of dispatcher is responsible for allowing another coroutine to do its work.
This also means that the programmer is responsible for letting the coroutines yield in time so other coroutines can do their work.
The advantage of this system is that you know that the operations in a single coroutine are executed sequentially until a yield happens or the task finishes.

Moreover, coroutines do not end until all sub coroutines end, they run in a hierarchy.
This makes coroutines a lot easier to reason about because this way no runaway side processes are spawned.
This is part of what's called structured concurrency.

TODO structured concurrency link

## Coroutines in Kotlin

The Kotlin language itself only has support for coroutines.
In the philosophy of the language the actual implementation is done by libraries.
The one they offer themselves, and which we will be using here is called [Kotlinx-coroutines](https://github.com/Kotlin/kotlinx.coroutines)

### A simple example
Let's try running a simple coroutine 

<code class="kotlin-code" 
    auto-indent="true" 
    highlight-on-fly="true" >
import kotlinx.coroutines.*    
//sampleStart
fun main() {
    GlobalScope.launch {
        println("running my first coroutine")
    }
    println("end")
}
//sampleEnd</code>


output: 
```text 
end
```

That was kinda disappointing wasn't it? Nothing seems to have happened because the launch() method by default launches in a separate thread in a Thread pool.
In truth sometimes you're lucky and the message 'running my first coroutine' is actually printed.
If you're unlucky, the test's main thread finishes before the coroutine's thread. 
We could solve this by putting a thread.sleep at the end of the test method; obviously that's a rather ugly solution since it completely blocks a thread for an arbitrary amount of time. 
Luckily the kotlin team provides us with a function which just blocks until the completion of the coroutine!

TODO: timeline image!

<code class="kotlin-code" 
    auto-indent="true" 
    highlight-on-fly="true" >
import kotlinx.coroutines.*
fun main() {
    launchingASimpleCoroutine2()
}
//sampleStart
internal fun launchingASimpleCoroutine2() {
    runBlocking {
        println("running my first coroutine")
    }
    println("end")
}
//sampleEnd</code>

output: 
```text 
    running my first coroutine
    end
```

So far nothing special. Even worse! If you use Globalscope.launch() you create a task in a thread over which you have no control and is unpredictable!
So where does the magic start you ask? Once we're inside a coroutine things become a lot more predictable! 

TODO bridge this shit
Let's try testing the built in delay function using junit 5

<code class="kotlin-code"  
    data-highlight-only>
public suspend fun delay(timeMillis: Long) { /* */ }
</code>

<code class="kotlin-code" 
    data-target-platform="junit" 
    auto-indent="true" 
    highlight-on-fly="true">
import kotlinx.coroutines.*
import org.junit.Test
import org.junit.Assert.assertTrue
class Test {
//sampleStart
    @Test
    internal fun testDelay() {
        runBlocking {
            val startTime = System.currentTimeMillis()
            delay(100)
            val endTime = System.currentTimeMillis()
            assertTrue(endTime - startTime >= 100)
        }    
    }
//sampleEnd
}</code>

This seems very similar to good old 'Thread.sleep()' right? 
Lets try a more advanced example with 2 coroutines:

<code class="kotlin-code" 
    auto-indent="true" 
    highlight-on-fly="true" >
import kotlinx.coroutines.*
fun main() {
    runBlocking {
        launch {
            println("cr1: starting")
            println("cr1: YIELDING sub coroutine 1")
            delay(1000)
            println("cr1: RESUMING sub coroutine 1")
        }
        launch{
            println("cr2: STARTING sub coroutine 2")
            println("cr2: ENDING sub coroutine 2")
        }
    }
}
</code>

output 
```text 
cr1: starting
cr1: YIELDING sub coroutine 1
cr2: STARTING sub coroutine 2
cr2: ENDING sub coroutine 2
cr1: RESUMING sub coroutine 1
```
As you can see; the first sub coroutine does not block the other sub coroutines from executing!
The delay function yields and the current coroutine is suspended, which allows the dispatcher to instead run coroutine 2!
Which after finishing automatically yields, allowing the dispatcher to continue coroutine 1.
Notice the 'suspend' keyword on the delay function? This is kotlin's language feature preventing us from using this function outside a coroutine scope.
Suspending functions can only be used directly inside a coroutine or in another suspending function.


### Writing your own suspending functions
Let's figure out how delay achieves this by writing our own suspending function.

<code class="kotlin-code"  
    auto-indent="true" 
    highlight-on-fly="true">
import kotlinx.coroutines.*
import org.junit.Test
import org.junit.Assert.assertTrue
fun main() {
    selfImplementedSuspendingFunction()
}
//sampleStart
private suspend fun suspendingFunction(): String = withContext(Dispatchers.Default) {
    println("coroutine 1 dispatched starting execution")
    Thread.sleep(5000)
    println("coroutine 1 dispatched executed")
    return@withContext "result of subcoroutine"
}
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
//sampleEnd
</code>

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
to a different Context(in practice a different thread pool) using withContext(Dispatchers.Default).
The key to this is the withContext() function. 
Obviously if your concern is resource usage this is not solving anything, as instead of blocking the main thread you will block another thread. 
Although it allows other coroutines to be run in the current context and gives you a degree of flexibility on where this blocking code should be run. 

To spice things up I also added a return type in this function, as you can see, you can call this function as if it was a normal method and seemingly immediately use the return value!

### Exception handling and cancellation

When using other approaches like Promises , callbacks or reactive we usually pass a sort of on error function somewhere, highly dependant on the API we're using. 

Let's see what this looks like with Kotlin coroutines.

<code class="kotlin-code" 
    data-target-platform="junit" 
    auto-indent="true" 
    highlight-on-fly="true">
import kotlinx.coroutines.*
import org.junit.Test
import org.junit.Assert.assertTrue
class Test {
//sampleStart
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
//sampleEnd
}</code>

It's just plain old try catch!
Even though the execution might be spread over different threads or be done at different times, the stack is retained in the coroutine.
 
### Parallelism

By default coroutines run sequentially, though concurrently. 
Until a yield happens all code is executed sequentially. 
Concurrency just means that multiple different coroutines can be run intermittently.
If you do want to run processes in parallel you need to use the async function. 

<code class="kotlin-code">
import kotlinx.coroutines.*
//sampleStart
fun main() {
    val startTime = System.currentTimeMillis()
    runBlocking {
        val asyncCoroutine1Result: Deferred&lt;String&gt; = async {
            delay(1500)
            println("async coroutine 1 returning after ${System.currentTimeMillis() - startTime} millis " )
            return@async "result of async coroutine 1"
        }
        val asyncCoroutine2Result: Deferred&lt;String&gt; = async {
            delay(2000)
            println("async coroutine 2 returning after ${System.currentTimeMillis() - startTime} millis " )
            return@async "result of async coroutine 2"
        }
        println("Awaiting results of both coroutines in top coroutine after ${System.currentTimeMillis() - startTime} millis")
        println("result of both functions was ${asyncCoroutine1Result.await() + ' ' + asyncCoroutine2Result.await()} after ${System.currentTimeMillis() - startTime} millis")
    }
}
//sampleEnd
</code>


Output:
```text
Awaiting results of both coroutines in top coroutine after 1 millis
async coroutine 1 returning after 1507 millis 
async coroutine 2 returning after 2005 millis 
result of both functions was result of async coroutine 1 result of async coroutine 2 after 2005 millis
``` 
The async function creates and immediately launches a new asynchronous coroutine, by default, there's a parameter to make it lazy.
This is pretty comparable to promise type libraries like CompletableFutures in java.
A key difference is that, even though the sub coroutines are run in parallel, the outer coroutine will never finish before all 
inner coroutines are finished, even if the result is not used! 

### Structured concurrency

We programmers write horrendously big applications which are hard to fit in our mammal brains.
One thing what makes applications, even more so asynchronous applications, hard to understand and reason about are side effects.
If I am examining a piece of code and I need to inspect the code of every function it calls (and the functions they call) to understand what it does I will be wasting a lot of time and brainpower. 
It is very easy, especially with simple promise like libraries, to create seemingly synchronous functions which spawn a task somewhere doing stuff you have no control on anymore while you are unaware as a caller. 
This can lead to unwanted side effects and a waste of resource usage.

Coroutines tackle it like this: once you are in a coroutine every coroutine in its scope needs to finish before the parent finishes.
This is part of what's called 'Structured Concurrency'

<code class="kotlin-code">
import kotlinx.coroutines.*
@UseExperimental(kotlinx.coroutines.InternalCoroutinesApi::class)
//sampleStart
fun main() {
    val startTime = System.currentTimeMillis()
    runBlocking {
        val outerCoroutine: Job = launch {
            launch {
                delay(1000)
                println("sub cr 1 returning after ${System.currentTimeMillis() - startTime} millis")
            }
            launch {
                delay(1500)
                println("sub cr 2 returning after ${System.currentTimeMillis() - startTime} millis")
            }
            println("last line in outer coroutine reached at ${System.currentTimeMillis() - startTime} millis")
        }
        outerCoroutine.invokeOnCompletion(true, true) {
            println("completed outer coroutine")
        }
    }
}
//sampleEnd
</code>

output: 
```text
last line in outer coroutine reached at 4 millis
sub cr 1 returning after 1005 millis
sub cr 2 returning after 1505 millis
completed outer coroutine
```

If you want to learn more about this concept i will refer you to following article: 
TODO link syntax : https://vorpus.org/blog/notes-on-structured-concurrency-or-go-statement-considered-harmful/


## Failure and Cancellation

One has to take care when handling failure when running functions in parallel.
To higlight the differences we'll start by comparing with java's CompletableFuture API.
Imagine we want to start executing some code asynchronously while we continue processing .

Let's see what a naive approach looks like:

<code class="kotlin-code">
import kotlinx.coroutines.*
fun main() {
    runBlocking(Dispatchers.IO) {
    //sampleStart
        val asyncMethodResult = asyncMethod()
        val syncMethodResult = doStuff()
        println("combined result was ${asyncMethodResult.await()} $syncMethodResult")
    //sampleEnd
    }
}
private fun CoroutineScope.asyncMethod(): Deferred<String> {
    val job = async {
        val startTime = System.currentTimeMillis()
        while (System.currentTimeMillis() - startTime < 1500) {
            println("Async method is processing")
            delay(300)
        }
        return@async "result of async method"
    }
    job.invokeOnCompletion { ex ->
        if (ex is CancellationException) {
            println("async method cancelled!")
        } else {
            println("completed because of other reasons")
        }
    }
    val asyncMethodResult = job
    return asyncMethodResult
}
fun doStuff(): String {
    Thread.sleep(200)//Giving a chance to the async method to start
    if (true) {
        println("throwing exception in sync method")
        throw RuntimeException("Oh shi...")
    } else {
        return "result"
    }
}
</code>

The async method keeps processing even though the result will be wasted!
Depending on the library you're using there are usually some ways around this; usually the solution is to wrap the synchronous method(s) in (an) async call(s) 
and combine them using a function like CompletableFuture.allOf() or completableFuture.thenAcceptBoth().
While doing the job these approaches require you to wrap other 'regular' synchronous code(introducing unnecessary complexity) 
and require you to find the right method for your usecase.

Now let's see how kotlin tackles this: 

<code class="kotlin-code">
import kotlinx.coroutines.*
fun main() {
/*Using async with the default Dispatcher makes it default to use the same Thread...*/
    runBlocking(Dispatchers.IO) {
    //sampleStart
        val asyncMethodResult = asyncMethod()
        val syncMethodResult = doStuff()
        println("combined result was ${asyncMethodResult.await()} $syncMethodResult")
    //sampleEnd
    }
}
private fun CoroutineScope.asyncMethod(): Deferred<String> {
    val job = async {
        val startTime = System.currentTimeMillis()
        while (System.currentTimeMillis() - startTime < 1500) {
            delay(300)
            println("Async method still processing")
        }
        return@async "result of async method"
    }
    job.invokeOnCompletion { ex ->
        if (ex is CancellationException) {
            println("async method cancelled!")
        } else {
            println("completed because of other reasons")
        }
    }
    val asyncMethodResult = job
    return asyncMethodResult
}
fun doStuff(): String {
    Thread.sleep(200)//Giving a chance to the async method to start
    if (true) {
        println("throwing exception in sync method")
        throw RuntimeException("Oh shi...")
    } else {
        return "result"
    }
}
</code>

The code may look not too different from the 'naive' completableFuture example but the outcome is very different;
the async method is cancelled! This happens because the exception is raised to the outer coroutine and it will try to cancel all child coroutines.

There's a caveat though: let's take previous example and change the async function to mimic a heavy algorithm.

<code class="kotlin-code">
import kotlinx.coroutines.*
import java.util.concurrent.Executors
fun main() {
/*Using async with the default Dispatcher makes it default to use the same Thread...*/
    runBlocking(Dispatchers.IO) {
        val asyncMethodResult = asyncMethod()
        val syncMethodResult = doStuff()
        println("combined result was ${asyncMethodResult.await()} $syncMethodResult")
    }
}
private fun CoroutineScope.asyncMethod(): Deferred<String> {
    val job = async {
        //sampleStart
        val startTime = System.currentTimeMillis()
        var previousTimeDecis = 0L
        //Executing a computation loop which will exhaust the thread and print every 100 ms
        while (System.currentTimeMillis() - startTime < 700) {
            val currTimeMillis = System.currentTimeMillis()
            val currentTimeDecis = currTimeMillis - currTimeMillis % 100
            if (previousTimeDecis != currentTimeDecis) {
                println("Heavy computation in progress")
            }
            previousTimeDecis = currentTimeDecis
        }
        return@async "result of async method"
        //sampleEnd
    }
    job.invokeOnCompletion { ex ->
        if (ex is CancellationException) {
            println("async method cancelled!")
        } else {
            println("completed because of other reasons")
        }
    }
    val asyncMethodResult = job
    return asyncMethodResult
}
suspend fun doStuff(): String {
    delay(200)//Giving a chance to the async method to start
    if (true) {
        println("throwing exception in sync method")
        throw RuntimeException("Oh shi...")
    } else {
        return "result"
    }
}
</code> 

The async method is not cancelled here until it's done with the 'algorithm'!
Coroutines are cancellation cooperative; this means the coroutine code has to check for cancellation itself!
When using built-in functions like delay this should be handled for you, but when implementing your own heavy operations you should take care of it yourself!

One way to resolve this is by calling the built-in [yield](https://kotlin.github.io/kotlinx.coroutines/kotlinx-coroutines-core/kotlinx.coroutines/yield.html) function.
This function is cancellable() and will notice the coroutine has been marked as cancelled.

For example: 
<code class="kotlin-code">
import kotlinx.coroutines.*
import java.util.concurrent.Executors
fun main() {
/*Using async with the default Dispatcher makes it default to use the same Thread...*/
    runBlocking(Dispatchers.IO) {
        val asyncMethodResult = asyncMethod()
        val syncMethodResult = doStuff()
        println("combined result was ${asyncMethodResult.await()} $syncMethodResult")
    }
}
private fun CoroutineScope.asyncMethod(): Deferred<String> {
    val job = async {
        val startTime = System.currentTimeMillis()
        var previousTimeDecis = 0L
        //Executing a computation loop which will exhaust the thread and print every 100 ms
        //sampleStart
        while (System.currentTimeMillis() - startTime < 700) {
            val currTimeMillis = System.currentTimeMillis()
            val currentTimeDecis = currTimeMillis - currTimeMillis % 100
            if (previousTimeDecis != currentTimeDecis) {
				yield()
                println("Heavy computation in progress")
            }
            previousTimeDecis = currentTimeDecis
        }
        //sampleEnd
        return@async "result of async method"
    }
    job.invokeOnCompletion { ex ->
        if (ex is CancellationException) {
            println("async method cancelled!")
        } else {
            println("completed because of other reasons")
        }
    }
    val asyncMethodResult = job
    return asyncMethodResult
}
suspend fun doStuff(): String {
    delay(100)//Giving a chance to the async method to start
    if (true) {
        println("throwing exception in sync method")
        throw RuntimeException("Oh shi...")
    } else {
        return "result"
    }
}
</code>

You can also do it manually by checking the isActive property in the coroutine's scope:

<code class="kotlin-code">
import kotlinx.coroutines.*
import java.util.concurrent.Executors
fun main() {
/*Using async with the default Dispatcher makes it default to use the same Thread...*/
    runBlocking(Dispatchers.IO) {
        val asyncMethodResult = asyncMethod()
        val syncMethodResult = doStuff()
        println("combined result was ${asyncMethodResult.await()} $syncMethodResult")
    }
}
private fun CoroutineScope.asyncMethod(): Deferred<String> {
    val job = async {
        //sampleStart
        val startTime = System.currentTimeMillis()
        var previousTimeDecis = 0L
        //Executing a computation loop which will exhaust the thread and print every 100 ms
        while (System.currentTimeMillis() - startTime < 700 && isActive) {
            val currTimeMillis = System.currentTimeMillis()
            val currentTimeDecis = currTimeMillis - currTimeMillis % 100
            if (previousTimeDecis != currentTimeDecis) {
                println("Heavy computation in progress")
            }
            previousTimeDecis = currentTimeDecis
        }
        return@async "result of async method"
        //sampleEnd
    }
    job.invokeOnCompletion { ex ->
        if (ex is CancellationException) {
            println("async method cancelled!")
        } else {
            println("completed because of other reasons")
        }
    }
    val asyncMethodResult = job
    return asyncMethodResult
}
suspend fun doStuff(): String {
    delay(100)//Giving a chance to the async method to start
    if (true) {
        println("throwing exception in sync method")
        throw RuntimeException("Oh shi...")
    } else {
        return "result"
    }
}
</code>

(Regular) Kotlin coroutines also support manual cancellation; it looks like this: 

<code class="kotlin-code">
import kotlinx.coroutines.*
//sampleStart
fun main() {
    runBlocking{
        var coroutine =launch{
            println("starting coroutine")
        	delay(2000)
        	println("this will never happen")
    	}  
        delay(1000)
        println("cancelling")
        coroutine.cancelAndJoin()
        println("cancelled")
    }
}
//sampleEnd
</code>

Simple right? 
There are a few caveats though: coroutines are cancellation cooperative; this means the coroutine has to support cancellation itself.
If your coroutine code is doing a large computation you need to take care of this yourself.
For example:

<code class="kotlin-code">
import kotlinx.coroutines.*
//sampleStart
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (i < 5) { // computation loop, just wastes CPU
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")    
}
//sampleEnd</code>
([shamelessly copied from the kotlin website](https://kotlinlang.org/docs/reference/coroutines/cancellation-and-timeouts.html))

As you can see the coroutine  isn't cancelled until the computation is done!
There's a few ways around this; you can call another suspending function like yield() or you can manually check the isActive property of the job in the loop.

<code class="kotlin-code">
import kotlinx.coroutines.*
//sampleStart
fun main() = runBlocking {
    val startTime = System.currentTimeMillis()
    val job = launch(Dispatchers.Default) {
        var nextPrintTime = startTime
        var i = 0
        while (isActive) { // computation loop, just wastes CPU
            // print a message twice a second
            if (System.currentTimeMillis() >= nextPrintTime) {
                println("job: I'm sleeping ${i++} ...")
                nextPrintTime += 500L
            }
        }
    }
    delay(1300L) // delay a bit
    println("main: I'm tired of waiting!")
    job.cancelAndJoin() // cancels the job and waits for its completion
    println("main: Now I can quit.")    
}
//sampleEnd
</code>


Obviously this can be tricky. 
Luckily in-built functions from kotlin already take care of this and if you're using any I/O libraries they should also be taking this into account.
For most purposes you should not be worried about this.

If you want a full explanation i refer you to the [kotlin website](https://kotlinlang.org/docs/reference/coroutines/cancellation-and-timeouts.html).

## Channels
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
already some effort going on in the spring environment to [bridge](https://spring.io/blog/2019/04/12/going-reactive-with-spring-coroutines-and-kotlin-flow) them.

## Performance

TODO

## Conclusion

Although still in its infancy there's some very interesting


<!-- transforms interactive Kotlin code snippets -->
<script src="https://unpkg.com/kotlin-playground@1" data-selector="code.kotlin-code"></script>

 
