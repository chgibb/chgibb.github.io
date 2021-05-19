
[Hydro-SDK](https://github.com/hydro-sdk/hydro-sdk) is a project with one large, ambitious goal. "Become React Native for Flutter".  
It aims to do that by:
1. Decoupling the API surface of Flutter from  the Dart programming language.
2. Decoupling the development time experience of Flutter from the Dart programming language.
3. Providing first-class support for over-the-air distribution of code.
4. Providing an ecosystem of packages from `pub.dev`, automatically projected to supported languages and published to other package systems.

I wrote previously about the past and future of [Hydro-SDK](https://github.com/hydro-sdk/hydro-sdk) [here](https://chgibb.github.io/one-year-of-hydro-sdk/). 

In this article I want to delve into some of the gritty details of how Hydro-SDK compiles, manages and hot-reloads Typescript code.

`ts2hc` is a command line program distributed as part of each Hydro-SDK release. `ts2hc`'s job is to turn Typescript code into bytecode and debug symbols. Users generally don't interact with it directly but rather indirectly through commands like `hydroc build` and `hydroc run`.

## Compile Time
On its way to bytecode, Typescript is first lowered into Lua by leveraging the excellent [Typescript to Lua (TSTL) library](https://github.com/TypeScriptToLua/TypeScriptToLua). Consider the following excerpt from the [Counter-App showcase](https://github.com/hydro-sdk/counter-app):
```typescript
//counter-app/ota/lib/counterApp.ts
import {
    //...
    StatelessWidget,
} from "@hydro-sdk/hydro-sdk/runtime/flutter/widgets/index";
import { 
    //...
    MaterialApp, 
} from "@hydro-sdk/hydro-sdk/runtime/flutter/material/index";
import { Widget } from "@hydro-sdk/hydro-sdk/runtime/flutter/widget";

export class CounterApp extends StatelessWidget {
    public constructor() {
        super();
    }

    public build(): Widget {
        return new MaterialApp({
            title: "Counter App",
            initialRoute: "/",
            home: new MyHomePage("Counter App Home Page"),
        });
    }
}
```

`ts2hc` will lower `counter-app/ota/lib/counterApp.ts`, all of its dependencies, and then bundle the result into something like the following:

```lua
local package = {preload={}, loaded={}}
local function require(file)
    local loadedModule = package.loaded[file]
    if loadedModule == nil
    then
        loadedModule = package.preload[file]()
        package.loaded[file] = loadedModule
        return loadedModule
    end
    return loadedModule
end
-- ...
package.preload["ota.lib.counterApp"] = (function (...)
    require("lualib_bundle");
    local ____exports = {}
    local ____index = require("ota.@hydro-sdk.hydro-sdk.runtime.flutter.widgets.index")
    -- ...
    local StatelessWidget = ____index.StatelessWidget

    -- ...
    local ____index = require("ota.@hydro-sdk.hydro-sdk.runtime.flutter.material.index")
    local MaterialApp = ____index.MaterialApp

    ____exports.CounterApp = __TS__Class()

    local CounterApp = ____exports.CounterApp

    CounterApp.name = "CounterApp"

    __TS__ClassExtends(CounterApp, StatelessWidget)

    function CounterApp.prototype.____constructor(self)
        StatelessWidget.prototype.____constructor(self)
    end

    function CounterApp.prototype.build(self)
        return __TS__New(
            MaterialApp,
            {
                title = "Counter App",
                initialRoute = "/",
                home = __TS__New(MyHomePage, "Counter App Home Page")
            }
        )
    end

    return ____exports

end)
```

The lowered Lua still somewhat resembles the input Typescript. Typescript ES6 modules are wrapped into Lua immediately invoked function expressions (IIFE), assigned string keys in the `package.preload` map and their `exports` made available by `require`ing them. This pattern should be familiar to anyone who's hacked on Javascript bundlers/module resolvers like Browserify or Rollup.

Lua lacks builtin object-oriented programming (OOP) facilities (wether prototypal or otherwise). Typescript language features which don't quite map one-to-one with Lua are shimmed using `__TS_*` functions made available through the `lualib_bundle` module (which `ts2hc` injects during bundling). Above, the `CounterApp` class is lowered into a series of calls to `__TS__Class` and `__TS__ClassExtends`, followed by placing its declared methods on its `prototype`.

### Mangling
In addition to lowering, `ts2hc` also undertakes an analysis pass of each output Lua module to discover its functions.

From the above example, `ts2hc` will record the following functions:
```
CounterApp.prototype.____constructor
CounterApp.prototype.build
```
corresponding to the constructor and `build` method declarations of the original `CounterApp` class. These names are obviously captured from the original declaration names. For cases like our example (where clear function names where given), this works well enough.

`ts2hc` cannot rely on the programmer giving it clear and comprehensible function names however. `ts2hc` takes the original symbol names from the output Lua module and performs [name mangling](https://en.wikipedia.org/wiki/Name_mangling#Name_mangling_in_C.2B.2B). The intent of name mangling is to uniquely identify a given function, no matter where or how it was declared. `ts2hc`'s name mangling approach is heavily inspired by the IA-64 Itanium C++ ABI as well as Rust's name mangling approach. Whereas those two approaches rely heavily on type-information to produce their mangled names, Lua is a dynamic and un-typed language and offers no such convenience. `ts2hc` first considers the names and declaration order of function arguments resulting in the following:
```
CounterApp.prototype.____constructor::self
CounterApp.prototype.build::self
```
That is not quite enough information to uniquely identify a function however. `ts2hc` further considers the hash of the Typescript filename, and a disambiguation index suffix to resolve mangled name conflicts by declaration order resulting in the following: 
```
_Lae3eafcf842016833530caebe7755167b0866b5ac96416b45848c6fc6d65c58f::CounterApp.prototype.____constructor::self::0
_Lae3eafcf842016833530caebe7755167b0866b5ac96416b45848c6fc6d65c58f::CounterApp.prototype.build::self::0    
```   
This form works perfectly for functions that are named by the programmer like class methods or free functions. Consider however if the `build` method on the `CounterApp` class were written to have an anonymous closure:
```typescript
public build(): Widget {
        return new MaterialApp({
            title: "Counter App",
            initialRoute: "/",
            home: (() => new MyHomePage("Counter App Home Page"))(),
        });
    }
```

For anonymous closures, `ts2hc` simply names them "anonymous_closure". In order to uniquely identify anonymous closures (or any nested function declarations), a [dominator analysis](https://en.wikipedia.org/wiki/Dominator_(graph_theory)) with respect to the declaration of every function is performed. The mangled name of the immediate dominator as well as the mangled names of every member of the dominance frontier of each function are prefixed to its mangled name. For the anonymous closure in the example above, it results in the following mangled name:
```
_Lae3eafcf842016833530caebe7755167b0866b5ac96416b45848c6fc6d65c58f::CounterApp.prototype.build::self::0::anonymous_closure::0
```
This form encodes enough information to uniquely identify a function. `ts2hc` will join each functions mangled name with information like the line/column numbers in the original Typescript file, the line/column numbers in the Lua module the original Typescript file was lowered into, the line/column numbers the function ended up in in the final Lua bundle and other information into a single debug symbol. These debug symbols are what power function maps, provide readable stack traces as well as hot-reload. 

## Runtime
Common Flutter Runtime (CFR) is a blanket term given to the Lua 5.2 virtual machine, binding system and other libraries at the core of Hydro-SDK's runtime environment. Users generally don't interact with the CFR directly, but rather through widgets like `RunComponent` and `RunComponentFromFile`.

`ts2hc` and CFR are at the very core of the developer experience and runtime system of Hydro-SDK. They work together to support goal 2 above by providing an analog to Flutter's killer development-time feature; hot-reload.

In Flutter, hot-reload is provided by Dart VM. Dart VM's hot-reload is based on a few [pillars](https://github.com/dart-lang/sdk/wiki/Hot-reload#immutable-methods):

Pervasive Late-Binding
- The program behaves as if method lookup happens at every call

Immutable Methods
- The "atoms" of reload are methods. Methods are never mutated. Changes to a method declaration create a new method, mutating the class or library's method dictionary. The old method may still exist if it has been captured by a closure or stack frame.
- Closures capture their function when they are created. A closure always has the same function before and after a change, and all invocations of a given closure run the same function.

State is Retained
- Hot reload does not reset fields, neither the fields of instances nor those of classes or libraries

CFR's hot-reload is inspired by (and largely abides by) the same pillars.