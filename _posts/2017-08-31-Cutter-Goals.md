---
layout: single
title: Cutter, A Minimal Java AOP Library (Part 1 -  Motivation and Goals)
---
[Cutter](https://github.com/Spaceman1701/Cutter) is intended to be a fast and
lightweight [aspect-oriented
programming](https://en.wikipedia.org/wiki/Aspect-oriented_programming) library
(I’m not going to bother explaining aspect-oriented programming here, since
Wikipedia probably will do a better job). While there are already several java
frameworks that allow for aspect oriented programming in one way or another
([AspectJ](https://eclipse.org/aspectj/index.php),
[JBoss](http://jbossaop.jboss.org/),
[Spring](https://docs.spring.io/spring/docs/current/spring-framework-reference/html/aop.html))
they all are giant monolithic frameworks (of doom) that essentially dictate the
entire design of your application. While this is okay in some cases, it’s
frustrating to want a single feature of a library only to find that you need a
special compiler plugin, a large (slow) runtime, and/or tenuous
interoperability.

This is where Cutter comes in. The driving principle behind Cutter’s design is
that 99% of the time 1% of the functionality in a large framework is being used.
Cutter is designed to have no bloat whatsoever. Sometimes this can lead to more
awkward API syntax, but it has the advantage of allowing applications to
configure Cutter however they like by writing plain old Java.

Cutter allows for the definition of
[Pointcuts](https://en.wikipedia.org/wiki/Pointcut) in Java. In Cutter,
Pointcuts are defined by a special “Cut” annotation and a plain abstract Java
class for defining [Advice](https://en.wikipedia.org/wiki/Advice_(programming)).
This ensures that Cutter is *completely modular* and *completely pluggable*.

The guiding principles behind Cutter are:

-   Pure Java

    -   Cutter should 100% pure Java. Using Cutter should be 100% pure Java.

-   Minimal

    -   Anything that can be added by standard in Java by the user shouldn’t be
        in cutter-lib.

-   No Configuration

    -   Cutter shouldn’t require any configuration files to use. Using Cutter
        just requires that it is in the classpath.

-   Interoperable

    -   Cutter should be seamlessly interoperable with any other Java code.

-   Secure

    -   Cutter should be safe to use for security features and should not
        require special permissions from the security manager at runtime.

-   Fast

    -   Cutter shouldn’t incur any significant performance penalty

In order to meet this requirements Cutter is implemented using the Pluggable
Annotation API and does not require Java agent or special initialization. All of
the work is done statically at compile time. In order to ensure performance is
sufficient and to avoid insure reflection calls to change method access, Cutter
does not depend on reflection for basic Pointcuts.

The core Cutter library, cutter-lib, has exactly one feature: Pointcuts on
methods. These are the most common types of Pointcuts and are the most useful.
While other libraries allow Pointcuts on fields, these can be implemented in
Java by the user (through getter methods), so they were not included in the
library.

What Cutter is not:

-   A feature-for-feature competitor with AspectJ (or Spring)

    -   Cutter is intended to be small and lightweight. AspectJ and Spring will
        always do more than Cutter does, and that’s okay. Some of the major features Cutter will probably never have:

        -   Almost everything in Spring AOP

            -   Philosophically cutter-lib will never have the same complex
                features as Spring (though many of them will be possible to
                implement in Cutter-based libraries)

        -   Pointcuts defined away from the declaration-site

            -   While this may change in the future, right now Cutter is
                focused on Pointcuts defined with “Cut” annotations directly
                at the method declaration they target. This ensures that
                Pointcuts are always defined explicitly

            -   It is possible, however, to write a Cutter-based library
                that adds this feature

        -   Pointcuts targeting fields

            -   This probably indicates bad design anyway (And violates
                basic Java design principles).

-   A “true” aspect-oriented programming framework

    -   Cutter supports a subset of features that are *similar* to
        aspect-oriented programming. APIs are named to resemble these features
        to improve understandability.

        -   A Cut is not a true Pointcut because it targets exactly one
            Joinpoint

        -   There is no concept of an “Aspect” in Cutter

    -   What Cutter adds is probably most similar to the way [Python
        decorators](https://wiki.python.org/moin/PythonDecorators) work

[You can download an early version of Cutter right now from my Github](https://github.com/Spaceman1701/Cutter). There’s no binary distribution
yet, but building Cutter is as simple as ```mvn clean install```
