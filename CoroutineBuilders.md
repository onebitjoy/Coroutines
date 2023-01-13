# Coroutine Builders

Coroutines are light-weight threads that do not block the main thread while providing the ability to call suspending functions on the main thread. Suspending functions are functions that have the ability to suspend the execution of current coroutine without blocking the current thread. These functions can resume execution at some point. The suspending feature of these functions helps us to write asynchronous code in a synchronous manner. It is also worthwhile to note that suspending functions are not asynchronous themselves.

```kotlin
suspend fun doSomething() : String {
    return “Do something”
}
```

Then how do we start Coroutines? Use *Coroutine Builders*.

## Coroutine Builders

Coroutine builders are the piece of code that can create a coroutine. They are not suspending, hence can be used as link between suspending functions and other parts of the code.

There are 3 types of builders - 
1. runBlocking
2. async
3. launch

### runBlocking

runBlocking is a coroutine builder which can block the current thread until all the tasks of the coroutine it creates has finished. So why to block when it was a necessity for a coroutine to not suspend the execution thread. It's because it is used to test the coroutines. We make sure that the test is not completed until the test suspended function finishes.

### launch

Also called `fire and forget.` Launch is a coroutine builder that is "Use and Forget". It creates a new coroutine that will not return anything to the caller. It can also start a coroutine in the background.
1. Launches a new coroutine without blocking the current thread.
2. Inherits the thread & coroutine scope of the immediate parent coroutine.
3. Returns a Job object which can be used to cancel or continue and wait for the coroutine to finish.

```kotlin
import kotlinx.coroutines.*

@OptIn(DelicateCoroutinesApi::class)
fun main() {
    println("Starting main function..")

    GlobalScope.launch { // GlobalScope explained below
        println(myDelayFunc())
    }

    runBlocking {
        /* Alone GlobalScope.launch won't be able to keep the JVM alive, since program doesn't wait for coroutines unless main thread is suspended using the runBlocking{}*/

        delay(4000L) // make sure to keep the JVM alive in order to wait for doSomething() to execute
    }
}

/* The total execution time is 4 seconds, within which the launch{} code was executed, the main thread was suspended for 4000ms by the runBlocking method*/

suspend fun myDelayFunc() : String {
    delay(3000L) // simulate long running task
    return "A 3 second long execution..."

```

The reason why we use **GlobalScope** with launch is because if we launch launch{} over an activity, which is living only until the activity itself is running(a local scoped coroutine is created), If the user hits back button and exists, or something happens and the activity or function stops, the coroutine will be destroyed. Hence to give the coroutine the scope of a lifetime we use the **GlobalScope** companion object.

GlobalScope is important when the function needs to be running all the time. For example -
- A download must complete, and it shouldn't depend on where the user is at.
- A song must be playing even if user exists the main screen and should continue until user turns it off or a change occurs(Battery Low/ No Internet etc).

The pure launch{} must be used in the scenerios such as -
- Some Data Computation which needs to be destroyed when user exists
- Login limited to only one screen.

**By default use launch{}**

In the chapter 1(WhatIsCoroutine.md), we created a GlobalScoped launch built coroutine, which was running inside a runBlocking built coroutine, which runs on main thread. Hence the child launch{} created coroutine runs on the main thread.


_How to not hardcode the coroutine timing and give it time to complete -_
```kotlin
import kotlinx.coroutines.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
    
    println("Starting main function..")

    val job : Job = launch {
        println(doSomething())
    }
    
//    delay(4000L)

    /* Instead of hardcoding the time here, we can just take the returned Job object from the launch call and call a function over it. The join function waits until the launch coroutine is completed and then the further execution starts sequentially*/

    job.join()
    
    println("This line is printed after the job is completed.")
}

suspend fun doSomething() : String {
    delay(3000L) // simulate long running task
    return "Did something that was 3 seconds long"
}
```


### async

async is a coroutine builder which **returns some value to the caller**. async can be used to perform an asynchronous task which returns a value. This value in Kotlin terms is a Deferred<T> value.

The below example doesn't return anything. So it is not advisable to use async here -
```kotlin
import kotlinx.coroutines.*

fun main() {
    println("Hello, ")
    GlobalScope.async {
        doSomething()
    }
    
    runBlocking {
        delay(3000L) // only used to keep the JVM alive
    }
}
suspend fun doSomething() {
    delay(2000L)
    print("this is bad example for using async!!!")
}
```

To use async as a coroutine builder, we need to use a catcher(or await) which waits for the coroutine to be finished and return a value - 

### async-await

await() is a suspending function that is called upon the async builder to fetch the value of the Deferred object that is returned. The coroutine started by async will be suspended until the result is ready. When the result is ready, it is returned and the coroutine resumes.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("Thread started on: ${Thread.currentThread().name}")
c
    val jobDeffered : Deferred<Int> = async {
        mySuspendFunction(5,8)
    }

    println("The value of a+b is ${jobDeferred.await()}. This line is printed after 2s as delay is used.")
}

suspend fun mySuspendFunction(a:Int, b:Int): Int{
    delay(2000)
    return a+b
}
```

Remember that we are using coroutines created by runBlocking function, hence the inner code is working on the main thread in this case. We can also launch a global coroutine which runs on a different global coroutine by using GlobalScope.

> GlobalScope.async{ .... }

Conclusion -
1. The async function launches a coroutine just like the launch function. Hence, it doesn't blocks the thread it is executing on.
2. Just like launch{...}, it inherits the thread & coroutine scope of the parent coroutine.
3. But, instead of returning a Job object, _It returns a deferred<T> object._ Which is the sub class of Job interface.
4. Using the deferred object we can -
    - Get the returned value.
    - Use _.cancel()_ or _.join()_ on the object.

### Extending runBlocking to Testing

We can use runBlocking to test our suspending function.

```kotlin
import org.junit.Assert
import org.junit.Test
import kotlinx.coroutines.runBlocking

class TestMe {
    @Test
    fun tester() = runBlocking{
        Assert.assertEquals(15, mySuspendFunction(5,10))
    }
}
```