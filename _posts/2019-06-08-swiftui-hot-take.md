---
layout: post
title: My SwiftUI Hot Take 🔥
date: 2019-06-08
tags: [Swift,SwiftUI,Apple,iOS,macOS,Xcode]
---

SwiftUI looks amazing. There have been a lot of attempts to bring React concepts to Swift. This is the first implimentation I've seen that looked right.

## Only Apple could do this

Part of that is because Apple is backing the framework. You don't have to worry about it integrating well with UIKit/AppKit. You also don't have to worry about being punished next year for using something non-standard. You can be sure that Apple will support andy new features from the OS in SwiftUI.

But the big reason it works where other frameworks didn't is because Apple actually added several new features to Swift itself. [SE-0244 (enabling `some View`)](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md) was approved recently. 2 other features are so new they don't even have evolution numbers. [Property wrappers (enabling `@State var`)](https://forums.swift.org/t/pitch-3-property-wrappers-formerly-known-as-property-delegates/24961) has gone through several pitch rounds and is still being ironed out. [Function builders (enabling implicit returns of multiple children)](https://forums.swift.org/t/pitch-function-builders/25167) wasn't even mentioned until SwiftUI was announced.

This is clearly only something Apple can do. Sure, theoretically Swift Evolution is an open process and anyone could have made these pitches, but Apple is using a pretty heavy hand to get them pushed through. I think it's especially telling that they waited until after announcing SwiftUI to introduce the property builders pitch. I can imagine that if that had been introduced a few weeks ago, it might have easily been rejected for not being Swifty enough. The details of those features may change over the summer, but I garuntee you that some form of them will pass review by the time the Xcode 11 GM ships.

## At least for now, SwiftUI is high level

The actual engine that renders views is versitile enough to render even the smallest details of a UI. But the views that Apple has built in are all generic abstractions. That makes it great for hight level abstractions that can be adapted to multiple platforms. It makes it a little more difficult to fine tune your UI. Will your view render a UILabel or an NSMenuItem? You're not suppose to know. We'll see where this goes.

## There are going to be things that are hard

Reactive programming is really useful and powerful, but there are still interactions that are easier to do with imperitive code. Even more so, it's going to be a big shift in how we think about rendering UI.

## I wouldn't hold my breath for other platforms

A lot of people are talking about SwiftUI on Android or the Web. It's an obvious next step. Apple has made an **excellent** cross platform framework, and people immediately want to use it for all the platforms. But I highly doubt Apple will ever add support for Android or the Web. And furthermore, it wouldn't be native on those platforms either unless Google abandons their react like frameworks and adopts Swift (unlikely).

## Xcode 11 isn't as polished as the demos imply

On my 6 Core/32GB MacBook Pro previews take a lot longer to render than they did in those demos. Sometimes they pause rendering for some reason. This is with the very basic templates. Nothing fancy. I'm not sure what those demos are doing differently (perhaps that's why Apple really needed to make the Mac Pro) but your milage may vary.

## It's still early stages

Don't get me wrong, you can build production apps with this come the Fall, but you might not want to. This feels a lot like Swift 1.0. Back then a lot of people (myself included) paniced and felt like ObjC would be deprecated immediately. Those who stuck with ObjC know that it hasn't gone anywhere. UIKit will still work great for years to come. You have time. Which brings me to:

## The fact that it's iOS 13/macOS 10.15/etc only is a good thing

Swift suffered from too much attention in it's infancy. I think the fact that most people won't be able to ship anything for a year or 2 with this will help SwiftUI breath and develop more gracefully.