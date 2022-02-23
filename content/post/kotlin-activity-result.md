+++
author = "Kaustubh Patange"
title = "Handle onActivityResult(), the Kotlin way"
cover = "images/kotlin-activity-result.png"
date = "2022-02-22"
tags = [
    "android", "kotlin"
]
+++

ActivityResult API is an improvement over the traditional `onActivityResult()` method. But it can get much better with Kotlin, let's see how?

<!--more-->

Traditionally we are used to the catch Activity results in `onActivityResult()` method overrides that Activity/Fragment has, with Jetpack Activity `v1.2.0-alpha02`, [ActivityResultRegistry](https://developer.android.com/jetpack/androidx/releases/activity#1.2.0-alpha02) was introduced which allowed us to register for activity result & made handling the result much efficiently.

The working is pretty simple, `ActivityResultRegistry` maintains a map of `key: String` to a `ActivityResultCallback` and invokes it whenever `ComponentActivity`'s `onActivityResult()` is called by first verifying the contract which was used to execute the required input & then invokes the appropriate `ActivityResultCallback` from the `map`.

## Limitations

You can only register before `onStart`, in other words with each call to `registerForActivityResult()` a `LifecycleOwner` is passed which is used to check whether you are calling this method before `onStart`, if not it throws an `IllegalStateException`.

This limitation can cause various problems where if you register the callback after a button is clicked it would simply not work. This is why I think `registerForActivityResult()` methods are not meant for listening one shot Activity Result.

## Solution

Jetpack compose does it differently, if you look at the code of [`rememberLauncherForActivityResult()`](https://cs.android.com/androidx/platform/frameworks/support/+/androidx-main:activity/activity-compose/src/main/java/androidx/activity/compose/ActivityResultRegistry.kt;l=82?q=rememberLauncherForActivityResult) you'll see it makes use of `ComponentActivity`'s `ActivityResultRegistery` to register the callback when the composition is started & unregister when the composition is disposed/abandoned (`DisposableEffect`).

We can apply this logic to construct something similar,

```kotlin
fun <I, O> ComponentActivity.launchWithResult(
    input: I,
    contract: ActivityResultContract<I, O>,
    onResult: (O) -> Unit
) {
    val contractKey = "contract_${UUID.randomUUID()}"
    var launcher: ActivityResultLauncher<I?>? = null
    launcher = activityResultRegistry.register(contractKey, contract) { result ->
        launcher?.unregister()
        onResult(result)
    }
    launcher.launch(input)
}
```

What are we doing here is simply registering a contract with a unique key & unregistering the contract when the result is obtained, thus making this a one shot listener for Activity Result.

We can built another extension function on top of it to listen for `onActivityResult()`,

```kotlin
fun ComponentActivity.startActivityWithResult(
    input: Intent,
    onActivityResult: (ActivityResult) -> Unit
) : Unit = launchContractWithResult(input, ActivityResultContracts.StartActivityForResult(), onActivityResult)
```

Now using this is pretty simple as you just've to call it wherever you want to listen for activity result for an intent maybe on a button click or something else,

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(...) {
        ...
        binding.btnLogin.setOnClickListener {
            val intent: Intent = ...
            startActivityWithResult(intent) { result ->  // <--
                // do something with the result.
            }
        }
    }
}
```

Here is full [gist of extension functions](https://gist.github.com/KaustubhPatange/bb70d2bfdacfe6cbea077f73c492e975) which you might find helpful for such use cases.

## Conclusion

- The API is not bad we just have to find such workarounds (or maybe not a workaround as Jetpack Compose is doing it) to find a solution to the problem we've.
- You can also convert this callback into a suspend function using `suspendCoroutine` so that you can use it in a `CoroutineScope` or inside a suspend function whose `coroutineContext` is `Dispatchers.Main`.

{{< css.inline >}}

<style>
    pre code, pre, code {
        white-space: pre !important;
        overflow-x: auto !important;
        word-break: keep-all !important;
        word-wrap: initial !important;
    }
    .article {
        text-align: start;
    } 
    a {
        text-decoration: underline;
    }
</style>

{{< /css.inline >}}
