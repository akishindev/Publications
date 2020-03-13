# Easy logging in a multi-module Android project

No doubt, every Android delevoper is familiar with the concept of logging. We can log errors, debug messages and other useful information we might need during development. 

And although logging in Android is pretty easy and straightforward, it still can impose some challenges in a multi-module project. In this post we will look at at one way use can solve this problems.

_Note: we will use Kotlin, but it still can be easily adapted for a Java project._

## The problem

Suppose we have a multi-module project. Among our modules we have a _domain_ module, that contains business logic of the app - domain entities and interactors. And as this module does not really need to know anything about Android, we made it a pure Kotlin module (i.e. without any Android dependencies).

So, we need a logging mechanism that can be accessed in every module (including _domain_).
Let's see what options we have:

1) The first thing that comes to mind is, of course, an Android [Log](https://developer.android.com/reference/android/util/Log) class. It is a standard Android way of writing logs, that can be than [viewed in logcat](https://developer.android.com/studio/debug/am-logcat). There is a concise syntax in place and support for different log levels (error, debug, info, etc.), which is a good thing. The problem here though is that it is part of an Android library, so we can't use it in our _domain_ module (and other pure Kotlin modules we might have).

2) We might also look at Kotlin's [println](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.io/println.html) function. With this we can very easily print logs into logcat, and the messages will appear under the tag "System.out". But while we can use this method from anywhere in the app, it doesn't provide different log levels. Moreover, we might probably want to organize logging in one place, rather that having it scattered throughout the app.

3) The next thing we might try is using one of the third-party libraries, most notably [Timber](https://github.com/JakeWharton/timber). And while such a library could alleviate all of our problems, it would mean that we had to add the dependency to it in every module of our app, including our _domain_, and we would want to avoid that, especially for something as trivial as logging.

4) Finally, probably the cleanest way would be to set up a logging interface in _domain_, implement it somewhere else (for example in the _app_ module), and provide it via [dependency injection](https://developer.android.com/training/dependency-injection). But that would mean we have to inject it in every class we need logging in, which is far from convenient and seems like an overkill.

So it looks like all the options are not quite what we need...

## Logger to the rescue!



 
 
