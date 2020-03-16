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

Luckily, there is already a perfect logging mechanish out of the box - [java.util.logging.Logger](https://developer.android.com/reference/java/util/logging/Logger). And all we have to do is use it! So let's look at how to do this in Android.

The process is actually quite simple - we obtain a logger object via the static `Logger.getLogger` function, and use it to log messages with various `log` methods (that allow us specify a log level, a message and an optional `Throwable` exception). Then these messages get forwarded to registered [handlers](https://developer.android.com/reference/java/util/logging/Handler), that are free to handle these messages however they like. 

Basically, we can look at it like this - there is a ready-to-use logging interface in place, and our job is just to provide its implementation. And an important thing is that `Logger` is a java class, meaning we can use it in non-Android modules - which is exactly what we want.

First, let's write an implementation of the handler. You can place it in any Android module visible by the main `app` module, or the `app` module itself.

```kotlin
class AndroidLoggingHandler : Handler() {

    override fun isLoggable(record: LogRecord?): Boolean =
      super.isLoggable(record) && BuildConfig.DEBUG

    override fun close() {
        // ignore
    }

    override fun flush() {
        // ignore
    }

    override fun publish(record: LogRecord) {
        val tag = record.loggerName
        val level = getAndroidLevel(record.level)
        val message = record.thrown?.let { thrown ->
            "${record.message}: ${Log.getStackTraceString(thrown)}"
        } ?: record.message

        try {
            Log.println(level, tag, message)
        } catch (e: RuntimeException) {
            Log.e(this.javaClass.simpleName, "Error logging message", e)
        }
    }

    private fun getAndroidLevel(level: Level): Int =
        when (level.intValue()) {
            Level.SEVERE.intValue() -> Log.ERROR
            Level.WARNING.intValue() -> Log.WARN
            Level.INFO.intValue() -> Log.INFO
            Level.FINE.intValue() -> Log.DEBUG
            else -> Log.DEBUG
        }
}
```

In `isLoggable()` we can control if a message can be logged. Here we use the default implementation (via a call to `super.isLoggable()`) and also add an additional condition - we want to enable logging only for debug builds.

The `close()` and `flush()` methods are ignored as we don't need them.

Finally, we implement the `publish()` function, that actually does the logging. We use the beforementioned Android [Log](https://developer.android.com/reference/android/util/Log) class and its `println()` method, that expects a tag, a log level and a message. As the tag we take the name of the logger. The log level is mapped in the `getAndroidLevel()` function. And the message is composed from the log message and a stack trace string of the exception, if present. We also wrap `println()` in a try-catch block, just in case something went wrong.

Next, let's add a static function to setup the handler:
```kotlin
class AndroidLoggingHandler : Handler() {

   // ...
   
   companion object {
        fun setup() {
            val rootLogger = LogManager.getLogManager().getLogger("")
            for (handler in rootLogger.handlers) {
                rootLogger.removeHandler(handler)
            }
            rootLogger.addHandler(AndroidLoggingHandler())
            rootLogger.level = Level.FINE
        }
    }
}
```
We remove all handlers from the root logger and add our own implementation. We also specify [Level.FINE](https://developer.android.com/reference/java/util/logging/Level#FINE) as the minimum level for messages we want to log.

Now that we have our `Handler` ready, time to set it up in our [Application](https://developer.android.com/reference/android/app/Application) class:

```kotlin
class App : Application() {

    override fun onCreate() {
        super.onCreate()
        // ...
        
        AndroidLoggingHandler.setup()
    }
}
```

That's it! We can now use `Logger` from anywhere in the app:

```kotlin
class MainActivity : Activity(){

    override fun onPause() {
        super.onPause()
        Logger.getLogger("MainActivity").log(Level.FINE, "Activity is paused")
    }
}
```

...and the messages will be visible in [logcat](https://developer.android.com/studio/debug/am-logcat):
![logcat](logging-1.png)
 
However, the syntax is still not very concise and can be approved upon. So let's utilize Koltin [extensions](https://kotlinlang.org/docs/reference/extensions.html). Here are some extension functions we can write (we can place them in `domain`):

```kotlin
fun Any.logD(message: String) {
    logger.log(Level.FINE, message)
}

fun Any.logE(message: String) {
    logger.log(Level.SEVERE, message)
}

fun Any.logE(message: String, throwable: Throwable) {
    logger.log(Level.SEVERE, message, throwable)
}

private val Any.logger: Logger
    get() = Logger.getLogger(this::class.java.simpleName)
```

And now it is much more convenient and pretty:
```kotlin
class MainActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        logD("Activity is created")
        logE("Something went wrong", Throwable("some random error"))
    }
}
```
