## 使用空值

我们把没有任何指向的变量称为`null`（我们经常说这个变量“是空的”）。在Python里我们用`None`来表示, 在Kotlin中`null`并不是一个对象，它只是用来代表一个变量的指向是空的关键字，或者在`if`语句中做非空判断(在Kotlin中判断是否相同必须使用'=='或者'!=')。因为`null`是编程错误的一个常见来源，Kotlin鼓励尽量避免使用`null`--实际开发中变量不可以是空值，除非它被声明为允许空值，你可以在类型的后面加上`？`。例如：

```kotlin
fun test(a: String, b: String?) {
}
```

编译器会允许在调用函数时出现`test("a", "b")` 或者 `test("a", null)`，像`test(null, "b")` 或者 `test(null, null)`将不被允许。函数`test(a, b)`只有在编译器确认`a`不会为空时继续编译。在`test`函数里，编译器将会禁止调用`b`的一些方法来防止发生异常--例如你可以调用`a.length`,但是不能调用`b.length`。然而，你可以在函数里对`b`做了非空判断之后再调用被禁止的方法:

```kotlin
if (b != null) {
    println(b.length)
}
```

Or:
~~~~~~~~~~~~
```kotlin
if (b == null) {
    // Can't use members of b in here
} else {
    println(b.length)
}
```

频繁的进行非空判断会使代码不够优雅，如果必须使用`null`，Kotlin提供了几个很有用的操作符来使用可能为空的值，如下所述。

## 安全调用操作符

`x?.y`将对`x`进行非空判断，如果`x`非空，将对`x.y`进行非空判断(不重新判断`x`)，其结果将作为表达式的最后结果--否则，将会得到null。这在函数链式调用时也生效--例如`x?.y()?.z?.w()`如果`x`, `x.y()`, 和 `x.y().z`中任意一个为`null`整个函数式将返回`null`；当然，如果都不为空将正常返回函数式的结果。

## Elvis 操作符

`x ?: y` 将对 `x`进行非空判断，如果`x`为空将返回`null`，否则将返回`y`（非空类型）。我们称之为“Elvis运算符”。你可以防止`x`为空提前返回一个确定的值。

```kotlin
val z = x ?: return y
```

当`x`不为空时，`x`的值将赋给我`z`,如果`x`为空时，包含这个表达式的函数将停止并将`y`的值赋给`z`(这是合法的因为`return`也是表达式,它将返回参数的运算结果)

## 非空断言操作符

有些情况，我们可以确定某个变量是非空的，但编译器无法作出判断。我们在使用java编写逻辑比较复杂的代码时，编译器往往不能准确推理其结果，我们通过重构代码仍然无法通过编译时，我们可以采用`x!!`来声明这个变量是非空的:

```kotlin
val x: String? = javaFunctionThatYouKnowReturnsNonNull()
val y: String = x!!
```

也可以写成一个表达式: `val x = javaFunctionThatYouKnowReturnsNonNull()!!`。

`!!`将会在变量为空时抛出`NullPointerException`。当你必须使用它时需要考虑它可能会抛出一个异常（`maybeNull!!.importantFunction()`），这里有一个更好的解决方案（因为从变量名解读的信息量有限）:

```kotlin
val y: String = x ?: throw SpecificException("Useful message")
y.importantFunction()
```

The above could also be a oneliner - and note that the compiler knows that because the `throw` will prevent `y` from coming into existence if `x` is null, `y` must be non-null if we reach the line below. Contrast this with `x?.importantFunction()`, which is a no-op if `x` is null.

上面的代码也可以使用一行代码的方式--请注意，当`x`为空时，编译器知道`throw`会阻止对`y`的赋值，当运行到下一行时，说明此时`y`的值是非空的。这和`x?.importantFunction()`形成鲜明对比，当`x`为空时，将没有任何操作。


---

[← 上一节：异常](exceptions.html) | [下一节：函数式编程 →](functional-programming.html)


---

*本资料英文原文的作者是 [Aasmund Eldhuset](https://eldhuset.net/)；其所有权属于[可汗学院（Khan Academy）](https://www.khanacademy.org/)，授权许可为 [CC BY-NC-SA 3.0 US（署名-非商业-相同方式共享）](https://creativecommons.org/licenses/by-nc-sa/3.0/us/)。请注意，这并不是可汗学院官方产品的一部分。中文版由[灰蓝天际](https://hltj.me/)、[Yue-plus](https://github.com/Yue-plus) 翻译，遵循相同授权方式。*
