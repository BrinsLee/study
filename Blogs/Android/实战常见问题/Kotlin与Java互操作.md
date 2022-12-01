## Kotlin 调用Java

### 可空性

Java默认实现是可空性的，所以kotlin调用java方法时不知道会不会收到空值，所以在kotlin中添加?或! 提示kotlin可能出现空值

例如

```java
public Set<String> toSet(Collection<String> elements){
    //TODO
}
```

那么Kotlin在调用的时候是不能确定输入和输出是否可为空的。就需要使用？或者 ！来辅助判断。为了方便Kotlin调用，我们通常使用 @NotNull 注解来标识Java代码的非原始参数、字段、返回值。

```java
@NotNull
Set<@NotNull String> toSet(@NotNull Collection<@NotNull String> elements){
    //TODO
}
```



### 前缀属性（getter、setter）

如果是使用Java bean，那么我们在Kotlin中调用就没有什么问题。

如果你的空参数方法是以get开头的，那么Kotlin就知道这是getter，就可以通过属性名来访问它。
相同的如果是由set开头的单一参数方法，那么Kotlin就知道这是setter，就通过属性名直接赋值。
当然is的工作原理也是和它们类似的。

### 关键字（keywords）

kotlin中有很多系统定义的关键字如 fun is in objects、typeof、val、var、when、typealias等。

这些关键字在java是可以被使用的，但是在kotlin却是不行的。

函数或者参数使用了这些关键字，那么kotlin在调用的时候会出现一些问题，比如Java中定义了一个方法名叫 is 的方法。那么在Kotlin中直接调用就会报错。

那么最简单的方法就是重命名Java方法，但如果调用的是三方库的方法，就很难去重命名了。
所以我们另一种解决方式是在Kotlin调用java方法的时候加上 `` 反引号来使用。

```kotlin
 Utils.`is`()
```



## Java调用Kotlin

### @JvmName & @JvmMultifileClass

### JvmField

在 kotlin中我们使用的数据类即 data class 是不需要指定getter和setter的，可以直接通过字段名来访问它们。但是如果是在Java文件中调用data class依旧是需要使用getter和setter方法进行调用的。这里我们是可以修改他们的，那就是使用 @JvmField 注解，通过注解，可以直接将字段暴露出去进行访问。

```java
data class Person(
 
    @JvmField var name: String,
    @JvmField var age: Int
)
 
//java中调用
Person person = new Person("",1);
person.name = "";
person.age = 10;
```

但是也有例外就是lateinit修饰的字段会自动暴露，无需指定@JvmField注解。还有const修饰的字段也是一样会自动暴露。

### JvmStatic

当我们把Java文件的静态方法迁移到Kotlin时，我们会将其放在 companion object中，但是这样处理之后在Java文件中无法直接调用，得通过companion对象实例方法来调用。

可以使用@JvmStatic注解，他会让Kotlin编译器完成类的封装后生成一个静态方法

```kotlin
class MyService {
    internal fun doWork() {
        ...
    }
 
    companion object {
        @JvmStatic
        fun schedule(context: Context) {
            ...
        }
    }
}
 
//在Java中调用
MyService.schedule(this);
```

### JvmOverloads

在Kotlin中我们可以给函数的参数设置默认值，即默认参数。但是这个功能在Java中是没有的。如果不做任何处理，那么在Java中调用函数的时候，就必须每个参数都要传入。那么我们设置的默认参数就没有任何意义了。

所以，Kotlin给我们提供了 @JvmOverloads注解，使用这个注解后，会让Kotlin编译器按照从左向右的顺序依次为每一个可选参数生成重载。