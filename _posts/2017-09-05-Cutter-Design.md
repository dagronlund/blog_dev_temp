---
layout: single
title: Cutter, A Minimal Java AOP Library (Part 2 -  High Level Design) 
---
In [part 1](https://spaceman1701.github.io/2017/08/31/Cutter-Goals/), I described the motivation for Cutter and many of the broad design goals. This post is going to go in depth into Cutter's actual design. If you've read the previous post (and I suggest you do) or looked at the [Cutter](https://www.github.com/spaceman1701/cutter/) git repository, then you know that Cutter is implemented using [Java's Pluggable Annotation API](https://www.jcp.org/en/jsr/detail?id=269). There were two motivations for doing this: First, it allowed Cutter to be written using standard Java. Second, it allows Cutter to be used by simply placing the cutter-lib jar in the class-path.
# Pluggable Annotation API
While the Java Pluggable Annotation API is very powerful, it has some major limitations. The biggest of which is that it doesn't explicitly allow for existing Java files to be modified. This poses a major problem for a library like Cutter because of the requirement that Cutter integrates seamlessly. This means that the user shouldn't be expected to do anything special with their code to use Cutter. It also means the Cutter shouldn't interfere with debugging or interoperability. 

Fortunately, the people behind [Project Lombok](https://projectlombok.org/) discovered that (at least in Oracle JDKs) changes made to the Abstract Syntax Tree during the annotation processing phase are reflected in the compiled code. This allows for arbitrary modification of Java files before compilation. Unfortunately, the Sun Tree APIs are poorly documented and difficult to use. The difficulty is compounded by the fact that in Cutter, a Cut must be invisible to at both the declaration and call site.
# High Level Design
The key design goal of Cutter from a user-facing standpoint is that code that calls a Cut method should not have to know anything about the Cut. Furthermore, external libraries that call Cut code may not be able to be changed to know about the Cut.

This makes call site mutation very unrealistic. At best, a solution based on call site manipulation would require slowly searching out every reference to a Cut method and making appropriate modifications. At worst it's completely impossible because the calling code is already compiled. 

Cutter instead uses declaration site modification. It creates a facade method (in some cases AspectJ does this too) which takes the place of the Cut method. The new method is responsible for creating the Aspect object declared in the Cut annotation and for calling the original wrapped method. The old method is made private, but can still be called by it's wrapper because they belong to the same class.

Since this design doesn't require reflection at all, there's no chance of interfering with the security policy. Standard method calls are also very fast in Java (and Cutter uses direct references to all the classes it uses, meaning that there isn't even overhead from interface method invocations vs standard invocations).  This design is also minimally invasive (and with some slight-of-hand byte-code consistency for debugging can be preserved). 

So, in essence, Cutter takes the following code:
```java
/* The code that gets written */
class Foo {
	@Cut(ExampleAdvice.class)
	int bar(String name) {
		return name.length();
	}
}
```
And covert's it to:
```java
/* The code that gets compiled */
class Foo {
    int bar(String name) {
        ExampleAdvice advice = new ExampleDevice(...);
        if (advice.before()) {
            return advice.after(__wrapped_bar(advice.getParameterValue(0)));
        }
        return advice.onSkip();
    }

    int __wrapped__bar(String name) {
        return name.length();
    }
}
```
Cutter also has a fairly simple (and still changing) packaging scheme. Cutter has two artifacts `cutter-compile` and `cutter-lib`. Cutter-compile contains all the necessary components for the annotation processor (and the annotation processor itself). Cutter-lib is full library that contains the META-INF entry for the Cutter annotation processor and some basic extension classes (like VoidAdvice for convenient use of Cuts on void methods). This packaging scheme is designed with two priorities. The first is that the each package has its own domain. The second is to place roadblocks up to protected from feature-creep. Adding a feature to `cutter-compile` requires explicit thought and ceremony. This forces me (and maybe other developers in the future) to actually think about what's being added to the compile library.
# Problems With the Design
Now, if you've been paying attention, you may have come up with a laundry list of issues with the design I just laid out. Beyond that, the example code I wrote above has some concerning hand-wavy "..."s in it. Seems problematic.
Lets first work out the hand-waviness and come up with a more specific example:
 The original class:
```java
class Foo {
    @Cut(ExampleAdvice.class)
    int doBar(String bar, int i, float f, Object o) {
        System.out.println("bar is: " + bar);
        System.out.println("i is: " + i);
        System.out.println("f is: " + f);
        System.out.println("o is: " + o);
    }
}
```
After Cutter runs:
```java
class Foo {
    int doBar(String bar, int i, float f, Object o) {
        ExampleAdvice advice = new ExampleDevice(
            Cut.Utils.getJoinPoint(Foo.class, "doBar");
            Cut.Utils.getParameterArray(new Object[]{"bar", String.class, bar,
                            "i", int.class, i,
                            "f", float.class, f,
                            "o", Object.class, o})
            );
        if (advice.before()) {
            return advice.after(__wrapped__doBar(
                advice.getParameterValue(0),
                advice.getParameterValue(1),
                advice.getParameterValue(2)
            );
        }
        return advice.onSkip();
    }

    int __wrapped__doBar(String bar, int i, float f, Object o) {
        System.out.println("bar is: " + bar);
        System.out.println("i is: " + i);
        System.out.println("f is: " + f);
        System.out.println("o is: " + o);
    }
}
```
Now that the hand-waviness has been taken care of, this is now a good detailed example of how Cutter works. One abnormal aspect of the above code is the two method calls to `Cut.Utils`. `Cut.Utils` is a static class that is defined inside the `Cut` annotation. All the methods do is return a `JoinPoint` object and an array of `Parameter` objects. These objects aren't created directly for two reasons:

1. To solve potential import problems: There is no guarantee that the user has imported `JoinPoint` `Parameter`. Since the targeted method is annotated with `Cut`, there is a guarantee that `Cut` is imported.
2. From a design philosophy standpoint, as much logic as possible should be plain Java. This increases readability and testability.

## Design Issues
Now that we have a detailed example, I'll go through and address some missing and/or broken aspects of the design
### Handling Void Methods
I'll take the easiest one first. The given example has `return` statements for `Advice.after()` and `Advice.onSkip`. Clearly, this pattern can't simply be used for `void` methods. The simplest solution to that is to use two method patterns, one for `void` methods and one for methods which return a value. The user can then implement Advice with a generic type of `java.lang.Void`.
### Imports
I've already talked about this a bit, and in general what I said holds true. By nature of including a Cut, Cutter can infer what classes have been imported and work around that.
### Abstract methods and interface methods
Placing a cut on an abstract method or an interface method would lead to illegal code being generated. This isn't really a problem, however. Behavior from placing a Cut on an abstract method would already be confusing, and Java's annotation inheritance rules are sketchy at best. Thus, Cutter doesn't allow Cuts to be placed on abstract methods or interfaces.
### Inheritance
While it might seem like inheritance would lead to unusual behavior, it actually doesn't. This is because calls to super methods in java require an explicit method name. At the byte-code level, `super.x()` is very similar to `instance.x()`. Thus, Cuts are triggered in the usual way when super-methods are called. 

Another possible point of confusion is the handling of Cuts placed on overriden methods. Because of the way that java linkage works, **the Cut placed on the actually executed method will be triggered.**
For example:
```java
class Parent {
    @Cut(Bar.class)
    public void foo() {}
}
class OverrideChild() extends Parent {
    @Cut(OtherBar.class) {}
    @Override
    public void foo() {}
}
class EmptyChild() extends Parent{
}
```
If the actual type of the object where `foo` is called is `Parent` the advice `Bar` will be called. If the actual object type is `OverrideChild` then `OtherBar` will be called. If the object is `EmptyChild` then `Bar` will be called.

To be abundantly clear, the below code will cause `OtherBar` to be called.
```java
void test {
	Parent p = new OverrideChild();
	p.foo();
}
```
### Advice Parameters
If the Advice is triggered at a Cut is defined with a `.class` argument, how can you pass parameters to the Advice?

The solution is actually rather simple: Use a separate annotation. This makes a lot of sense of a couple reasons:

 1. Simplifies core Cutter functionality
 2. [Separates the "advice" domain and the "meta-data" domain.](#Further-Details)
 3. Allows for complex parameter requirements to be fulfilled - since parameter annotation are just plain Java annotations, they can have their own annotation processors if needed

Some disadvantages:

 1. [Reduces API simplicity - the user has to define their own annotations to pass parameters to Advice](#Further-Details)
 2. [Reduces API readability - Needing multiple annotations over a method can make the developer's intent more difficult to understand](#Further-Details)
 3. Reduces performance - Reflection is necessary to access Advice parameters

The main benefit is the increased simplicity of cutter-lib. The main disadvantages are the API readability and simplicity concerns. Passing any parameters to an Advice in Cutter requires an annotation to be defined which can carry those parameters. This has the potential to lead to cluttered method declarations. I decided that this cluttering was minimal enough that the increased implementation simplicity was more important. 

There may also be a lot of benefit in allowing parameters to be passed through standard Java mechanism. It gives a modular way to expand Cutter by using standard Java APIs. Consider the following example: I want to add an Advice that caches the result of a method. In order to this, I define a `Cacheable` Advice. Next, I define a `CacheParams` annotation which handles Joinpoint specific information. In this case, `CacheParams` has a `key` field which should contain the String name of the method argument to be used for cache key generation. Because `CacheParams` is a standard Java annotation, I can write an annotation processor that ensures that the value of `key` is valid at compile-time. 

This pattern allows Cutter to leverage powerful Java APIs like the annotation processor API without adding any complexity to Cutter itself. I was able to implement custom behavior that is enforced at compile-time using only standard Java code.

Another potentially overlooked advantage to this scheme is consistency: A Pointcut is always defined by an `@Cut`. Other schemes would have custom annotations for different Pointcuts or different Advice, which has the potential to become confusing.
# Future Work

 1. Compile-time "require annotations" - Soon there will be a `@RequiredAnnotations` annotation for Advice objects to allow compile-time checks that ensure required parameters exist
 2. Improved compile-time performance - Currently `cutter-compile` walks the entire AST to finds Cuts. This can be replaced with standard annotation processing API calls to find annotated methods.
 3. Better Anonymous Inner Class handling - Currently limitations in the Pluggable Annotation Processing API make it nearly impossible to detect Anonymous Inner Classes. This means that Cuts placed on Anonymous Inner Class methods don't get detected. They should at least cause a compiler error.

# Further Details

## Advice vs Parameter Domains
Early I claimed that there are separate "advice" and "parameter" domains for advices and advice parameters. This claims is based on the idea that advice parameters are really just meta-data about a particular Joinpoint and not really part of the advice. In theory, this generally holds true. Going back to the caching example, a unique-key argument for a cache is really a property of the Joinpoint. The fact that a method argument is unique for unique outputs has nothing to do with caching. However, in practice, the lines are somewhat blurry. For example, when using a unique-key for caching, it would be most clear to define a `CacheParams` annotation rather then several annotations that describe the Joinpoint in a more abstract way. Furthermore, the idea of a cache key is tightly coupled with the idea of a cache. So in practice a `unique-key` is tied to the domain of the actual cache (which would likely be implemented as an `Advice`.

While I do believe there are two domains for advice and advice parameters, the actual usefulness of this should be taken with a grain of salt, I guess.
## API Readability
There are some best practices which can significantly reduce the extra clutter from needing a second "parameters annotation." 

By convention, parameters annotations should be named following the pattern `xxxParams`. This makes it immediately clear why the annotation is present. (Example `CacheParams`).

Furthermore, the `Cut` annotation should be placed on the same line as a `Params` annotation. 

Example of incorrect usage:
```java
@Cut(Cache.class) //Parameters annotation should be on this line
@CacheInfo(key = "uniqueName")  //Parameters annotation should be named "CacheParams"
public void Foo(String uniqueName) {}
```
Example of correct usage
```java
@Cut(Cahce.class) @CacheParams(key = "uniqueName")
public void Foo(String uniqueName) {}
```