# <center>Coroutine</center>

<span style="font-variant:small-caps">CoroutineScope, CoroutineContext and Dispatchers</span>

**Every coroutine has its own coroutine scope instance attached to it. Using different coroutine builders will result in different types of instances of CoroutineScope Interface.**

```kotlin
launch { //this : CoroutineScope
    // 'this' is accessible inside this block  
}

async { //this : CoroutineScope
    // Some operation
}

runBlocking{ //this : CoroutineScope
    // Some operation
}
```

The signature of the coroutine scope tell about what kind of coroutine is getting executed.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    println("runBlocking: $this")
}
```

will show output like -
> runBlocking: BlockingCoroutine{Active}@d9c1e9ad

The coroutine built was a `Blocking coroutine`, which is currently `active` and has a Hash Code of `d9c1e9ad`.


The runBlocking, launch and async creates different instance of *CoroutineScope*.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    println("runBlocking: $this")

    launch {
        println("launch: $this")

        launch {
            println("child launch: $this")
        }
    }

    async {
        println("async: $this")
    }

    println("......Some code......")
}
```

Output will be like this -

```
runBlocking: BlockingCoroutine{Active}@b567f452
......Some code......
launch: StandaloneCoroutine{Active}@5198bfdb
async: DeferredCoroutine{Active}@d04a8881
child launch: StandaloneCoroutine{Active}@e2b11791
```
A `launch{...}` will create the coroutine instance of *StandaloneCoroutine*, while the `async{...}` will create an instance of *DefferedCoroutine* and `runBlocking{...}` will create an instance of *BlockingCoroutine*.

The child launch coroutine is also having different Instance of the CoroutineScope than its parent.

<hr/>

## CoroutineContext

<a href="https://ibb.co/S75WxPG"><img src="https://i.ibb.co/Qjpzcft/5.png" alt="5" border="0"></a>

Coroutines always execute in some context represented by a value of the CoroutineContext type, defined in the Kotlin standard library.
The coroutine context is a set of various elements. The main elements are the Job of the coroutine, which we've seen before, and its dispatcher.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {

    // this: CoroutineScope instance
    // coroutineContext: CoroutineContext instance

    /* Without Parameter: CONFINED      [CONFINED DISPATCHER]
        - Inherits CoroutineContext from immediate parent coroutine.
        - Even after delay() or suspending function, it continues to run in the same thread.  */
    launch {
        println("C1: ${Thread.currentThread().name}")       // Thread: main
        delay(1000)
        println("C1 after delay: ${Thread.currentThread().name}")   // Thread: main
    }

    /* With parameter: Dispatchers.Default [similar to GlobalScope.launch { } ]
        - Gets its own context at Global level. Executes in a separate background thread.
        - After delay() or suspending function execution,
            it continues to run either in the same thread or some other thread.  */
    launch(Dispatchers.Default) {
        println("C2: ${Thread.currentThread().name}")   // Thread: T1
        delay(1000)
        println("C2 after delay: ${Thread.currentThread().name}")   // Thread: Either T1 or some other thread
    }

    /*  With parameter: Dispatchers.Unconfined      [UNCONFINED DISPATCHER]
        - Inherits CoroutineContext from the immediate parent coroutine.
        - After delay() or suspending function execution, it continues to run in some other thread.  */
    launch(Dispatchers.Unconfined) {
        println("C3: ${Thread.currentThread().name}")   // Thread: main
        delay(1000)
        println("C3 after delay: ${Thread.currentThread().name}")   // Thread: some other thread T1
    }

    launch(coroutineContext) {
        println("C4: ${Thread.currentThread().name}")       // Thread: main
        delay(1000)
        println("C4 after delay: ${Thread.currentThread().name}")   // Thread: main 
    }

    println("...Main Program...")
}
```

<br/>

## <center>Dispatchers and threads</center>

The coroutine context includes a coroutine dispatcher (CoroutineDispatcher) that determines what thread or threads the corresponding coroutine uses for its execution. The coroutine dispatcher can confine coroutine execution to a specific thread, dispatch it to a thread pool, or let it run unconfined.

All coroutine builders like `launch` and `async` accept an *optional* `CoroutineContext` parameter that can be used to explicitly specify the dispatcher for the new coroutine and other context elements.

```kotlin
fun main() = runBlocking{
 launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking      : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined            : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(Dispatchers.Default) { // will get dispatched to DefaultDispatcher 
        println("Default               : I'm working in thread ${Thread.currentThread().name}")
    }
    launch(newSingleThreadContext("MyOwnThread")) { // will get its own new thread
        println("newSingleThreadContext: I'm working in thread ${Thread.currentThread().name}")
    }
}
```

Output(maybe in different order)-

```
Unconfined            : I'm working in thread main
Default               : I'm working in thread DefaultDispatcher-worker-1
newSingleThreadContext: I'm working in thread MyOwnThread
main runBlocking      : I'm working in thread main
```

1. When launch { ... } is used without parameters, it inherits the context (and thus dispatcher) from the CoroutineScope it is being launched from. In this case, it inherits the context of the main runBlocking coroutine which runs in the main thread.

2. Dispatchers.Unconfined is a special dispatcher that also appears to run in the main thread, but it is, in fact, a different mechanism that is explained later.

3. The default dispatcher is used when no other dispatcher is explicitly specified in the scope. It is represented by Dispatchers.Default and uses a shared background pool of threads.

4. `newSingleThreadContext` creates a thread for the coroutine to run. <span style="color:red">A dedicated thread is a very expensive resource.</span> In a real application it must be either released, when no longer needed, using the close function, or stored in a top-level variable and reused throughout the application.

<br/>

### <span style="text-decoration:underline">Unconfined vs confined dispatcher</span>

**Unconfined**: The `Dispatchers.Unconfined` coroutine dispatcher <span style="text-decoration:underline">starts a coroutine in the caller thread, but only until the first suspension point.</span> After suspension it resumes the coroutine in the thread that is fully determined by the suspending function that was invoked. The unconfined dispatcher is appropriate for coroutines which neither consume CPU time nor update any shared data (like UI) confined to a specific thread.

**Confined**: The dispatcher is inherited from the outer CoroutineScope by default. The default dispatcher for the runBlocking coroutine, in particular, is confined to the invoker thread, so inheriting it has the effect of confining execution to this thread with predictable *FIFO scheduling*.

```kotlin
fun main() = runBlocking<Unit> {
    launch(Dispatchers.Unconfined) { // not confined -- will work with main thread
        println("Unconfined      : I'm working in thread ${Thread.currentThread().name}")
        delay(500)
        println("Unconfined      : After delay in thread ${Thread.currentThread().name}")
    }

    launch { // context of the parent, main runBlocking coroutine
        println("main runBlocking: I'm working in thread ${Thread.currentThread().name}")
        delay(1000)
        println("main runBlocking: After delay in thread ${Thread.currentThread().name}")
    }    
}
```
So, the coroutine with the context inherited from `runBlocking {...}` continues to execute in the main thread, while the unconfined one resumes in the default executor thread that the delay function is using.

<br/>

## <center>Job in the Context</center>

The coroutine's Job is part of its context, and can be retrieved from it using the `coroutineContext[Job]` expression:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    println("My job is ${coroutineContext[Job]}")    
}
```

Output -
```
My job is BlockingCoroutine{Active}@d1dd940c
```

Note that `isActive` in CoroutineScope is just a convenient shortcut for `coroutineContext[Job]?.isActive == true`.


<br/>

### <span style="text-decoration:underline;">Children of a coroutine</span>

When a coroutine is launched in the CoroutineScope of another coroutine, it inherits its context via `CoroutineScope.coroutineContext` and the Job of the new coroutine becomes a child of the parent coroutine's job. When the parent coroutine is cancelled, all its children are recursively cancelled, too.

However, this parent-child relation can be explicitly overriden in one of two ways:

    1. When a different scope is explicitly specified when launching a coroutine (for example, GlobalScope.launch), then it does not inherit a Job from the parent scope.

    2. When a different Job object is passed as the context for the new coroutine (as shown in the example below), then it overrides the Job of the parent scope.

In both cases, the launched coroutine is not tied to the scope it was launched from and operates independently.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {
    // launch a coroutine to process some kind of incoming request
    val request = launch {
        // it spawns two other jobs
        launch(Job()) {
            println("job1: I run in my own Job and execute independently!")
            delay(1000)
            println("job1: I am not affected by cancellation of the request")
        }
        // and the other inherits the parent context
        launch {
            delay(100)
            println("job2: I am a child of the request coroutine")
            delay(1000)
            println("job2: I will not execute this line if my parent request is cancelled") // Total delay:1000ms, will get cancelled because the job is cancelled in 500ms
        }
    }
    delay(500)
    request.cancel() // cancel processing of the request
    println("main: Who has survived request cancellation?")
    delay(1000) // delay the main thread for a second to see what happens
}
```

Output-

```
job1: I run in my own Job and execute independently!
job2: I am a child of the request coroutine
main: Who has survived request cancellation?
job1: I am not affected by cancellation of the request
```

<br/>

### <span style="text-decoration:underline;">Parental responsibilities</span>

A parent coroutine always waits for completion of all its children. A parent does not have to explicitly track all the children it launches, and it does not have to use Job.join to wait for them at the end:

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    val request = launch {
        repeat(3) { i -> // launch a few children jobs
            launch  {
                delay((i + 1) * 1000L) // variable delay 200ms, 400ms, 600ms
                println("Coroutine $i is done")
            }
        }
        println("request: I'm done and I don't explicitly join my children that are still active")
    }
    request.join() //remember
    println("Now processing of the request is completed.")
}
```

Output -
```
request: I'm done and I don't explicitly join my children that are still active
Coroutine 0 is done
Coroutine 1 is done
Coroutine 2 is done
Now processing of the request is complete
```

<br/>

### <span style="text-decoration:underline;">Naming coroutines for debugging</span>

Automatically assigned ids are good when coroutines log often and you just need to correlate log records coming from the same coroutine. However, when a coroutine is tied to the processing of a specific request or doing some specific background task, it is better to name it explicitly for debugging purposes. The `CoroutineName` context element serves the same purpose as the thread name. It is included in the thread name that is executing this coroutine when the debugging mode is turned on.

```kotlin
import kotlinx.coroutines.*

fun log(msg: String) = println("[${Thread.currentThread().name}] $msg")

fun main() = runBlocking(CoroutineName("main")) {
    log("Started main coroutine")
    // run two background value computations
    val v1 = async(CoroutineName("v1coroutine")) {
        delay(500)
        log("Computing v1")
        252
    }
    val v2 = async(CoroutineName("v2coroutine")) {
        delay(1000)
        log("Computing v2")
        6
    }
    log("The answer for v1 / v2 = ${v1.await() / v2.await()}")    
}
```

> Don't forget to include `-Dkotlinx.coroutines.debug` in the JVM options(Run -> Edit Configuration -> VM options)

<br/>

### <span style="text-decoration:underline;">Combining context elements</span>

We can combine multiple values for the parameters using the `+` operator.
```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking<Unit> {
    launch(Dispatchers.Default + CoroutineName("myCoroutine")) {
        println("I'm working in thread ${Thread.currentThread().name}")
    }    
}
```

Output -

```kotlin
I'm working in thread DefaultDispatcher-worker-1 @myCoroutine#2
```

