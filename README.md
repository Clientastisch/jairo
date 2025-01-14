# jairo

<div align="center">
  <a href="https://www.oracle.com/java/">
    <img
      src="https://img.shields.io/badge/Written%20in-java-%23EF4041?style=for-the-badge"
      height="30"
    />
  </a>
  <a href="https://jitpack.io/#micartey/jairo/main-SNAPSHOT">
    <img
      src="https://img.shields.io/badge/jitpack-main-%2321f21?style=for-the-badge"
      height="30"
    />
  </a>
  <a href="https://micartey.github.io/jairo/docs" target="_blank">
    <img
      src="https://img.shields.io/badge/javadoc-reference-5272B4.svg?style=for-the-badge"
      height="30"
    />
  </a>
</div>

<br>

<p align="center">
  <a href="#-introduction">Introduction</a> |
  <a href="#-build-tools">Maven/Gradle</a> |
  <!-- <a href="#-troubleshooting">Troubleshooting</a> | -->
  <a href="#-getting-started">Getting started</a>
</p>

## 📚 Introduction

jairo is a custom transformer which provides an easy way to change the method body of classes. The transformer will be used through the `Instrumentation` which is provided by java using the java agent. It heavily depends on the [javaassist](https://github.com/jboss-javassist/javassist) library to change the bytecode at runtime.

### Motivation

Changing code at runtime makes a lot of things easier. E.g. if you want to trigger an event after a class is created or a method is invoked whose implementation you have no control over. Maybe you wan't to benchmark a method etc... In such cases, it could be pleasent to just rewrite implementations at runtime without having to download, decompile, edit and recompile your dependencies.


## 🔗 Build Tools

To use jairo as a dependency you might want to use a build tool like maven or gradle. 
An easy way for each and every project, is to use [jitpack](https://jitpack.io/#micartey/jairo/main-SNAPSHOT) as it makes it easy to implement and use. 

You will also need to add  [javaassist](https://mvnrepository.com/artifact/org.javassist/javassist) to your project as viro doesn't include its source code.

## 🎈 Getting started

A good way to get started, is to check out the [javadoc](https://micartey.github.io/jairo/docs) in order to get an overview of available annotations as they are the key in order to control the behavior. There are a total of [6 annotations](https://micartey.github.io/jairo/docs/me/micartey/jairo/annotation/package-summary.html) you have to know and use.

<details open>
<summary> Expand for an overview of available annotations </summary>
<br>

| Annotation | Description | Scope    | |
|------------|-------------|----------|-|
| [@Field](https://micartey.github.io/jairo/docs/me/micartey/jairo/annotation/Field.html)     | The `@Field` annotation is used to specify the parameter name at runtime. Make sure to avoid name collisions | Class |
| [@Hook](https://micartey.github.io/jairo/docs/me/micartey/jairo/annotation/Hook.html) | The `@Hook` annotation specifies on which class the injection should be applied | Class |
| [@Name](https://micartey.github.io/jairo/docs/me/micartey/jairo/annotation/Name.html) | The `@Name` annotation is used in case your method name deffers from the method you are trying to inject. | Method | optional |
| [@Overwrite](https://micartey.github.io/jairo/docs/me/micartey/jairo/annotation/Overwrite.html) | The `@Overwrite` annotation specifies which type of injection should be used. You can choose between: <ul><li>Before</li><li>After</li><li>Replace</li></ul> | Method |
| [@Parameter](https://micartey.github.io/jairo/docs/me/micartey/jairo/annotation/Parameter.html) | The `@Parameter` annotation is only used in case many methods in a class share the same name. While they can share the same name, they cannot share the same signature. By specifiying the method parameters of the target method, the right method can be injected | Method | optional |
| [@Return](https://micartey.github.io/jairo/docs/me/micartey/jairo/annotation/Return.html) | The `@Return` annotation can only be used with `Overwrite.Type.Replace` and will replace the target methods content with a return statement | Method | optional |

</details>

### Hook to a Class

In order to overwrite methods of a class, jairo needs to know on which classes it should apply (re-)transformations. This is being archived by the `@Hook` annotation as previously described.

```java
package my.test.path;

public class Target {
    ...
}
```

Let's assume Target is the target class on which we want to perform some transformations. Now create another class which has the `@Hook` annotation:

```java
package some.random.path;

@Field("targetInjectorInstance")
@Hook("my.test.path.Target")
public class TargetInjector {
    ...
}
```

As you can see, the `@Field` annotations was also added. This annotation is very important as it specifies the instance name of the global variable which will be added in order to hold an instance of `TargetInjector` inside `Target`.

The following section illustrates what is happening to the `Target` class at runtime:

```java
package my.test.path;

public class Target {
    private TargetInjector targetInjectorInstance = new TargetInjector()
    ...
}
```

### Overwrite a method

Next up is to overwrite a method. Let's assume that you have a method called `addNumbers` as follows:

```java
public int addNumbers(int a, int b) {
    return a + b;
}
```

And we want to print both numbers out before the are being calculated for some reason.

```java
@Name("addNumber")
@Overwrite(Overwrite.Type.BEFORE)
public void printNumbersBeforeAdding(Object instance, int a, int b) {
    System.out.println(String.format("a: %s b: %s", a, b));
}
```


## 💉 Injecting into a JVM

There are different ways to inject into a JVM. The best option is to specify a javaagent with the start-up command. This will ensure that the transformations will be applied, since classes don't have to be retransformed.

### Command line

```text
java -javaagent:Agent.jar -jar MyApplication.jar
```

```text
Manifest-Version: 1.0
Premain-Class: my.path.to.Agent
```

```java
public static void premain(String arguments, Instrumentation instrumentation) {
    JairoTransformer transformer = new JairoTransformer(Daddy.class);
    instrumentation.addTransformer(transformer);
}
```

### Injection

I encountered issues with re-transforming classes at runtime.
Therefore, I recommend the [Command line](#command-line) option.

```text
Manifest-Version: 1.0
Agent-Class: my.path.to.Agent
Can-Redefine-Classes: true
Can-Retransform-Classes: true
```

```java
public static void agentmain(String args, Instrumentation instrumentation) {
    JairoTransformer transformer = new JairoTransformer(Daddy.class);
    transformer.retransform(instrumentation);
}
```