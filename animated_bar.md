# Easy animation with ConstraintLayout and LayoutTransition

![Image](https://github.com/FinchMoscow/AnimatedBar/blob/master/art/demo2.gif)

Some time ago we published a tiny library called [AnimatedBar](https://github.com/FinchMoscow/AnimatedBar). 
The library is quite simple and provides a view that animates visibility of its items' titles when an item is selected. 

In this post we will see how this animation can be (and was) very easily achieved with the help of two powerful Android components: [ConstraintLayout](https://developer.android.com/reference/android/support/constraint/ConstraintLayout) and [LayoutTransition](https://developer.android.com/reference/android/animation/LayoutTransition).

## Preparations

First, let's lay some foundation for our view. 

Let's say we want each item to have an icon and a title next to it. So here is a layout:
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/vPanel"
    android:layout_width="wrap_content"
    android:layout_height="match_parent"
    android:gravity="center_vertical"
    android:orientation="horizontal"
    android:padding="8dp">

    <ImageView
        android:id="@+id/ivIcon"
        android:layout_width="18dp"
        android:layout_height="18dp" />

    <TextView
        android:id="@+id/tvTitle"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginStart="8dp" />

</LinearLayout>
```
... and the view for the item:
```kotlin
class ItemView @JvmOverloads constructor(
    context: Context,
    attributeSet: AttributeSet? = null
): FrameLayout(context, attributeSet){

    init {
        inflate(context, R.layout.view_item, this)
        isSelected = false
    }

    var itemId: String = ""

    var title: String
        get() = tvTitle.text.toString()
        set(value) {
            tvTitle.text = value
        }

    var icon: Drawable? = null
        set(value) {
            field = value
            ivIcon.setImageDrawable(value)
        }
        
    override fun setSelected(selected: Boolean) {
        super.setSelected(selected)
        tvTitle.visibility = if (isSelected) View.VISIBLE else View.GONE
    }
}
```
We override the `setSelected` method so that the visibility of the title changes when the item gets selected.

Next, we need a data class to represent an item:
```kotlin
data class Item(
    val id: Long,
    val title: String,
    val icon: Drawable
)
```

And now we can create our AnimatedBar:
```kotlin
class AnimatedBar @JvmOverloads constructor(
    context: Context,
    attrs: AttributeSet? = null
) : FrameLayout(context, attrs) {

    init {
        inflate(context, R.layout.animated_bar, this)
    }

    var items: List<Item> = listOf()
        set(value) {
            field = value
            vContainer.removeAllViews()
            value.forEach { item ->
                vContainer.addView(
                    ItemView(context).apply {
                        itemId = item.id
                        title = item.title
                        icon = item.icon
                        setOnClickListener {
                            selectedItemId = item.id
                        }
                    }
                )
            }
        }

    var selectedItemId: String? = null
        set(value) {
            if (field == value) return
            field = value
            onItemViews { itemView ->
                itemView.isSelected = value != null && itemView.itemId == value
            }
        }

    private fun onItemViews(action: (ItemView) -> Unit) {
        vContainer.run {
            for (i in 0 until childCount) {
                (getChildAt(i) as? ItemView)?.run(action)
            }
        }
    }
}
```
For now we use a `LinearLayout` to lay out items:
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:id="@+id/vContainer"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal" />
```
