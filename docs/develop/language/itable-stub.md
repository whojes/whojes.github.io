---
layout: default
title: itable, vtable stub
parent: java/kotlin
grand_parent: 개발
created_at: 2023.04.26
print_title: true
share_enable: true
permalink: develop/java/itable-stub
---

## *WIP*

itable/vtable stub 이 뭘까? 

[읽어봄직1](http://insightfullogic.com/2014/May/12/fast-and-megamorphic-what-influences-method-invoca/)  
[읽어봄직2](https://mrale.ph/blog/2015/01/11/whats-up-with-monomorphism.html)  
[*너무길어서 못읽겠는 흑마술*](https://shipilev.net/blog/2015/black-magic-method-dispatch/)  

> An "itable stub" is a term from the Java Virtual Machine (JVM) world, and it refers to the method stubs that are generated as part of the Interface Method Resolution process. An itable (short for Interface Method Table) is a data structure that enables efficient dispatching of interface method calls on the JVM.
> 
> When a Java class implements an interface, it needs to provide an implementation for each method declared in the interface. At runtime, when an object of the class is used to call an interface method, the JVM needs to find the correct method implementation to invoke. This is where the itable comes into play.
> 
> The itable is an array-like data structure, where each entry corresponds to a method in the interface. The itable stubs are small pieces of generated code that perform a lookup in the itable and dispatch the method call to the appropriate implementation. When a class implements an interface, the JVM generates an itable stub for each method declared in the interface and associates it with the class.
> 
> During runtime, when a method call on an interface is encountered, the JVM:
> 
> 1. Determines the actual class of the object.
> 2. Finds the itable associated with the class and the interface.
> 3. Uses the itable stub to look up the correct method implementation in the itable.
> 4. Invokes the method implementation.  
>
> The itable and itable stubs enable fast and efficient dispatching of interface method calls in the JVM, even in the presence of complex class hierarchies and multiple inheritance of interfaces.

> A "vtable stub" refers to a method stub generated as part of the virtual method table (vtable) resolution process in object-oriented programming languages that run on a virtual machine, such as Java or C++. A vtable (short for virtual method table) is a data structure that enables efficient dispatching of virtual method calls at runtime.
> 
> When a class has one or more virtual methods (or overrides a method from a base class), a vtable is created for the class. The vtable is an array-like data structure, where each entry corresponds to a virtual method in the class. The vtable stubs are small pieces of generated code that perform a lookup in the vtable and dispatch the method call to the appropriate implementation. When a class has a virtual method, the runtime environment generates a vtable stub for each method declared in the class hierarchy and associates it with the class.
> 
> During runtime, when a virtual method call is encountered, the runtime environment:
> 
> 1. Determines the actual class of the object (in the case of inheritance).
> 2. Finds the vtable associated with the class.
> 3. Uses the vtable stub to look up the correct method implementation in the vtable.
> 4. Invokes the method implementation.
> 
> The vtable and vtable stubs enable fast and efficient dispatching of virtual method calls at runtime, supporting polymorphism and allowing objects of different classes to be treated as objects of a common superclass or interface. This mechanism is a key feature of object-oriented programming languages like C++, Java, and C#.