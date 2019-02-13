---
layout: post
title: Dependency Options for iOS apps
date: 2016-08-22
tags: [iOS]
redirect_from:
  - /blog/2016/08/22/Dependency-Options-for-iOS.html.html
---

Many languages today either come with a full featured dependency manager, or have a canonical one that everyone uses and is supported universally. A dependency manager does the work of downloading and integrating any dependencies (3rd party libraries/frameworks) including dependencies of those dependencies. Unfortunately, Apple doesn't seem to accept the existence of 3rd party code[^1]. This has led to several options for incorporating external code into a project. I have personally used all of these options in shipping apps[^2].

All of these options apply to macOS, watchOS and tvOS, but the history of all these options were largely driven by iOS not supporting dynamic frameworks like macOS.

## No dependencies

This may sound scandalous to most developers that are familiar with languages like Ruby or Javascript, but it's quit possible to ship an iOS or mac app without any external dependencies. For starters, the systems themselves have a large number of frameworks already available to you, such as UIKit, Foundation, CoreData and may more. Even developers who do use dependencies on Apple platforms usually don't include that many.

There are *many* advantages to this approach. It makes for a much simpler setup process, with less configuration to maintain. When new betas of Xcode come out, you won't have to worry about compatibility or updating code you don't own.

Of course, the disadvantage is that you have to either live without certain features or write them yourself. If you only have 1 or 2 dependencies, I would highly encourage you to look at removing them entirely.

## Copy and paste

Who among us has truly written their app entirely themselves without copy and pasting at least a few lines from StackOverflow? Why stop there though, when you could copy entire folders of code into your project. In fact, this was the de facto standard for iOS apps before CocoaPods became popular. Open source libraries would literally have instructions in their Readmes on how to properly copy the code into an apps project. And while it's pretty rare these days to see instructions to do this on Github, you can still do it fairly safely.

This ends up being just as simple as not using dependencies at all. The external code ends up looking and acting like your own code. And you should probably think if it that way too. Anything you paste into a project you should consider your complete responsibility to maintain. Granted, that probably goes for any external code you import. After all, if your user's find a bug they aren't going to blame the random GitHub developer that published the library you're using.

The biggest downside to doing this is that it becomes difficult to keep the code up to date when new versions of the library are released. It's tempting to edit anything you copy into your project, which will get wiped out if you copy in a new version. I would recommend doing this approach sparingly, and only for small dependencies that you don't anticipate changing or improving.

## [Git Submodules](https://git-scm.com/book/en/v2/Git-Tools-Submodules)

In the bad old days, the really advanced developers would modify the copy and paste method to use Git submodules instead. You were still responsible for integrating the code into your project, but it made it much easier to update and even modify dependencies. Changes get tracked and can be merged with new versions. Git in general can be fairly sticky to navigate, and submodules only complicate that, but if you use a high quality GUI client like [Tower](https://www.git-tower.com), it should be fairly straightforward.

With both copy and paste and submodules, things get a lot trickier when your dependencies have dependencies; so called, *indirect dependencies*. You need to read through the documentation for any library you include and manually import it's dependencies as well. For this reason, iOS libraries developed a culture of not including indirect dependencies if at all possible. And even though CocoaPods handles this much better, it's still rare to see a large number of indirect dependencies in a project.

## Prebuilt Binaries

macOS has always supported dynamic frameworks, however iOS just got in on the fun with iOS 8. The popularity of iOS and the fact that it didn't support frameworks for so long led to a sharp decline in their popularity overall. But they have a lot to offer. Because they are prebuilt, theoretically you shouldn't need to worry about them compiling with new versions of Xcode[^3]. And, again theoretically, you should be able to just drop them into a project and have everything play nice 🌈.

Unfortunately that isn't really the case, especially on iOS. While Apple platforms have a rich support for "fat" binaries (single binaries that include multiple architectures such as 32 and 64bit), iOS has 2 very different types of architectures: Intel where the simulator runs, and ARM, where your apps run on device. You'll see many frameworks for iOS packaged with 4 architectures: Intel 32bit, Intel 64bit, ARM7 and ARM64. But here's the thing, dynamic frameworks (unlike static libraries) don't get their unused architectures stripped when they are archived. Meaning that if you include one of these frameworks in your app, you'll be shipping unused framework code to your users. And in fact, in Xcode 8 Apple explicitly forbids this. And it's not simple to conditionally use one or the other based on if you are building for the simulator.

It is possible however, to create a static framework. You need the framework wrapper if you want to import the library in Swift, but the actual binary can still be statically linked. This is an excellent solution when possible. Unused code should be stripped out automatically, making the resulting binary smaller, and you remove any [launch time overhead from runtime linking](https://developer.apple.com/videos/play/wwdc2016/406/). The downside of this though is that you can't include any resources (xib, storyboards, images etc.) and none of the code in the framework can be written in Swift.

## [Carthage](https://github.com/Carthage/Carthage)

Carthage tries to keep things simple. It combines the approach of submodules[^4] and frameworks and tries to let Xcode do most of the work. You create a file that lists what dependencies you want along with constraints on what version to use (i.e. greater than 2.3.0 but less than 3.0.0) and it selects the newest set of dependencies, along with indirect dependencies, that satisfy all of your version requirements. It then compiles all of those dependencies into dynamic frameworks and places them in a single folder. It is up to you from that point to import the compiled frameworks into Xcode. Framework authors can include a prebuilt binary in GitHub and Carthage will use that instead of compiling the framework itself when possible.

> Note that the frameworks that Carthage compiles also include simulator binaries. In iOS 10 these won't upload to iTunes Connect and it is unclear at this time how the maintainers will respond and what they will do to work around this.

Carthage has a lot of flexibility, but with great flexibility comes great confusion. If you exercise your right to stray from the path that the documentation lays out, you will be on your own in terms of configuring your project correctly.

Carthage strongly discourages you from checking in the built frameworks into version control (git). This means that you must rebuild whenever you pull down changes to the dependency graph, which can take quit a while because it uses a release configuration (which enables extra optimization) and builds for both the simulator and device. All this means that someone cannot just download your project and build it out of the box.

Carthage is also still somewhat new and a lot of open source projects haven't embraced it quit yet. Carthage claims that there is little to no configuration required for a project to be compatible, but in reality few are setup the way it expects. When I migrated one project to Carthage, I had to add support for it myself to about half a dozen projects, but most were open to merging those changes in. One project merged in my change, but hasn't released a new version with that support in 6 months. Another project supports it, but only on a branch which breaks version tags.

## [CocoaPods](https://cocoapods.org)

By *far* the most popular solution to dependencies is CocoaPods. It is heavily influenced by RubyGems and actually written in Ruby. Indeed, I would attribute the explosion of open source iOS code to CocoaPods.

CocoaPods approach is to try and make integrating dependencies as easy as possible. Like Carthage you create a file with the dependencies and versions you want, and it downloads everything. But instead of building that code, it generates Xcode project files that connect to your project. Theoretically, you just need to list your dependencies and then they will be available to use without any further work. When CocoaPods work, they work really well. But as Xcode changes, inevitably things break and the curtain gets pulled back. Sometimes you are left with no solution other than to wait for the maintainers to fix the issue.

If you are new to iOS development, I would highly recommend you start with CocoaPods. It's easy to setup and simple to use. When it does break, because there are so many people using it, it usually gets fixed fairly rapidly. Version 1.0 was released a few months ago and it has become much more stable than it has been in past years. If you haven't tried CocoPods since 1.0 was released, I would encourage you to give it another try.

## [Swift Package Manager](https://github.com/apple/swift-package-manager)

SwiftPM is extremely new. It was announced when Swift itself was open sourced and is still in active development. Despite the name it does support ObjC, C and C++. It's an interesting project given that it was created inside of Apple, who normally doesn't acknowledge the need for 3rd party dependencies. It handles the entire build process, downloading and compiling not just your dependencies, but integrating and compiling the app itself as well.

It is unclear though at this time how it will integrate with macOS and iOS apps. It's main focus is on server side Swift and in fact it will not work with iOS projects.

Something I find really interesting about SwiftPM is that it only ever statically links dependencies. If you create a framework in Xcode you can only dynamically link it. While there are benefits to dynamic linking (particularly related to extensions on iOS that share dependencies with the main app), static linking is much more efficient.


[^1]: At WWDC this year I talked to several Apple Xcode and Swift engineers, that had no idea how Cocoapods or Carthage worked.
[^2]: Technically I've only used Carthage in a TestFlight beta so far.
[^3]: Swift throws a really big wrench into this theory, at least until it reaches ABI stability. Until then (planned for Swift 4 a year from now), a Swift framework must be compiled with the exact same version of the compiler as the app it is linked with.
[^4]: While Carthage can use submodules, it prefers to download code itself, albeit using git.