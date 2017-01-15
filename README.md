# Async/Await
A Kotlin library for Android making writing asynchronous code simpler and more reliable using `async`/`await` approach, like:

```Kotlin
async {
   progressBar.visibility = View.VISIBLE
   // Release main thread and wait until text is loaded in background thread
   val loadedText = await { loadFromServer() }
   // Loaded successfully, come back in UI thread and show the result
   txtResult.text = loadedText
   progressBar.visibility = View.INVISIBLE
}
```
As you see in example above, you can write asynchronous code in a imperative style, step by step. Calling `await` to run code in background doesn't lock the UI thread. And execution _continues_ in UI thread after background work is finished. There is no magic, see [how it works](#how-it-works).

## Dependency
```Groovy
compile 'co.metalab.asyncawait:asyncawait:0.8'
```
Library is built upon preview version of Kotlin 1.1-M4, see [how to set it up](#how-to-setup).


## Usage
### `async`

Coroutine code has to be passed as a lambda in `async` function
```Kotlin
async {
   // Coroutine body
}
```

### `await`

Long running code has to be passed as a lambda in `await` function
```Kotlin
async {
   val result = await {
      //Long running code
   }
   // Use result
}
```
You may have many `await` calls inside `async` block, or have `await` in a loop

```Kotlin 
async {
   val repos = await { github.getRepos() }
   showList(repos)
   repos.forEach { repo ->
      val stats = await { github.getStats(repo.name) }
      showStats(repo, stats)
   }
}
```

### `awaitWithProgress`

Use it to show loading progress, its second parameter is a progress handler.
```Kotlin
val loadedText = awaitWithProgress(::loadTextWithProgress) {
         // Called in UI thread
         progressBar.progress = it
         progressBar.max = 100
      }
```
A data loading function (like the `loadTextWithProgress` above) should has a functional parameter of type `(P) -> Unit` which can be called in order to push progress value. For example, it could be like:
```Kotlin
private fun loadTextWithProgress(handleProgress: (Int) -> Unit): String {
   for (i in 1..10) {
      handleProgress(i * 100 / 10) // in %
      Thread.sleep(300)
   }
   return "Loaded Text"
}
```

### Handle exceptions using `try/catch`

```Kotlin
async {
   try {
      val loadedText = await {
         // throw exception in background thread
      }
      // Process loaded text
   } catch (e: Exception) {
      // Handle exception in UI thread
   }
}
```

### Handle exceptions in `onError` block

Could be more convenient, as resulting code has fewer indents. `onError` called only if exception hasn't been handled in `try/catch`.
```Kotlin
async {
   val loadedText = await {
      // throw exception in background thread
   }
   // Process loaded text
}.onError {
   // Handle exception in UI thread
}
```

Unhandled exceptions and exception delivered in `onError` wrapped by `AsyncException` with convenient stack trace to the place where `await` been called originally in UI thread 

### `finally` execution
`finally` always executed after calling `onError` or when the coroutine finished successfully.
```Kotlin
async {
   // Show progress
   await { }
}.onError {
   // Handle exception
}.finally {
   // Hide progress
}
```

### Safe execution

The library has `Activity.async` and `Fragment.async` extension functions to produce more safe code. So when using `async` inside Activity/Fragment, coroutine won't be resumed if `Activity` is in finishing state or `Fragment` is detached.

### Avoid memory leaks

Long running background code referencing any view/context may produce memory leaks. To avoid such memory leaks, call `async.cancelAll()` when all running coroutines referencing current object should be interrupted, like
```Kotlin
override fun onDestroy() {
      super.onDestroy()
      async.cancelAll()
}
```
The `async` is an extension property for `Any` type. So calling `[this.]async.cancelAll` intrerrupts only coroutines started by `[this.]async {}` function.

### Common extensions

The library has a convenient API to work with Retrofit and rxJava.

#### Retorift
* `awaitSuccessful(retrofit2.Call)`

Returns `Response<V>.body()` if successful, or throws `RetrofitHttpError` with error response otherwise.  
```Kotlin
async {
   reposList = awaitSuccessful(github.listRepos(userName))
}
```
#### rxJava
* await(Observable<V>)

Waits until `observable` emits first value.
```Kotlin
async {
   val observable = Observable.just("O")
   result = await(observable)
}
```

### How to create custom extensions
You can create your own `await` implementations. Here is example of rxJava extension to give you idea. Just return the result of calling `AsyncController.await` with your own lambda implementation. The code inside `await` block will be run on a background thread.
```Kotlin
suspend fun <V> AsyncController.await(observable: Observable<V>): V = this.await {
   observable.toBlocking().first()
}
```

## How it works

The library is built upon [coroutines](https://github.com/Kotlin/kotlin-coroutines/blob/master/kotlin-coroutines-informal.md) introduced in [Kotlin 1.1](https://blog.jetbrains.com/kotlin/2016/07/first-glimpse-of-kotlin-1-1-coroutines-type-aliases-and-more/), with major update made in Kotlin [1.1 M4](https://blog.jetbrains.com/kotlin/2016/12/kotlin-1-1-m04-is-here/).

The Kotlin compiler responsibility is to convert _coroutine_ (everything inside `async` block) into a state machine, where every `await` call is a non-blocking suspension point. The library is responsible for thread handling, error handling and managing state machine. When background computation is done the library delivers result back into UI thread and resumes _coroutine_ execution.

# How to setup

As for now Kotlin 1.1 is not released yet, you have to download and setup latest Early Access Preview release. 
* Go to `Tools` -> `Kotlin` -> `Configure Kotlin Plugin updates` -> `Select EAP 1.1` -> `Check for updates` and install latest one. 
* Make sure you have similar config in the main `build.gradle`
```
buildscript {
    ext.kotlin_version = '1.1-M04'
    repositories {
        ...
        maven {
            url "http://dl.bintray.com/kotlin/kotlin-eap-1.1"
        }
    }
    dependencies {
        classpath 'com.android.tools.build:gradle:2.2.3'
        classpath "org.jetbrains.kotlin:kotlin-gradle-plugin:$kotlin_version"
    }
}
```
* Make sure you have similar config at the bottom of app's `build.gradle`
```
repositories {
    mavenCentral()
    maven {
        url "http://dl.bintray.com/kotlin/kotlin-eap-1.1"
    }
}
```
and this section for getting latest kotlin-stdlib
```
dependencies {
    ...
    compile "org.jetbrains.kotlin:kotlin-stdlib:$kotlin_version"
    ...
}
```
