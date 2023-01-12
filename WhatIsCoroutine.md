# Coroutines (Chapter 1)
A coroutine is an instance of suspendable computation. It is conceptually similar to a thread, in the sense that it takes a block of code to run that works concurrently with the rest of the code. However, a coroutine is not bound to any particular thread. It may suspend its execution in one thread and resume in another one.

An app when opened by the user becomes a thread, a thread is a sequentially executing instruction set. Any  i/O operation that is performed inside the app, works on the **Main Thread**. A main thread handles fast and small instructions. But if we burden it with long operations, our app might become unusable as it will wait for the longer operation to complete until it executes the rest of the code.

![M2s2.png](https://img.photouploads.com/file/PhotoUploads-com/M2s2.png)


So to counter this, we can make use of a worker thread(or many), individually on which we will be executing a set of instruction. This is **Multithreading**

![M2sK.png](https://img.photouploads.com/file/PhotoUploads-com/M2sK.png)

But, there is a limit on how many threads we can create for a single application as threads are resource intensive operations, hence something other than this must be done.

Instead of using multithreading, we can use Coroutines. A coroutines initialises a background thread, on which multiple coroutines can run(File upload, Network Request, Image loading, Database query etc.).

## Coroutines - Definition

A coroutine can be explained as a Lightweight thread. Just like thread, many coroutine can run in parallel, wait for each other and communicate with each other. But **Coroutines != Threads**. Coroutines are very low memory expensive, and many coroutines can run in a single app without memory issue. 

---

## Using coroutines

Before using the coroutines, we need to add it into the build file(build.gradle.kts  if the system is gradle), inside **dependencies** section.

```gradle
implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.6.4")
```
Then sync the file. And now we can use kotlin coroutine. Also, add "jcenter()" inside **repositories**.

### Creating a thread
```kotlin
 thread { // Creates a background thread
        println("The thread is still: ${Thread.currentThread().name} and ID: ${Thread.currentThread().id}")
        Thread.sleep(1000L) // Waits for passed time, dummy task period
        println("fake work")
    }
```
This is a different thread (not the main thread).

Full file -
```kotlin
import kotlin.concurrent.thread

fun main(){
    println("Current thread: ${Thread.currentThread().name} and ID: ${Thread.currentThread().id}")

    thread { // new thread
        println("The thread is still: ${Thread.currentThread().name} and ID: ${Thread.currentThread().id}")
        Thread.sleep(1000L) 
        println("fake work")
    }

    println("The thread is still: ${Thread.currentThread().name}")
}
```
The first println() of the main thread is executed independently of the execution of new thread. It totally depends upon CPU to process a task in any way it can.

_The code in the background thread willn't blocked the code in the main thread._

### Creating a coroutine

Unlike for a thread, the application doesn't stop and wait for the coroutines to end. It ends with the main thread, whether or not the coroutine has completed or not.

```kotlin
// Instead of thread block, use this -
GlobalScope.launch{ // suppose on thread T1
    println("Fake work on ${Thread.currentThread().name} started.") // executes on T1
    delay(1000) // Thread T1 is free
    println("Fake work on ${Thread.currentThread().name} ended.") // may execute on T1 or any other thread
}

runBlocking{
    delay(2000)
}
```

Unlike the Thread.sleep() function, which blocks the thread on which it is executing, the delay() doesn't, hence it won't delay the execution of other coroutines on the same thread.

The **delay()** is a suspend function(hence _suspend_ is added before fun like this - "suspend fun delay(...){...}"), a suspend function is a function which can only be called inside a coroutine or inside another suspend function.

The different between GlobalScope.launch{...} and runBlocking{...} is that the launch{} doesn't suspend the thread it is executing on(main thread in this case) while the runBlocking{...} stops the execution of the thread it is executing on.

```kotlin

import kotlinx.coroutines.*

@OptIn(DelicateCoroutinesApi::class)
fun main(){
    runBlocking {
        println("Current thread: ${Thread.currentThread().name} and ID: ${Thread.currentThread().id}")

        GlobalScope.launch { // Creates a background thread
            println("Fake work started at : ${Thread.currentThread().name} and ID: ${Thread.currentThread().id}")
            delay(1000) // Waits for passed time, just a dummy task period
            println("fake work finished!")
        }

        delay(2000) // not a practical way
    
        println("The thread is still: ${Thread.currentThread().name}")
    }
}
```
### or


```kotlin

import kotlinx.coroutines.*

@OptIn(DelicateCoroutinesApi::class)
fun main() = runBlocking {
        println("Current thread: ${Thread.currentThread().name} and ID: ${Thread.currentThread().id}")

        GlobalScope.launch { // Creates a background thread
            println("Fake work started at : ${Thread.currentThread().name} and ID: ${Thread.currentThread().id}")
            delay(1000) // Waits for passed time, just a dummy task period
            println("fake work finished!")
        }

        delay(2000) // not a practical way
    
        println("The thread is still: ${Thread.currentThread().name}")
    }

```

**The GlobalScope.launch and runBlocking both create coroutines.**

