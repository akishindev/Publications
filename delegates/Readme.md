# Kotlin Delegates in Android

Kotlin truly is a beautiful language with some great features that make application development a fun and exciting experience. One of such features is [Delegated Properties](https://kotlinlang.org/docs/reference/delegated-properties.html). In this post we will see how delegates can make our life easier in Android development.

## Basics

First of all, what is a delegate and how does it work? Well, it's not that complicated. A delegate is just a class that provides the value for a property and handles its changes. This allows us to move, or delegate, the getter-setter logic from the property itself to a separate class, letting us reuse this logic. 

You can read more in the official [docs](https://kotlinlang.org/docs/reference/delegated-properties.html).

## Fragment arguments

We often need to pass some parameters to a fragment. It usually looks something like this:
```kotlin
class DemoFragment : Fragment() {

	private var param1: Int? = null
	private var param2: String? = null

	override fun onCreate(savedInstanceState: Bundle?) {
		super.onCreate(savedInstanceState)
		arguments?.let { args ->
			param1 = args.getInt(Args.PARAM1)
			param2 = args.getString(Args.PARAM2)
		}
	}

	companion object {
		private object Args {
			const val PARAM1 = "param1"
			const val PARAM2 = "param2"
		}

		fun newInstance(param1: Int, param2: String): DemoFragment = DemoFragment().apply {
			arguments = Bundle().apply {
				putInt(Args.PARAM1, param1)
				putString(Args.PARAM2, param2)
			}
		}
	}
}
```

So we pass the parameters when creating a fragment via its `newInstance` static method. Inside it, we put the parameters to the fragment's arguments to retrieve them later in `onCreate`.

We can make the code a little prettier, moving argument related logic to properties' getters and setters:
```kotlin
class DemoFragment : Fragment() {

	private var param1: Int?
		get() = arguments?.getInt(Args.PARAM1)
		set(value) {
			value?.let {
				arguments?.putInt(Args.PARAM1, it)
			} ?: arguments?.remove(Args.PARAM1)
		}
	private var param2: String?
		get() = arguments?.getString(Args.PARAM2)
		set(value) {
			arguments?.putString(Args.PARAM2, value)
		}

	companion object {
		private object Args {
			const val PARAM1 = "param1"
			const val PARAM2 = "param2"
		}

		fun newInstance(param1: Int, param2: String): DemoFragment = DemoFragment().apply {
			this.param1 = param1
			this.param2 = param2
		}
	}
}
```

But we still have to write basically the same code for each property, which can be a chore if we have many of them. Besides, it looks a little messy with all this explicit work with arguments.

So is there a way to further beautify the code? The answer is yes! And as you may have guessed, we will use property delegates. 

First, let's make some preparations. Fragment's arguments are stored in a `Bundle` object, that has separate methods for putting different types of values. So let's make an extension function that tries to put a value of arbitrary type to the bundle, and throws an exception if the type is not supported.
```kotlin
fun <T> Bundle.put(key: String, value: T) {
	when (value) {
		is Boolean -> putBoolean(key, value)
		is String -> putString(key, value)
		is Int -> putInt(key, value)
		is Short -> putShort(key, value)
		is Long -> putLong(key, value)
		is Byte -> putByte(key, value)
		is ByteArray -> putByteArray(key, value)
		is Char -> putChar(key, value)
		is CharArray -> putCharArray(key, value)
		is CharSequence -> putCharSequence(key, value)
		is Float -> putFloat(key, value)
		is Bundle -> putBundle(key, value)
		is Parcelable -> putParcelable(key, value)
		is Serializable -> putSerializable(key, value)
		else -> throw IllegalStateException("Type of property $key is not supported")
	}
}
```

Now we are ready to create the delegate itself:
```kotlin
class FragmentArgumentDelegate<T : Any> : ReadWriteProperty<Fragment, T> {

    @Suppress("UNCHECKED_CAST")
    override fun getValue(thisRef: Fragment, property: KProperty<*>): T =
        thisRef
            .arguments
            ?.get(property.name) as? T
            ?: throw IllegalStateException("Property ${property.name} could not be read")

    override fun setValue(thisRef: Fragment, property: KProperty<*>, value: T) {
        if (thisRef.arguments == null)
            thisRef.arguments = Bundle()

        val key = property.name

        thisRef.arguments?.put(key, value)
    }
}
```
The delegate reads property value from the fragment arguments. And when the property value changes, the delegate writes new value to the arguments (using the `Bundle.put` extension function we created before).

Notice how we use the name of the property as the key for the argument, so we don't have to store the keys as constants anymore.

Another thing to note is that we explicitly set the type as non-nullable, and throw an exception if the value cannot be read. This allows us to have non-nullable properties in our fragment, sparing us from pesky null-checks.

Sometimes we need a property to be nullable. So let's create another delegate that, if the argument is not found, doesn't throw an exception, but returns `null` instead:
```kotlin
class FragmentNullableArgumentDelegate<T : Any?> : ReadWriteProperty<Fragment, T?> {

    @Suppress("UNCHECKED_CAST")
    override fun getValue(thisRef: Fragment, property: KProperty<*>): T? =
        thisRef
            .arguments
            ?.get(property.name) as? T

    override fun setValue(thisRef: Fragment, property: KProperty<*>, value: T?) {
        if (thisRef.arguments == null)
            thisRef.arguments = Bundle()

        val key = property.name

        value?.let { thisRef.arguments?.put(key, it) } ?: thisRef.arguments?.remove(key)
    }
}
```

Next, let's make some functions for convenience (it is not necessary, just purely for aesthetic purposes):
```kotlin
fun <T : Any> delegateArg(): ReadWriteProperty<Fragment, T> =
	FragmentArgumentDelegate()

fun <T : Any> delegateArgNullable(): ReadWriteProperty<Fragment, T?> =
	FragmentNullableArgumentDelegate()
```

Finally, let's use our delegates in the fragment:
```kotlin
class DemoFragment : Fragment() {

	private var param1: Int by delegateArg()
	private var param2: String by delegateArg()

	companion object {
		
		fun newInstance(param1: Int, param2: String): DemoFragment = DemoFragment().apply {
			this.param1 = param1
			this.param2 = param2
		}
	}
}
```

Looks pretty neat, doesn't it?

## SharedPreferences delegates

Quite often we need to store some values in memory to quickly retrieve them next time the app launches. For example, we might want to store some user preferences that let users customize the app. A common way to do this is to use [SharedPreferences](https://developer.android.com/reference/android/content/SharedPreferences.html) and save key-value data in them. 

So, let's say we have some class that is responsible for saving and obtaining three parameters:
```kotlin
class Settings(context: Context) {

	private val prefs: SharedPreferences = PreferenceManager.getDefaultSharedPreferences(context)

	fun getParam1(): String? {
		return prefs.getString(PrefKeys.PARAM1, null)
	}

	fun saveParam1(param1: String?) {
		prefs.edit().putString(PrefKeys.PARAM1, param1).apply()
	}

	fun getParam2(): Int {
		return prefs.getInt(PrefKeys.PARAM2, 0)
	}

	fun saveParam2(param2: Int) {
		prefs.edit().putInt(PrefKeys.PARAM2, param2).apply()
	}

	fun getParam3(): String {
		return prefs.getString(PrefKeys.PARAM3, null) ?: DefaulsValues.PARAM3
	}

	fun saveParam3(param3: String) {
		prefs.edit().putString(PrefKeys.PARAM2, param3).apply()
	}

	companion object {
		private object PrefKeys{
			const val PARAM1 = "param1"
			const val PARAM2 = "param2"
			const val PARAM3 = "key_param3"
		}
		private object DefaulsValues{
			const val PARAM3 = "defaultParam3"
		}
	}
}
```

Here we obtain default SharedPreferences and provide methods for getting and saving the values of our parameters.

Again, we can see that we have some code duplication here. We can move some of it into private methods, of course. But it still leaves us with rather cumbersome code. Besides, what if we want to reuse this logic in some other class? Let's see how delegates can make the code a lot cleaner.

To spice things up, let's try a slightly different approach. This time, we will utilize [Object expressions](https://kotlinlang.org/docs/reference/object-declarations.html) and create extension functions for the SharedPreferences class.
```kotlin
fun SharedPreferences.string(
	defaultValue: String = "",
	key: (KProperty<*>) -> String = KProperty<*>::name
): ReadWriteProperty<Any, String> =
	object : ReadWriteProperty<Any, String> {
		override fun getValue(thisRef: Any, property: KProperty<*>) =
			getString(key(property), defaultValue)

		override fun setValue(
			thisRef: Any,
			property: KProperty<*>,
			value: String
		) = edit().putString(key(property), value).apply()
	}
```
Here we made a SharedPreferences extension function, that returns an object of an anonymous ReadWriteProperty subclass for our delegate. 

The delegate reads property value as String from preferences, using provided `key` function for preference key. By default, the key is a property name, so we don't have to keep and pass any constants. At the same time, we still have an option to pass a custom key, if, for instance, we are afraid to run into key collisions inside the preferences, or want to be able to access the value explicitly. We can also provide default value for the property, in case it is not found in the preferences.

The delegate also takes care of storing new property value in the preferences, using the same `key` function.

To make our `Settings` example work, we need to add two more delegates for String? and Int types, that work pretty much the same:
```kotlin
fun SharedPreferences.stringNullable(
	defaultValue: String? = null,
	key: (KProperty<*>) -> String = KProperty<*>::name
): ReadWriteProperty<Any, String?> =
	object : ReadWriteProperty<Any, String?> {
		override fun getValue(thisRef: Any, property: KProperty<*>) =
			getString(key(property), defaultValue)

		override fun setValue(
			thisRef: Any,
			property: KProperty<*>,
			value: String?
		) = edit().putString(key(property), value).apply()
	}
	
fun SharedPreferences.int(
	defaultValue: Int = 0,
	key: (KProperty<*>) -> String = KProperty<*>::name
): ReadWriteProperty<Any, Int> =
	object : ReadWriteProperty<Any, Int> {
		override fun getValue(thisRef: Any, property: KProperty<*>) =
			getInt(key(property), defaultValue)

		override fun setValue(
			thisRef: Any,
			property: KProperty<*>,
			value: Int
		) = edit().putInt(key(property), value).apply()
	}	
```

And now we can finally beatify our `Settings` class:
```kotlin
class Settings(context: Context) {

	private val prefs: SharedPreferences = PreferenceManager.getDefaultSharedPreferences(context)

	var param1 by prefs.stringNullable()
	var param2 by prefs.int()
	var param3 by prefs.string(
		key = { "key_param3" },
		defaultValue = "defaultParam3"
	)
}
```

Looks a lot cooler!