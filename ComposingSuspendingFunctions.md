# <center>Composing Suspending Function</center>

<span style="font-variant:small-caps">Sequential, Concurrent and Lazy Execution</span>

Revising -
A suspending function is a function that starts with the keyword `suspend`.
Suspend function is a function that could be started, paused, and resume. One of the most important points to remember about the suspend functions is that they are only allowed to be called from a coroutine or another suspend function.

```kotlin
suspend fun msgOne(){
    delay(1000)
    println("First msg!")
}
```

<hr>

**Inside a coroutine, the code is always executed in the sequential manner by default.**

### <span style="color:cyan">1. Sequential Execution</span>

```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {

    println("The thread running: ${Thread.currentThread().name}")

    val time = measureTimeMillis { // the time will be calculated for the code inside this
        val msgOne = msgFuncOne()
        val msgTwo = msgFuncTwo()
        // The above 2 functions are executed sequentially
        println("The entire msg is: ${msgOne + msgTwo}")
    }

    println("The code is executed in $time ms")
}

suspend fun msgFuncOne(): String {
    delay(500L)
    return "Hello "
}

suspend fun msgFuncTwo(): String {
    delay(500L)
    return "World!"
}
```

<br/>

### <span style="color:cyan">2. Concurrent(Parallel) Execution</span>

The code below is executing in parallel, it will execute very fast as compared to the sequential excution in the above code.


```kotlin
import kotlinx.coroutines.*
import kotlin.system.measureTimeMillis

fun main() = runBlocking {

    println("The thread running: ${Thread.currentThread().name}")

    val time = measureTimeMillis { // the time will be calculated for the code inside this
        val msgOne = async { msgFuncOne() }
        val msgTwo = async { msgFuncTwo() }

        println("The entire msg is: ${msgOne.await() + msgTwo.await()}")
    }

    println("The code is executed in $time ms")
}

suspend fun msgFuncOne(): String {
    delay(500L)
    return "Hello "
}

suspend fun msgFuncTwo(): String {
    delay(500L)
    return "World!"
}
```

<br/>

### <span style="color:cyan">3. Lazy Concurrent Execution</span>

The Lazy Concurrent coroutine is a coroutine builder that initiates execution of the coroutine, only when it is used somewhere. 

The `async` builder builds the coroutine and waits. To execute the coroutine, the Deferred value that was returned to some variable must be used somewhere. In this case, `msgOne.await()` forces the execution of the coroutine.

If there is no usage of the Deferred value, then the coroutine won't start and the suspending functions willn't execute.

If coroutine *Job is cancelled* before it even had a chance to start executing, then it will not start its execution at all, but will complete with an <span style="color:red;">exception.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("The thread running: ${Thread.currentThread().name}")

        val msgOne: Deferred<String> = async(start = CoroutineStart.LAZY) { msgFuncOne() }
        val msgTwo: Deferred<String> = async(start = CoroutineStart.LAZY) { msgFuncTwo() }
        
        println("The entire msg is: ${msgOne.await() + msgTwo.await()}") //comment this, and coroutine won't execute

        // The above line is responsible for execution of the coroutine.

    println("Still executing inside: ${Thread.currentThread().name}")
}

suspend fun msgFuncOne(): String {
    delay(500L)
    println("After working in first function.")
    return "Hello "
}

suspend fun msgFuncTwo(): String {
    delay(500L)
    println("After working in second function.")
    return "World!"
}
```
