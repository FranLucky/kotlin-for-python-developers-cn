## 属性引用

Consider this class: 

```kotlin
class Person(val name: String, var age: Int) {
    fun present() = "I'm $name, and I'm $age years old"
    fun greet(other: String) = "Hi, $other, I'm $name"
}
```

You can get reference to its `name` property like this:

```kotlin
val prop = Person::name
```

The result is an object which represents a reference to the property (the "Platonic ideal" property, not a property on a particular instance). There's a type hierarchy for property objects: the base interface is `KProperty`, which lets you get metadata about the property, such as its name and type. If you want to use the property object to read or modify the property's value in an object, you need to use a subinterface that specifies what kind of property it is. Immutable properties typically are `KProperty1<R, V>`, and mutable properties typically are `KMutableProperty1<R, V>`. Both of these are generic interfaces, with `R` being the receiver type (the type on which the property is declared, in this case `Person`) and `V` being the type of the property's value.

Given an `R` instance, `KProperty1<R, V>` will let you read the value that the property has in that instance by calling `get()`, and `KMutableProperty1<R, V>` will also let you change the property value in the instance by calling `set()`. Using this, we can start writing functions that manipulate properties without knowing in advance which property (or which class) they are going to deal with:

```kotlin
fun <T> printProperty(instance: T, prop: KProperty1<T, *>) {
    println("${prop.name} = ${prop.get(instance)}")
}

fun <T> incrementProperty(
    instance: T, prop: KMutableProperty1<T, Int>
) {
    val value = prop.get(instance)
    prop.set(instance, value + 1)
}

val person = Person("Lisa", 23)
printProperty(person, Person::name)
incrementProperty(person, Person::age)
```

You can also get a reference to a top-level property by just prefixing the property name with `::` (e.g. `::foo`), and its type will be `KProperty0<V>` or `KMutableProperty0<V>`.


## 函数引用

Functions act similarly to properties, but can be referenced as two different kinds of types.

If you want to look at the metadata of a function (e.g. its name), use `KFunction<V>` or one of its subinterfaces, where `V` is the function's return type. Here's a basic example:

```kotlin
val person = Person("Lisa", 32)
val g: KFunction<String> = Person::greet
println(g.name)
println(g.call(person, "Anne"))
```

Invoking `call()` on a function object will call the function. If it is a member function, the first parameter must be the _receiver_ (the object on which the function is to be invoked, in this case `person`), and the remaining parameters must be the ordinary function parameters (in this case `"Anne"`).

Since the parameter types are not encoded as generic type parameters in `KFunction<V>`, you won't get compile-time type validation of the parameters you pass. In order to encode the parameter types, use one of the subinterfaces `KFunction1<A, V>`, `KFunction2<A, B, V>`, `KFunction3<A, B, C, V>`, and so on, depending on how many parameters the function has got. Keep in mind that if you are referencing a member function, the first generic type parameter is the receiver type. For example, `KFunction3<A, B, C, V>` may reference either an ordinary function that takes `A`, `B`, and `C` as parameters and returns `V`, or it may reference a member function on `A` that takes `B` and `C` as parameters and returns `V`. When you use any of these types, you can call the function through its reference as if the reference were a function, e.g. `function(a, b)`, and this call will be type-safe.

You can also reference a member property directly on an object, in which case you get a member function reference that is already bound to its receiver, so that you don't need the receiver type in the signature. Here's an example of both approaches:

```kotlin
fun <A, V> callAndPrintOneParam(function: KFunction1<A, V>, a: A): V {
    val result = function(a)
    println("${function.name}($a) = $result")
    return result
}

fun <A, B, V> callAndPrintTwoParam(function: KFunction2<A, B, V>, a: A, b: B): V {
    val result = function(a, b)
    println("${function.name}($a, $b) = $result")
    return result
}

val p = Person("Lisa", 32)
callAndPrintOneParam(p::greet, "Alice")
callAndPrintTwoParam(Person::greet, person, "Lisa")
```

If you only want to call the function and don't care about the metadata, use a function type, e.g. `(A, B) -> V` for an ordinary function reference or a bound member function reference, or `A.(B, C) -> V` for an unbound member function reference on `A`. Note that `KFunction<V>` and its subinterfaces are only available for declared functions (obtained either by explicitly referencing it in the code, or through reflection, as shown later) - only function types are available for function literals (lambda expressions or anonymous functions).

You can get a reference to an top-level function by prefixing the function name with `::` (e.g. `::foo`).


## 由类引用获取成员引用

While it is possible in Kotlin to dynamically create new classes at runtime or to add members to a class, it's tricky and slow, and generally discouraged. However, it is easy to dynamically inspect an object to see e.g. what properties and functions it contains and which annotations exist on them. This is called _reflection_, and it's not very performant, so avoid it unless you really need it.

Kotlin has got its own reflection library (`kotlin-reflect.jar` must be included in your build). When targeting the JVM, you can also use the Java reflection facilities. Note that the Kotlin reflection isn't quite feature-complete yet - in particular, you can't use it to inspect built-in classes like `String`.

Warning: using reflection is usually the wrong way to solve problems in Kotlin! In particular, if you have several classes that all have some common properties/functions and you want to write a function that can take an instance of any of those classes and use those properties, the correct approach is to define an interface with the common properties/functions and make all the relevant classes implement it; the function can then take that interface as a parameter. If you don't control those classes, you can use the [Adapter pattern](https://en.wikipedia.org/wiki/Adapter_pattern) and write wrapper classes that implement the interface - this is very easy thanks to Kotlin's [delegation feature](inheritance.html#委托). You can also get a lot of leverage out of using generics in clever ways.

Appending `::class` to a class name will give you a `KClass<C>` metadata object for that class. The generic type parameter `C` is the class itself, so you can use `KClass<*>` if you're writing a function that can work with metadata for any class, or you can make a generic function with a type parameter `T` and parameter type `KClass<T>`. From this, you can obtain references to the members of the class. The most interesting properties on `KClass` are probably `primaryConstructor`, `constructors`, `memberProperties`, `declaredMemberProperties`, `memberFunctions`, and `declaredMemberFunctions`. The difference between e.g. `memberProperties` and `declaredMemberProperties` is that the former includes inherited properties, while the latter only includes the properties that have been declared in the class' own body.

In this example, using `Person` and `callAndPrintTwoParam()` from the previous section, we locate a member function reference by name and call it:

```kotlin
val f = Person::class.memberFunctions.single { it.name == "greet" } as KFunction2<Person, String, String>
callAndPrintTwoParam(f, person, "Lisa")
```

The signature of `greet()` is `KFunction2<Person, String, String>` because it's a function on `Person` that takes a `String` and returns a `String`.

Constructor references are effectively factory functions for creating new instances of a class, which might come in handy:

```kotlin
val ctor = Person::class.primaryConstructor!! as (String, Int) -> Person
val newPerson = ctor("Karen", 45)
```


## Java 风格反射

If you're targeting the JVM platform, you can also use Java's reflection system directly. In this example, we grab a function reference from an object's class by specifying the function's name as a string (if the function takes parameters, you also need to specify their types), and then we call it. Note that we didn't mention `String` anywhere - this technique works without knowing what the object's class is, but it will raise an exception if the object's class doesn't have the requested function. However, Java-style function references do not have type information, so you won't get verification of the parameter types, and you must cast the return value:

```kotlin
val s = "Hello world"
val length = s.javaClass.getMethod("length")
val x = length.invoke(s) as Int
```

If you don't have an instance of the class, you can get the class metadata with `String::class.java` (but you can't invoke any of its members until you have an instance).

If you need to look up the class dynamically as well, you can use `Class.forName()` and supply the fully-qualified name of the class.




---

[← 上一节：扩展函数/属性](extension-functionsproperties.html) | [下一节：注解 →](annotations.html)


---

*本资料英文原文的作者是 [Aasmund Eldhuset](https://eldhuset.net/)；其所有权属于[可汗学院（Khan Academy）](https://www.khanacademy.org/)，授权许可为 [CC BY-NC-SA 3.0 US（署名-非商业-相同方式共享）](https://creativecommons.org/licenses/by-nc-sa/3.0/us/)。请注意，这并不是可汗学院官方产品的一部分。中文版由[灰蓝天际](https://hltj.me/)、[Yue-plus](https://github.com/Yue-plus) 翻译，遵循相同授权方式。*