## 对象声明

If you need a _singleton_ - a class that only has got one instance - you can declare the class in the usual way, but use the `object` keyword instead of `class`:

```kotlin
object CarFactory {
    val cars = mutableListOf<Car>()
    
    fun makeCar(horsepowers: Int): Car {
        val car = Car(horsepowers)
        cars.add(car)
        return car
    }
}
```

There will only ever be one instance of this class, and the instance (which is created the first time it is accessed, in a thread-safe manner) has got the same name as the class:

```kotlin
val car = CarFactory.makeCar(150)
println(CarFactory.cars.size)
```


## 伴生对象

If you need a function or a property to be tied to a class rather than to instances of it (similar to `@staticmethod` in Python), you can declare it inside a _companion object_:

```kotlin
class Car(val horsepowers: Int) {
    companion object Factory {
        val cars = mutableListOf<Car>()

        fun makeCar(horsepowers: Int): Car {
            val car = Car(horsepowers)
            cars.add(car)
            return car
        }
    }
}
```

The companion object is a singleton, and its members can be accessed directly via the name of the containing class (although you can also insert the name of the companion object if you want to be explicit about accessing the companion object):

```kotlin
val car = Car.makeCar(150)
println(Car.Factory.cars.size)
```

In spite of this syntactical convenience, the companion object is a proper object on its own, and can have its own supertypes - and you can assign it to a variable and pass it around. If you're integrating with Java code and need a true `static` member, you can [annotate](annotations.html) a member inside a companion object with `@JvmStatic`.

A companion object is initialized when the class is loaded (typically the first time it's referenced by other code that is being executed), in a thread-safe manner. You can omit the name, in which case the name defaults to `Companion`. A class can only have one companion object, and companion objects can not be nested.

Companion objects and their members can only be accessed via the containing class name, not via instances of the containing class. Kotlin does not support class-level functions that also can be overridden in subclasses (like `@classmethod` in Python). If you try to redeclare a companion object in a subclass, you'll just shadow the one from the base class. If you need an overridable "class-level" function, make it an ordinary open function in which you do not access any instance members - you can override it in subclasses, and when you call it via an object instance, the override in the object's class will be called. It is possible, but inconvenient, to call functions via a class reference in Kotlin, so we won't cover that here.


## 对象表达式

Java only got support for function types and lambda expressions a few years ago. Previously, Java worked around this by using an interface to define a function signature and allowing an inline, anonymous definition of a class that implements the interface. This is also available in Kotlin, partly for compatibility with Java libraries and partly because it can be handy for specifying event handlers (in particular if there is more than one event type that must be listened for by the same listener object). Consider an interface or a (possibly abstract) class, as well a function that takes an instance of it:

```kotlin
interface Vehicle {
    fun drive(): String
}

fun start(vehicle: Vehicle) = println(vehicle.drive())
```

By using an _object expression_, you can now define an anonymous, unnamed class and at the same time create one instance of it, called an _anonymous object_:

```kotlin
start(object : Vehicle {
    override fun drive() = "Driving really fast"
})
```

If the supertype has a constructor, it must be invoked with parentheses after the supertype name. You can specify multiple supertypes if need be (but as usual, at most one superclass).

Since an anonymous class has no name, it can't be used as a return type - if you do return an anonymous object, the function's return type must be `Any`.

In spite of the `object` keyword being used, a new instance of the anonymous class will be created whenever the object expression is evaluated.

The body of an object expression may access, and possibly modify, the local variables of the containing scope.




---

[← 上一节：继承](inheritance.html) | [下一节：泛型 →](generics.html)


---

*本资料英文原文的作者是 [Aasmund Eldhuset](https://eldhuset.net/)；其所有权属于[可汗学院（Khan Academy）](https://www.khanacademy.org/)，授权许可为 [CC BY-NC-SA 3.0 US（署名-非商业-相同方式共享）](https://creativecommons.org/licenses/by-nc-sa/3.0/us/)。请注意，这并不是可汗学院官方产品的一部分。中文版由[灰蓝天际](https://hltj.me/)、[Yue-plus](https://github.com/Yue-plus) 译，遵循相同授权方式。*