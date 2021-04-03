---
layout: post
title: Expressing Inexpressible Constant Values
---

Adventures in automatically binding Dart to Typescript.

[Hydro-SDK](https://github.com/hydro-sdk/hydro-sdk) is a project with one large, ambitious goal. "Become React Native for Flutter".  
It aims to do that by:
1. Decoupling the API surface of Flutter from  the Dart programming language.
2. Decoupling the development time experience of Flutter from the Dart programming language.
3. Providing first-class support for over-the-air distribution of code.
4. Providing an ecosystem of packages from `pub.dev`, automatically projected to supported languages and published to other package systems.

I wrote previously about the past and future of [Hydro-SDK](https://github.com/hydro-sdk/hydro-sdk) [here](https://chgibb.github.io/one-year-of-hydro-sdk/). 

Structured Wrapper and Interface generator for Dart (SWID) is a component of Hydro-SDK supporting goals 1 and 4. SWID takes as input a Dart package (like `package:flutter`) and produces as output Typescript files representing the input Dart package's public API and Dart files to allow for gluing Typescript code written against that API together into a host app that can be run on Hydro-SDK's Common Flutter Runtime (CFR). This process is referred to as "language projection". 

Representing Dart constructs in Typescript has come with many challenges. One recently encountered is how to express constant values which are composed of private symbols. 

Every Flutter programmer should be familiar with the concept of `const` in Dart. It’s used perhaps most prolifically in Flutter’s Material and Cupertino design icons APIs. For instance, the `Icons` class in Flutter’s `material` library contains thousands of static fields that appear like the following which represents the "directions boat" material design icon: 
```dart
//package:flutter/lib/src/material/icons.dart
class Icons {
    Icons._();
    //abridged
    static const IconData directions_boat = IconData(0xe6b9, fontFamily: 'MaterialIcons'); 
}
``` 

Describing how Flutter `IconData` instances, font ligatures and font loading work is out of scope of this article.

When performing language projection of `package:flutter`, SWID will emit the above declaration as the following:

```typescript
//runtime/flutter/material/icons.ts
import { IconData } from "./../widgets/iconData";
export class Icons {
    //abridged
    public static directions_boat = new IconData(0xe6b9, {
        fontFamily: "MaterialIcons",
    });
}
```

SWID will perform a simple syntax transform for the `Icons` class and `directions_boat` field while making sure the translation unit (`flutter/material/icons.ts`) imports required symbols (in this case, simply `IconData`). 

The only area of language projection that SWID aims to describe one-to-one between Dart and Typescript is `const` values. This is done to free host applications from having to compile every possible constant value that guest code might want to access. This approach works great for constant values that consist simply of public symbols (like `IconData` above), and primitive values.

This becomes trickier however when fields consist of private symbols. For instance, the `Rect` class from `dart:ui` has some static constant fields which are declared as the following:
```dart
//package:sky_engine/ui/geometry.dart
class Rect {
    static const Rect zero = Rect.fromLTRB(0.0, 0.0, 0.0, 0.0);
    static const double _giantScalar = 1.0E+9; 
    static const Rect largest = Rect.fromLTRB(-_giantScalar, -_giantScalar, _giantScalar, _giantScalar);
}
```

The `zero` field references a public static method, so it is fine to perform a simple syntax transform. At first glance, the `largest` field appears impossible to express in Typescript. There is no way to expose the `_giantScalar` symbol in a way that it can be accessed from Typescript. 

SWID has enough semantic understanding to understand that the reference to `_giantScalar` from `largest` is not only a field on `Rect` but that it's also just a primitive. Therefore, both are safe to emit.

```typescript
//runtime/dart/ui/rect.ts
export class Rect {
    public static zero = Rect.fromLTRB(0.0, 0.0, 0.0, 0.0);
    private static _giantScalar = 1.0e9;
    public static largest = Rect.fromLTRB(
        -Rect._giantScalar,
        -Rect._giantScalar,
        Rect._giantScalar,
        Rect._giantScalar
    );
}
```

SWID's semantic understanding of the Dart code it is projecting allow it to categorize constant values into two categories: "expressible" and "inexpressible". Constant values which can be decomposed into references to primitives or public symbols are considered to be "expressible". In the `Rect` example above, `_giantScalar`, despite being private, decomposes into a reference to a primitive which allows `largest` to decompose into primitive references. `largest` referencing `_giantScalar` causes `_giantScalar` to be emitted in the final translation unit.

Consider the `Endian` class from `dart:typed_data`:
```dart
//package:sky_engine/typed_data/typed_data.dart
class Endian {
  final bool _littleEndian;
  const Endian._(this._littleEndian);

  static const Endian big = const Endian._(false);
  static const Endian little = const Endian._(true);
  //abridged
}
```

Here, `big` and `little` both include references to a private constructor. This can't be further decomposed into an expressible form and so both fields are considered to be "inexpressible". SWID expresses "inexpressible" constant values by emitting code like the following:

```typescript
//runtime/dart/typed_data/endian.ts
declare const dart: {
    typed_data: {
        endianBig: (this: void) => Endian;
        endianLittle: (this: void) => Endian;
    };
};
//abridged
export class Endian {
    public static big = dart.typed_data.endianBig();
    public static little = dart.typed_data.endianLittle();
}
```

The `declare` block is what SWID calls a "virtual machine declaration". This is a typed description of the environment that the code in the given translation unit expects. "virtual machine declarations" in SWID's Typescript backend and their associated "namespace symbol declarations" that fulfill their expectations in SWID's Dart backend are how SWID expresses API bindings not just for "inexpressible" constant values but for all methods and fields. This binding system will be the subject of a future blog post.

As seen above, `big` and `little` are expressed as calls to their associated declarations. Unfortunately, this scheme prevents the host application from ever being able to tree-shake away the definitions of `big` and `little` but still allows guest code access to these fields.

Hopefully this problem and its solution was as fun to read about as it was to discover and solve. Hydro-SDK is an endless fractal of problems like this. Hopefully I’ll be able to make “Adventures in automatically binding Dart to Typescript” a continuing series of posts.

