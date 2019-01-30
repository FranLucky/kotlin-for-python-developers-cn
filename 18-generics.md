*本资料的作者是 [Aasmund Eldhuset](https://eldhuset.net/)；其所有权属于[可汗学院（Khan Academy）](https://www.khanacademy.org/)，授权许可为 [CC BY-NC-SA 3.0 US（署名-非商业-相同方式共享）](https://creativecommons.org/licenses/by-nc-sa/3.0/us/)。请注意，这并不是可汗学院官方产品的一部分。中文版由[灰蓝天际](https://hltj.me/)译，遵循相同授权方式。*

---


## 泛型类型参数

One might think that static typing would make it very impractical to make collection classes or any other class that needs to contain members whose types vary with each usage. Generics to the rescue: they allow you to specify a "placeholder" type in a class or function that must be filled in whenever the class or function is used. For example, a node in a linked list needs to contain data of some type that is not known when we write the class, so we introduce a _generic type parameter_ `T` (they are conventionally given single-letter names):

```kotlin
class TreeNode<T>(val value: T?, val next: TreeNode<T>? = null)
```

Whenever you create an instance of this class, you must specify an actual type in place of `T`, unless the compiler can infer it from the constructor parameters: `TreeNode("foo")` or `TreeNode<String>(null)`. Every use of this instance will act as if it were an instance of a class that looks like this:

```kotlin
class TreeNode<String>(val value: String?, val next: TreeNode<String>? = null)
```

Member properties and member functions inside a generic class may for the most part use the class' generic type parameters as if they were ordinary types, without having to redeclare them. It is also possible to make functions that take more generic parameters than the class does, and to make generic functions inside nongeneric classes, and to make generic top-level functions (which is what we'll do in the next example). Note the different placement of the generic type parameter in generic function declarations:

```kotlin
fun <T> makeLinkedList(vararg elements: T): TreeNode<T>? {
    var node: TreeNode<T>? = null
    for (element in elements.reversed()) {
        node = TreeNode(element, node)
    }
    return node
}
```


## 约束

You can restrict the types that can be used for a generic type parameter, by specifying that it must be an instance of a specific type or of a subclass thereof. If you've got a class or interface called `Vehicle`, you can do:

```kotlin
class TreeNode<T : Vehicle>
```

Now, you may not create a `TreeNode` of a type that is not a subclass/implementor of `Vehicle`. Inside the class, whenever you've got a value of type `T`, you may access all the public members of `Vehicle` on it.

If you want to impose additional constraints, you must use a separate `where` clause, in which case the type parameter must be a subclass of the given class (if you specify a class, and you can specify at most one) _and_ implement all the given interfaces. You may then access all the public members of all the given types whenever you've got a value of type `T`:

```kotlin
class TreeNode<T> where T : Vehicle, T : HasWheels
```


## 型变


### 简介

Pop quiz: if `Apple` is a subtype of `Fruit`, and `Bowl` is a generic container class, is `Bowl<Apple>` a subtype of `Bowl<Fruit>`? The answer is - perhaps surprisingly - _no_. The reason is that if it were a subtype, we would be able to break the type system like this:

```kotlin
fun add(bowl: Bowl<Fruit>, fruit: Fruit) = bowl.add(fruit)

val bowl = Bowl<Apple>()
add(bowl, Pear()) // Doesn't actually compile!
val apple = bowl.get() // Boom!
```

If the second-to-last line compiled, it would allow us to put a pear into what is ostensibly a bowl of only apples, and your code would explode when it tried to extract the "apple" from the bowl. However, it's frequently useful to be able to let the type hierarchy of a generic type parameter "flow" to the generic class. As we saw above, though, some care must be taken - the solution is to restrict the direction in which you can move data in and out of the generic object.


### 声明处协变与逆变

If you have an instance of `Generic<Subtype>`, and you want to refer to it as a `Generic<Supertype>`, you can safely _get_ instances of the generic type parameter from it - these will truly be instances of `Subtype` (because they come from an instance of `Generic<Subtype>`), but they will appear to you as instances of `Supertype` (because you've told the compiler that you have a `Generic<Supertype>`). This is safe; it is called _covariance_, and Kotlin lets you do _declaration-site covariance_ by putting `out` in front of the generic type parameter. If you do, you may only use that type parameter as a return type, not as a parameter type. Here is the simplest useful covariant interface:

```kotlin
interface Producer<out T> {
    fun get(): T
}
```

It is safe to treat a `Producer<Apple>` as if it were a `Producer<Fruit>` - the only thing it will ever produce is `Apple` instances, but that's okay, because an `Apple` is a `Fruit`.

Conversely, if you have an instance of `Generic<Supertype>`, and you want to refer to it as a `Generic<Subtype>` (which you can't do with nongeneric classes), you can safely _give_ instances of the generic type parameter to it - the compiler will require those instances to be of the type `Subtype`, which will be acceptable to the real instance because it can handle any `Supertype`. This is called _contravariance_, and Kotlin lets you do _declaration-site contravariance_ by putting `in` in front of the generic type parameter. If you do, you may only use that type parameter as a parameter type, not as a return type. Here is the simplest useful contravariant interface:

```kotlin
interface Consumer<in T> {
    fun add(item: T)
}
```

It is safe to treat a `Consumer<Fruit>` as a `Consumer<Apple>` - you are then restricted to only adding `Apple` instances to it, but that's okay, because it is capable of receiving any `Fruit`.

With these two interfaces, we can make a more versatile fruit bowl. The bowl itself needs to both produce and consume its generic type, so it can neither be covariant nor contravariant, but it can implement our covariant and contravariant interfaces:

```kotlin
class Bowl<T> : Producer<T>, Consumer<T> {
    private val items = mutableListOf<T>()
    override fun get(): T = items.removeAt(items.size - 1)
    override fun add(item: T) { items.add(item) }
}
```

Now, you can treat a bowl of `T` as a producer of any superclass of `T`, and as a consumer of any subclass of `T`:

```kotlin
val p: Producer<Fruit> = Bowl<Apple>()
val c: Consumer<Apple> = Bowl<Fruit>()
```


### 型变方向

If the parameters or return types of the members of a variant type are themselves variant, it gets a bit complicated. Function types in parameters and return types make it even more challenging. If you're wondering whether it's safe to use a variant type parameter `T` in a particular position, ask yourself:

* If `T` is covariant: is it okay that the user of my class thinks that `T` in this position is a `Supertype`, while in reality, it's a `Subtype`?
* If `T` is contravariant: is it okay that the user of my class thinks that `T` in this position is a `Subtype`, while in reality, it's a `Supertype`?

These considerations lead to the following rules. A covariant type parameter `T` (which the user of an object might think is `Fruit`, while the object in reality is tied to `Apple`) may be used as:

*   `val v: T`

    A read-only property type (the user is expecting a `Fruit`, and gets an `Apple`)

*   `val p: Producer<T>`

    The covariant type parameter of a read-only property type (the user is expecting a producer of `Fruit`, and gets a producer of `Apple`)

*   `fun f(): T`

    A return type (as we've already seen)

*   `fun f(): Producer<T>`

    The covariant type parameter of a return type (the user is expecting that the returned value will produce a `Fruit`, so it's okay if it really produces an `Apple`)

*   `fun f(consumer: Consumer<T>)`

    The contravariant type parameter of a parameter type (the user is passing a consumer that can handle any `Fruit`, and it will be given an `Apple`)

*   `fun f(function: (T) -> Unit)`

    The parameter type of a function-typed parameter (the user is passing a function that can handle any `Fruit`, and it will be given an `Apple`)

*   `fun f(function: (Producer<T>) -> Unit)`

    The covariant type parameter of the parameter type of a function-typed parameter (the user is passing a function that can handle any `Fruit` producer, and it will be given an `Apple` producer)

*   `fun f(function: () -> Consumer<T>)`

    The contravariant type parameter of the return type of a function-typed parameter (the user is passing a function that will return a consumer of any `Fruit`, and the returned consumer will be given `Apple` instances)

*   `fun f(): () -> T`

    The return type of a function-typed return type (the user expects the returned function to return `Fruit`, so it's okay if it really returns `Apple`)

*   `fun f(): () -> Producer<T>`

    The covariant type parameter of the return type of a function-typed return type (the user expects the returned function to return something that produces `Fruit`, so it's okay if it really produces `Apple`)

*   `fun f(): (Consumer<T>) -> Unit`

    The contravariant type parameter of a parameter of a function-typed return type (the user will call the returned function with something that can consume any `Fruit`, so it's okay to return a function that expects to receive something that can handle `Apple`)

A contravariant type parameter may be used in the converse situations. It is left as an exercise to the reader to figure out the justifications for why these member signatures are legal:

* `val c: Consumer<T>`
* `fun f(item: T)`
* `fun f(): Consumer<T>`
* `fun f(producer: Producer<T>)`
* `fun f(function: () -> T)`
* `fun f(function: () -> Producer<T>)`
* `fun f(function: (Consumer<T>) -> Unit)`
* `fun f(): (T) -> Unit`
* `fun f(): (Producer<T>) -> Unit`
* `fun f(): () -> Consumer<T>`


### 类型投影（使用处协变与逆变）

If you're using a generic class whose type parameters haven't been declared in a variant way (either because its authors didn't think of it, or because the type parameters can't have either variance kind because they are used both as parameter types and return types), you can still use it in a variant way thanks to _type projection_. The term "projection" refers to the fact that when you do this, you might restrict yourself to using only some of its members - so you're in a sense only seeing a partial, or "projected" version of the class. Let's look again at our `Bowl` class, but without the variant interfaces this time:

```kotlin
class Bowl<T> {
    private val items = mutableListOf<T>()
    fun get(): T = items.removeAt(items.size - 1)
    fun add(item: T) { items.add(item) }
}
```

Because `T` is used as a parameter type, it can't be covariant, and because it's used as a return type, it can't be contravariant. But if we only want to use the `get()` function, we can project it covariantly with `out`:

```kotlin
fun <T> moveCovariantly(from: Bowl<out T>, to: Bowl<T>) {
    to.add(from.get())
}
```

Here, we're saying that the type parameter of `from` must be a subtype of the type parameter of `to`. This function will accept e.g. a `Bowl<Apple>` as `from` and `Bowl<Fruit>` as `to`. The price we're paying for using the `out` projection is that we can't call `add()` on `from()`, since we don't know its true type parameter and we would therefore risk adding incompatible fruits to it.

We could do a similar thing with contravariant projection by using `in`:

```kotlin
fun <T> moveContravariantly(from: Bowl<T>, to: Bowl<in T>) {
    to.add(from.get())
}
```

Now, the type parameter of `to` must be a supertype of that of `from`. This time, we're losing the ability to call `get()` on `to`.

The same type parameter can be used in both covariant and contravariant projections (because it's the generic classes that are being projected, not the type parameter):

```kotlin
fun <T> moveContravariantly(from: Bowl<out T>, to: Bowl<in T>) {
    to.add(from.get())
}
```

While doing so was not useful in this particular example, one could get interesting effects by adding an unprojected parameter type `via: Bowl<T>`, in which case the generic type parameter of `via` would be forced to be "in-between" those of `from` and `to`.

If you don't have any idea (or don't care) what the generic type might be, you can use a _star-projection_:

```kotlin
fun printSize(items: List<*>) = println(items.size)
```

When using a generic type where you have star-projected one or more of its type parameters, you can:

* Use any members that don't mention the star-projected type parameter(s) at all
* Use any members that return the star-projected type parameter(s), but the return type will appear to be `Any?` (unless the type parameter is constrained, in which case you'll get the type mentioned in the constraint)
* Not use any members that take a star-projected type as a parameter


## 具体化的类型参数

Sadly, Kotlin has inherited Java's limitation on generics: they are strictly a compile-time concept - the generic type information is _erased_ at runtime. Therefore, you can not say `T()` to construct a new instance of a generic type; you can not at runtime check if an object is an instance of a generic type parameter; and if you try to cast between generic types, the compiler can't guarantee the correctness of it.

Luckily, Kotlin has got _reified type parameters_, which alleviates some of these problems. By writing `reified` in front of a generic type parameter, it does become available at runtime, and you'll get to write `T::class` to get the [class metadata](member-references-and-reflection.html#由类引用获取成员引用). You can only do this in inline functions (because an inline function will be compiled into its callsite, where the type information _is_ available at runtime), but it still goes a long way. For example, you can make an inline wrapper function for a big function that has got a less elegant signature.

In the example below, we assume that there is a `DbModel` base class, and that every subclass has got a parameterless primary constructor. In the inline function, `T` is reified, so we can get the class metadata. We pass this to the function that does the real work of talking to the database.

```kotlin
inline fun <reified T : DbModel> loadFromDb(id: String): T =
    loadFromDb(T::class, id)

fun <T : DbModel> loadFromDb(cls: KClass<T>, id: String): T {
    val entity = cls.primaryConstructor!!.call()
    val tableName = cls.simpleName
    // DB magic goes here - load from table `tableName`,
    // and use the data to populate `entity`
    // (possibly via `memberProperties`)
    return entity
}
```

Now, you can say `loadFromDb<Exercise>("x01234567")` to load an object from the `Exercise` database table.




---

[← 上一节：对象与伴生对象](objects-and-companion-objects.html) | [下一节：扩展函数/属性 →](extension-functionsproperties.html)
