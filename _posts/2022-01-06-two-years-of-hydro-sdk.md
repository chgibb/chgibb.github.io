---
layout: post
title: Two Years of Hydro-SDK
---

370 releases in 365 days of 2021. Looking ahead to 2022.

[Flutter](https://flutter.dev/) is Google's UI toolkit for building beautiful, natively compiled applications for mobile, web, desktop and embedded devices from a single codebase.

[Hydro-SDK](https://github.com/hydro-sdk/hydro-sdk) is a software development kit (SDK) enabling Flutter developers to write portions of their app (or their entire app) using TypeScript. It is a project with one large, ambitious goal. "Become React Native for Flutter".  
It aims to do that by:
1. Decoupling the API surface of Flutter from  the Dart programming language.
2. Decoupling the development time experience of Flutter from the Dart programming language.
3. Providing first-class support for over-the-air distribution of code.
4. Providing an ecosystem of packages from `pub.dev`, automatically projected to supported languages and published to other package systems.

In 2021, [the Hydro-SDK Github organization](https://github.com/hydro-sdk) saw 11,710 commits, 1,616 pull requests merged and 136 issues closed. The [main Hydro-SDK repository](https://github.com/hydro-sdk/hydro-sdk) saw 2,342 commits, 382 pull requests merged, 136 issues closed, and 370 releases. While Hydro-SDK is still very much pre-beta software, releases are automated and frequent using Hydro-SDK's own [waveform-bot](https://github.com/waveform-bot). This has led to publishing 5 more releases than there are days in 2021.

The other 9,368 commits and 1,234 pull requests were made against Hydro-SDK's showcase and test repositories. In addition to the [extensive CI](https://github.com/hydro-sdk/hydro-sdk/tree/e03ea38704abdd13dce932fa10101ed0659a956b/.github/workflows) on the [main Hydro-SDK repository](https://github.com/hydro-sdk/hydro-sdk), these repositories act not only as starting points for users of Hydro-SDK but also as high-level integration tests. On every Hydro-SDK release, [waveform-bot](https://github.com/waveform-bot) opens pull requests updating the Hydro-SDK version used by these repositories to the latest (and resolves merge conflicts with itself).

The focus in 2021 was on proving the potential of the [Structured Wrapper and Interface Generator for Dart (SWID) project](https://hydro-sdk.io/docs/design-documents/swid) and building out a [codepush MVP](http://localhost:3000/blog/fluttering-over-the-air).

SWID started 2021 at ~26,509 LOC (lines of code) with ~3,692 LOC of tests and ended the year at ~98,291 LOC with ~34,399 LOC of tests. While LOC is not a measure of value, it is a reasonable proxy for complexity. Two sub-projects spun out of SWID ([Swidi](https://hydro-sdk.io/docs/design-documents/swidi), [SWARS](https://hydro-sdk.io/docs/design-documents/swars)) as a result of the growing complexity and needs of SWID. SWID even managed to trigger a bug in the Dart compiler ([dart-lang/sdk#46936](https://github.com/dart-lang/sdk/issues/46936)).

Getting to even the basic codepush MVP that is now available required completely rethinking both the development time and deployment experience of code built and ran using Hydro-SDK. Previously, all components of Hydro-SDK operated directly on bytecode chunks. A new [packaging format (`.ota` packages)](https://hydro-sdk.io/docs/design-documents/ota) was created along with new SDK-tools and supporting libraries to [build](https://github.com/hydro-sdk/hydro-sdk/tree/master/lib/build-project), [serve](https://github.com/hydro-sdk/hydro-sdk/blob/master/bin/run-project.dart), [run](https://github.com/hydro-sdk/hydro-sdk/tree/master/lib/runComponent), and [deploy](https://github.com/hydro-sdk/hydro-sdk/blob/master/bin/codepush.dart) `.ota` packages. The (closed-source) service powering the [Hydro-SDK Registry](https://hydro-sdk.io/docs/design-documents/registry) at https://registry.hydro-sdk.io was created along with [(open-source) support to interact with it](https://github.com/hydro-sdk/hydro-sdk/tree/master/lib/registry/dto).

Work done in 2021 proved the value of the approach taken by SWID to the larger problem space of automatic language projection and ecosystem bridging faced by Hydro-SDK (goals 1 and 4 above). The codepush MVP provides a complete steel thread from developing, to deploying and operating experiences built using Hydro-SDK.

The focus over the course of 2022 will be on the development experience for users of Hydro-SDK. It is hoped that SWID will be able to provide in-tree support for the whole of Flutter. The barrier to entry for web developers looking to Hydro-SDK as a way to bridge their skills into using Flutter (and mobile development in general) is high due to the inherent complexities of mobile development. Hopefully, exploring ways to lower these barriers for web developers can begin in 2022 [hydro-sdk/hydro-sdk#763](https://github.com/hydro-sdk/hydro-sdk/issues/763).