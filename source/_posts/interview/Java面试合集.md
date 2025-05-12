---
title: Java面试合集
category: interview
---

# String

### String底层为和将char[]改成了byte[]

在 Java 9 之前，String 的底层实现使用的是 char[]数组来存储字符数据。然而，随着 Unicode 编码的普及和多语言环境的需求增加，char 类型无法满足所有情况下的字符表示要求。因此，在 Java 9 中，String 的底层实现被修改为使用 byte[]数组来存储字符数据。

### 解释 Java 在处理字符串拼接时如何优化性能,尤其是在 Java 9 以后的版本

**Java 9 之前的优化:**
编译期优化：在 Java 9 之前，编译器对字符串拼接做了优化。对于简单的字符串常量拼接，编译器会在编译期进行优化，将其直接替换为一个单独的字符串常量。例如：

`String str = "Hello, " + "world!";`

这行代码在编译期会被优化成：

`String str = "Hello, world!";`

使用 StringBuilder 进行拼接：对于包含变量的字符串拼接，Java 编译器会自动将拼接操作转换为使用 StringBuilder 进行的拼接操作。例如：

````java
String str1 = "Hello";
String str2 = "world";
String result = str1 + ", " + str2 + "!";
````

在编译后的字节码中，上述代码会被转换为：

```java
StringBuilder sb = new StringBuilder();
sb.append(str1);
sb.append(", ");
sb.append(str2);
sb.append("!");
String result = sb.toString();
```

**Java 9 及以后的优化:**

在 Java 9 以后，JVM 引入了基于 `invokedynamic`指令的字符串拼接优化，进一步提升了性能。

`invokedynamic` 指令：Java 9 引入了`invokedynamic` 指令来优化字符串拼接。编译器在处理字符串拼接时，会生成一个调用 `invokedynamic` 指令的字节码，而不是直接使用 `StringBuilder`。 `invokedynamic` 指令在运行时决定最佳的拼接策略。

* 减少中间对象：通过动态拼接策略，减少了中间对象的创建，降低了内存使用和 GC 压力。

* 运行时优化：invokedynamic 指令允许 JVM 在运行时根据实际使用情况选择最佳策略，提高了代码的执行效率。
* 更高的灵活性：使用 invokedynamic 提供了更高的灵活性，允许未来的 JVM 优化进一步提升性能，而不需要改变应用代码。

### 描述一下字符串常量池（String Pool）的工作原理

字符串常量池（String Pool）是 Java 中用于存储字符串字面值的一种特殊内存区域。它的主要目的是为了节省内存和提高性能。以下是字符串常量池的工作原理和它如何影响字符串实例的具体描述：

* 字符串字面值的存储：
  * 当一个字符串字面值被创建时，如 String str = "Hello";，JVM 会先检查字符串常量池中是否已经存在一个值为 "Hello" 的字符串。
  * 如果常量池中已经存在这个字符串，JVM 不会创建新的字符串对象，而是直接返回常量池中的引用。这意味着相同的字符串字面值在内存中只会存储一次。
* 字符串的 intern() 方法：
  * 可以显式地将一个字符串添加到常量池中，使用 intern() 方法。例如：String str = new String("Hello").intern();
  * intern() 方法会检查常量池中是否存在一个值等于该字符串对象的字符串。如果存在，则返回池中的字符串引用；如果不存在，则将该字符串添加到常量池中，并返回该字符串的引用。

编译时的优化：编译器会自动将所有字符串字面值添加到常量池中。这意味着在编译时，字符串字面值就已经在常量池中，运行时直接使用。