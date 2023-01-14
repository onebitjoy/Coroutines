# <center>Cancellation of coroutines</center>
A coroutine can be cancelled for many reasons. Some of them are - 
1. The coroutine might not be required now.
2. The coroutine is taking longer than needed/required.

To cancel a coroutine, the coroutine should be cooperative.(Later explained)

```kotlin
val job: Job = launch{
    // Some task
}

job.cancel() // If the job is cooperative, cancel it
job.join() // If not, then wait for it's completion

```

We can also use this function to use both  cancel and join -

```kotlin
job.cancelAndJoin()
```
### <center>What makes a coroutine cooperative?</center>

Cooperative
    : A cooperative coroutine is one that respects the cancellation command of the user.

<br/>
<hr/>

## <center>Ways to make Coroutine Cooperative</center>

### <span style="color:cyan;font-variant:small-caps"> 1. Periodically invoke a suspending function that checks for cancellation</span>

- Only those suspending functions that belongs to **kotlinx.coroutines** package will make coroutine cooperative.
- _delay()_, _yield()_, _withContext()_, _withTimeout()_ etc. are suspending functions that belongs to kotlinx.coroutines package.

For example - 
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("Thread started on: ${Thread.currentThread().name}")

    val job : Job = launch {
        for(i in 1..100){
            print("${i}.")
            yield() //delay(50)
        }
    }

    delay(500)
    job.cancelAndJoin() // Cancel if respectful, continue if not
    println("Thread is working on: ${Thread.currentThread().name}")
    println("The above line may be printed in the same line as in the for loop items, " +
            "since the coroutines has been cancelled.")
}
```
<br/>

### <span style="color:cyan;font-variant:small-caps">2. Explicitly check for cancellation status within the coroutine</span>

<span style="color:white;font-weight:bold;">CoroutineScope.isActive boolean flag</span>

<span style="color:yellow;font-size:28px;">isActive</span> is a boolean flag which is an extension on the coroutine scope.
This flag can be used to check for the cancellation of coroutines.
- If it is _true_, the coroutine is active
- if false, the coroutine is inactive

Threads do not have such built-in mechanism to cancel itself internally.Check this property in long-running computation loops to support cancellation:

```kotlin
while(isActive){
    // computation logic
}
```

using isActive -
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("Thread started on: ${Thread.currentThread().name}")

    val job : Job = launch(Dispatchers.Default) {
        for(i in 1..1000){
            if(!isActive){
               return@launch //break
            }
            print("${i}-")
            Thread.sleep(1) // just to check if isActive flag works
        }
    }

    delay(2)
    job.cancelAndJoin()
    println("Thread is working on: ${Thread.currentThread().name}")
    println("The above line may be printed in the same line as in the for loop items, since the coroutines has been cancelled.")
}
```

The <span style="color:yellow">return@launch</span> when executed returns stops the executing coroutine and returns to the Dispatchers.Default(at launch coroutine - <span style="color:red">needs further explaination or correction</span>).

> Don't use Thread.sleep()

<br>
<hr/>

## <center>Handling Exceptions</center>

Cancellable suspending functions(all the suspendable functions inside **kotlinx.coroutines**) such as _yield()_, _delay())_ etc. throw <span style="color:red">CancellationException</span>  on the coroutine cancellation.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("Thread started on: ${Thread.currentThread().name}")

    val job : Job = launch(Dispatchers.Default) {
        try {
            for(i in 1..1000){
                print("${i}-")
                delay(1) //suspending function returns the CancellationException.
            }
        }
        catch (ex: CancellationException){ // Exception caught here
            println("Exception caught safely.\nException: ${ex.message}")
        }
    }

    delay(2)
    job.cancelAndJoin() // Cancellation order
}
```

Using a suspending function inside the finally block(not possible) until **withContext** is used.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("Thread started on: ${Thread.currentThread().name}")

    val job : Job = launch(Dispatchers.Default) {
        try {
            for(i in 1..1000){
                print("${i}-")
                delay(1)
            }
        }
        catch (ex: CancellationException){
            println("Exception caught safely.\nException: ${ex.message}")
        }
        finally {
            withContext(NonCancellable) {// A coroutine builder that starts another coroutine
                delay(1000) // suspendable function can't be used inside finally block unless as done here
                println("Close resource in finally.") //wont execute if withContext isn't used as it throws CancellationException
            }
        }
    }

    delay(2)
    job.cancelAndJoin()
}
```

<br/>
<hr/>

## <center>Timeouts</center>

<span style="color:cyan">withTimeout(){....}

Takes parameter - time(in millis)

**Runs a given suspending block of code inside a coroutine with a specified timeout and throws a TimeoutCancellationException if the timeout was exceeded.**

The code that is executing inside the block is cancelled on timeout and the active or next invocation of the cancellable suspending function inside the block throws a <span style="color:red">TimeoutCancellationException</span>.

The timeout event is asynchronous with respect to the code running in the block and may happen at any time, even right before the return from inside of the timeout block. Keep this in mind if you open or acquire some resource inside the block that needs closing or release outside of the block. See the Asynchronous timeout and resources [docs](https://kotlinlang.org/docs/reference/coroutines/cancellation-and-timeouts.html#asynchronous-timeout-and-resources) section of the coroutines guide for details.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("Thread started on: ${Thread.currentThread().name}")

    withTimeout(100){
        try {
            for(i in 1..500)
            {
                println("$i-")
                delay(500)
            }
        }
        catch (e:TimeoutCancellationException)
        {
            println("Exception: ${e.message}")
        }
        finally {
            println("Some code...")
        }

    }

}
```

## or
    
<br/>

<span style="color:cyan">withTimeoutOrNull(){....}

Takes parameter - time(in millis)

**Runs a given suspending block of code inside a coroutine with a specified timeout and <span style="text-decoration:underline;">returns null if this timeout was exceeded</span>.**

The code that is executing inside the block is cancelled on timeout and the active or next invocation of cancellable suspending function inside the block throws a <span style="color:red">TimeoutCancellationException</span>.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("Thread started on: ${Thread.currentThread().name}")

    val result:String? = withTimeoutOrNull(100){
            for(i in 1..500)
            {
                println("$i-")
                delay(500)
            }
        
            "This string is returned!" // Returned string(can be anything)
        }
    
    println("The following thing was returned: $result")
    
    }
```

Remember, if the job isn't completed within the time period provided inside the coroutine builder `withTimeoutOrNull()`, it returns <span style="color:red">null</span>.
