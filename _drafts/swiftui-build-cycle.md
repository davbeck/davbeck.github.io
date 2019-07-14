---
layout: post
title: Working out the SwiftUI build cycle
date: 2019-06-11
tags: [Swift,SwiftUI,Apple,iOS,macOS,Xcode]
---

Intro

## Plain variables

The simplest form of data in SwiftUI is a plain variable:

```swift
struct Child: View {
	var value: Int
	var body: some View {
		Text("test")
	}
}

struct Parent: View {
	var count: Int = 0
	var body: some View {
		VStack {
			Child(value: 5)
			Text("count: \(count)")
		}
	}
}

let viewController = UIHostingController(rootView: Parent())
Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { (timer) in
	viewController.rootView.count += 1
}
```

When we change the count in the parent view, it triggers a re-render and the body gets generated. You might think that would cause the entire hierachy to get re-rendered, but that's not the case here.

How does SwiftUI know not to re-render the child? The simplest answer would be that `View` impliments `Equatable` and SwiftUI would just compare the body output. However `View` isn't equatable.

It gets interesting when you change the type of the variable. 
If the value does change between renders, for instance if we use the count instead, the child does re-render just as we would expect. 
If you use a struct, even one that doesn't conform to `Equatable`, it will intelligently re-render.
If you use a class, it will always trigger a re-render, even if it is `Equatable` unless the instances are the exact same object (ie `===`). 

That behavior holds true if you include an object in a struct. Meaning that a struct will trigger a re-render if it has a different instance of an object. This can get insteresting when you have value types that internally use classes for their storage, in particular types that use copy on write. If you wrap a value in an array in your body, it will trigger a re-render in the child every time, even though the actual value hasn't changed.

```swift
struct Parent: View {
	var count: Int = 0
	var body: some View {
		VStack {
			Child(value: [5]) // this will cause Child to re-render every time
			Text("count: \(count)")
		}
	}
}
```

It's unclear how SwiftUI is handling this. My guess is that they are using [`Mirror`](https://developer.apple.com/documentation/swift/mirror) to inspect the properties of the view. I would love to know why Apple didn't use `Equatable` (or `Hashable`) here.

Regardless, this is a common theme in SwiftUI. You should expect `body` to be called at anytime. Make sure you aren't doing any work there, or making assumptions about when it gets called.

It's also important to remember that a re-render (as I call it) doesn't cause the actual underlying views to be recreated. Your `body` is a representation of view state, but the beauty of SwiftUI is that it will inteligently update the screen when it needs.

## @State

In the previous example, I modified the parents variable directly from the view controller. As far as I know, that's something you can only do at the top level. Because `View`s are structs you can't mutate them normally.
That's what `@State` is for. Let's move our count timer into the Parent View:

```swift
struct Parent: View {
	@State var count: Int = 0
	var body: some View {
		return VStack {
			Child(value: count)
			Text("count: \(count)")
		}
		.onAppear {
			Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { (timer) in
				self.count += 1
			}
		}
	}
}
```

When we mark a view's property with `@State`, SwiftUI will keep track of the value for us. This can be kind of hard to wrap our minds around (at least it is for me). Whenever a parent view re-renders, it creates brand new children. `init` gets called, default values get generated etc. However @State variables will maintain a consistent value between different instances as long as they are in the same rendered hierarchy.

For instance, if our child has it's own state:

```swift
struct Child: View {
	var value: Int
	@State var status: Int = 0
	var body: some View {
		print("render child")
		return Text("test: \(value) / \(self.status)")
			.tapAction {
				self.status += 1
			}
	}
}
```

As the parent re-renders and passes in different values for `value`, the child will get updated. However, even as that value changes, the @State will be maintained.

This poses a bit of a challenge for derived state. ...

We can manually trigger a state reset with `IDView`:

```swift
struct Parent: View {
	@State var count: Int = 0
	var body: some View {
		return VStack {
			IDView(Child(value: count), id: count)
			Text("count: \(count)")
		}
		.onAppear {
			Timer.scheduledTimer(withTimeInterval: 1, repeats: true) { (timer) in
				self.count += 1
			}
		}
	}
}
```

As far as I can tell though, there's no way to do this at the child view level. For instance, a view that loads an image based on a url would want to always want to reset it's downloaded image when the url changes.