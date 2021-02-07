---
layout: post
title: Closing Over Type Formals, Satisfying Type Parameters
---

Adventures in automatically binding Dart to Typescript.

[Hydro-SDK](https://github.com/hydro-sdk/hydro-sdk) is a project with one large, ambitious goal. "Become React Native for Flutter".  
It aims to do that by:
1. Decoupling the API surface of Flutter from  the Dart programming language.
2. Decoupling the development time experience of Flutter from the Dart programming language.
3. Providing first-class support for over-the-air distribution of code.
4. Providing an ecosystem of packages from `pub.dev`, automatically projected to supported languages and published to other package systems.

I wrote previously about the past and future of [Hydro-SDK](https://github.com/hydro-sdk/hydro-sdk) [here](https://chgibb.github.io/one-year-of-hydro-sdk/). 

Structured Wrapper and Interface generator for Dart (SWID) is a component of Hydro-SDK supporting goals 1 and 4. SWID takes as input a Dart package (like `package:flutter`) and produces as output Typescript files representing the input Dart package's public API and Dart files to allow for glueing Typescript code written against that API together into a host app that can be run on Hydro-SDK's Common Flutter Runtime (CFR). This process is referred to as "language projection". 

Representing Dart constructs in Typescript has come with many challenges. The most interesting so far has been Dart's [`factory` constructors](https://dart.dev/guides/language/language-tour#factory-constructors). Consider the `Iterable` class from `dart:core`:

```dart
//abridged for readability
class Iterable<E> {
    Iterable<E> Iterable();
    Iterable<T> map<T>(T f(E e))
    factory Iterable<E> Iterable<E>.empty()
}
```

The `empty` factory allows consumers to write code like the following:
```dart
Iterable<MyType> myEmptyIterable = Iterable<MyType>.empty();
```

Factory constructors are nothing new. While Typescript doesn't have direct support at the language level as Dart does, the same syntax for consumers can be had by the following class:
```typescript
class Iterable<E> {
    public constructor();
    public map<T>(f: (e: E) => T): Iterable<T>;
    public static empty(): Iterable<E>;
}
```
 Which would enable consumers to call the `empty` method from Typescript as the following:

 ```typescript
 let myEmptyIterable: Iterable<MyType> = Iterable.empty();
 ```

 It would, except the definition of `Iterable.empty` in our Typescript `Iterable` class above is actually invalid. The Typescript compiler tells us that `Static members cannot reference class type parameters.(2302)`. There is a fairly lengthy discussion on why the TS2302 error exists and why this is the case over on the Typescript repo [here](https://github.com/microsoft/TypeScript/issues/32211) which is out of scope for this post. The jist of the problem is that our `empty` method is trying to reference the type parameter `E` which is defined on the class. 

SWID is structured as a frontend which takes in Dart source code, producing an intermediate representation (IR). This IR is then passed onto the Typescript backend to produce Typescript source code. SWID IR closely mirrors an abstract syntax tree (AST) of the input Dart code. The `empty` `factory` constructor in the original `Iterable` class defines no type parameters of its own and makes references to `E` defined on the class and therefore so does the IR and the output Typescript. From a purely code generation perspective, our Typescript `Iterable` class is perfect. From a perspective of semantics preservation from our input language (and simply a correctness perspective), this is obviously far from perfect.

In SWID IR, the declaration of a generic type is said to be a type formal while uses of that generic are said to be type parameters. In SWID parlance, the `Iterable` class declares a type formal `E` while `empty` has a type parameter `E`.

In order to stop the Typescript backend from producing broken code in the face of patterns like `Iterable`, a type propagation pass was introduced prior to sending IR off to be turned into Typescript. The type propagation pass is responsible for rewriting IR in order to satisfy type parameters that are unsatisfied. 

In SWID IR, a type formal closure defines all of the type formals that are in scope for a particular IR node. A type parameter is said to be unsatisfied if there is no type formal in it's type formal closure that matches it. The extent to which a particular IR node closes over the type formals of its parent nodes depends on the position of the IR node.

The `map` method in the `Iterable` class for example has type parameters `E`, and `T`. The type formal closure of `map` consists of `T` because `T` is declared on `map` itself as well as `E` because `map` is an instance method of `Iterable`.

The `empty` method in the `Iterable` class has a type parameter `E` but its type formal closure contains nothing. `empty` is a static method and therefore does not close over the type formals of it's parent `class`.

The type propagation pass will discover the unsatisfied type parameter `E` in the `empty` method, realise that it could be satisfied if the type formal `E` declared on its parent `class` was also made a type formal of `empty` itself and perform the rewriting. This gives us the following `Iterable` class:

```typescript
class Iterable<E> {
    public constructor();
    public map<T>(f: (e: E) => T): Iterable<T>;
    public static empty<E>(): Iterable<E>;
}
```

With consumers able to use it like the following:
 ```typescript
 let myEmptyIterable: Iterable<MyType> = Iterable.empty<MyType>();
 ```

 The end result for consumers is having to shift where they ordinarily write their generics when calling `factory` constructors.

 The type propogation pass itself can be found [here](https://github.com/hydro-sdk/hydro-sdk/blob/0.0.1-nightly.99/lib/swid/ir/frontend/dart/util/propagateUnsatisfiedTypeParameters.dart). Tests for satisfying type parameters and propagating formals can be found [here](https://github.com/hydro-sdk/hydro-sdk/blob/0.0.1-nightly.99/test/swid/typeFormals/unsatisfiedTypeFormals_test.dart).

 Hopefully this problem and its solution was as fun to read about as it was to discover and solve. Hydro-SDK is an endless fractal of problems like this. Hopefully I'll be able to make "Adventures in automatically binding Dart to Typescript" a series of posts.