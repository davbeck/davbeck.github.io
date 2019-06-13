---
layout: post
title: Digging deeper into the Swift behind SwiftUI
date: 2019-06-11
tags: [Swift,SwiftUI,Apple,iOS,macOS,Xcode]
---

I don't know about you, but as awesome as SwiftUI looks, I can't help but wonder at how it's actually implimented. As Swift has "evolved" over the years, I feel like I've been able to keep up with how things actually work under the hood. Swift tends to build apon itself. For example, if you understand how properties work, it's easier to understand how subscripts work (they are just properties with an argument).

But this year SwiftUI has really pushed on my understanding of how Swift works. [John Sundell did a good job going over the high level features that power SwiftUI](https://www.swiftbysundell.com/posts/the-swift-51-features-that-power-swiftuis-api). But even with this overview, and reading through the related proposals, I've still been very surprised by some of the behaviors of SwiftUI.

What follows are just a few things I've run into that made me scratch my head.

## View returning functions

Here's a pretty typical example of a SwiftUI view:

```swift
struct Row: View {
	var body: some View {
		HStack {
			Image(systemName: "star.fill")
			Text("Hello World!")
			Spacer()
		}
		.padding()
	}
}
```

It's quit clean and looks a bit like HTML. Notice that the elements under the `HStack` are just declared one after the other. Normally, if you wanted to do something like this in Swift, you would need to return an array of those values. With [function builders](https://forums.swift.org/t/function-builders/25167), the compiler will automatically translate this for you. That's easy enough to grasp.

But what about the body property? That looks similar, but what if we put a second view in there:

```swift
struct Row: View {
	var body: some View {
		Text("Another one!")
		HStack // ...
	}
}
```

Suddenly we get a compile error: `function declares an opaque return type, but has no return statements in its body from which to infer an underlying type`.

What's happening? The body property isn't using function builders! Instead, it's expecting us to return a single view. And to avoid having a `return`, it's taking advantage of a new feature, [implicit returns from single-expression functions](https://github.com/apple/swift-evolution/blob/master/proposals/0255-omit-return.md). This essentially gives functions and properties the same behavior as closures where they can implicitly return their last line... but only if they only have a single line. So when we add an extra line to the body (in this case another view, but it could have been a print statement) the compiler freaks out because it doesn't think you're returning anything anymore.

2 new Swift features... that look very much alike... but are actually completely different.

## State details

Let's replace our label with a text field:

```swift
struct Row: View {
	struct User {
		var email: String
	}
	@State var user = User(email: "Hello")
	
	var body: some View {
		HStack {
			Image(systemName: "star.fill")
			TextField($user.email)
			Spacer()
		}
		.padding()
	}
}
```

Another new Swift feature, [property wrappers](https://forums.swift.org/t/pitch-3-property-wrappers-formerly-known-as-property-delegates/24961) power a lot of SwiftUI. By marking our property with `@State` we get some update magic (more on that later). After reading the proposal, we learn that `State` is just a struct that wraps it's property. In theory, it's pretty easy to reason about. `self.user` gives us the wrapped value, while `self.$user` gives us the actual struct.

But what about that `$user.email`? If `$user` returns the `State` struct, wouldn't we be pulling a value from that struct?

I got the answer to this one from [Session 415:Modern Swift API Design](https://developer.apple.com/videos/play/wwdc2019/415/). Turns out, there's another new Swift feature that is being used here: [key path member lookup](https://github.com/apple/swift-evolution/blob/master/proposals/0252-keypath-dynamic-member-lookup.md). I completely missed this when it went through evolution. It expands `@dynamicMemberLookup` to use key paths as an alternative to strings. What's actually happening with `$user.email`, is a dynamic member lookup, using key paths from the wrapped value, to return the child property... also wrapped in a `Binding` struct (with a type `Binding<String>`).

## How does it know about state?!?!

Apple says that when you wrap a property with `@State`,
SwiftUI "sees" when it's used in the build method.

How? What is state doing that allows SwiftUI to know when it's used.
How does it trigger a re-render on change?
Is it using thread local variables (not a good practice)?
Or since it only works on the main thread, maybe there is a global render object
(also not a good practice)?
Are they looking at the call stack?
That also doesn't seem like a good idea, 
but it's something Apple has done in the recent past.

## State being converted into Binding

Going further with our text field above, the reason we pass in `$user.email` is because it takes a binding so it can update the value. We actually can't pass in a static string. The dynamic member lookup wraps the property it returns into a binding so all is kosher.

But what if we were just using a state property directly?

```swift
struct Row: View {
	@State var email = "Hello"
	
	var body: some View {
		HStack {
			Image(systemName: "star.fill")
			TextField($email)
			Spacer()
		}
		.padding()
	}
}
```

I would think that wouldn't work, but it does. `TextField` takes a `Binding`, but here we are passing in a `State`.

How does this work? I don't know. `State` does conform to a protocol, [`BindingConvertible`](https://developer.apple.com/documentation/swiftui/bindingconvertible), but how this gets translated into a `Binding` is a mystery to me. It's behavior kind of reminds me of [_ObjectiveCBridgeable](https://swiftdoc.org/v1.2/protocol/_objectivecbridgeable/), but there might be a less magical explenation.

## Generated initializers and wrappers

Speaking of init params, let's define our own views with bindings:

```swift
struct MyTextField: View {
	@Binding var text: String
	
	var body: some View {
		TextField($text)
	}
}
```

I didn't specify an intializer here, but Swift of course generates one for us (thank you Swift). If we drop this into our example from earlier, autocomplete tells us that the generated init wants a `Binding`.

But what if we swap out `Binding` for `State`? (I don't think you would actually want to do this, but just go with it.) Well now the compiler complains that it can't convert `Binding<String>` to `String`. The generated init method all of a sudden changes from wanting the wrapper to wanting the wrapped value. But why?!?

I figured this one out from session 415 as well.
It comes down to how those wrappers are defined. 
`State` has the following init method:

```swift
init(initialValue value: Value)
```

This is a magic format for property wrappers. When you have an init method of that form, Swift knows how to create the wrapper with an initial value.

`Binding` on the other hand has several public init methods, but none that match that format. So Swift can't create it on it's own with a value, so it falls back to requiring you to manually create (or set) the wrapper.

Side note, you can actually initialize wrappers directly in your init method as well.
For instance, to expand upon the `UserDefault` example Apple has been using, 
we can allow customizing which `UserDefaults` instance gets used (along with a default):

```swift
@propertyDelegate
struct UserDefault<T> {
	let userDefaults: UserDefaults
	let key: String
	
	var value: T {
		get {
			return userDefaults.object(forKey: key) as! T
		}
		set {
			userDefaults.set(newValue, forKey: key)
		}
	}
}

class Foo {
	@UserDefault var bar: String
	
	init(userDefaults: UserDefaults = .standard) {
		$bar = UserDefault(userDefaults: userDefaults, key: "com.me.Foo.bar")
	}
}
```

## Some what?

When you define your views, they have 1 requirement: the body property. It returns another view. But not just *any* view! It returns `some View`. What's the deal?

The `View` protocol has an associated type [`Body`](https://developer.apple.com/documentation/swiftui/view/3125355-body). It's similar to how `Collection` has an associated type for it's elements. It's a form of generics.

Normally, if you have a type that conforms to a protocol with an associated type, you need to define that type **somehow**. Either with a typealias, or implicitly like so:

```swift
struct Row: View {
	var body: HStack {
		HStack {
			Image(systemName: "star.fill")
			TextField($email)
			Spacer()
		}
		.padding()
	}
}
```

Except that won't work. `HStack` is also generic. The actual return type should be: `_ModifiedContent<HStack<TupleView<(Image, Text, Spacer)>>, _PaddingLayout>`. Kind of a mouthful huh? And it would change if we changed our hierarchy. Essentially every view we render in the entire hierarchy ends up being in that type, and then some. If you change the text color for instance, the `Text` part of that type becomes `_ModifiedContent<Text, _EnvironmentKeyWritingModifier<Color?>>`.

**This is generics on steroids. 😳**

And we can't just reuturn `View`. `Protocol 'View' can only be used as a generic constraint because it has Self or associated type requirements`. You may have seen an error like that if you've ever tried to use `Equatable` like a regular protocol.

SwiftUI does come with [AnyView](https://developer.apple.com/documentation/swiftui/anyview), which is a type erased view, but you have to explicitely wrap your result in it. From what I've heard, using it has performance implications. 
Not to mention, how this works (it's `Body` type is `Never` 😅) just leads to more questions.

Obviously normal users would never be able to figure out what their associated type should be without some help. So, we have [opaque result types](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md) instead. `some View` basically asks the compiler to figure out what our return type is. But the type is still there. That's actually important for performance reasons though...

Since it's inception, Swift has tried to hide as much of it's safety and performance features as possible. For example, it's **very** strongly typed. Everything must have a very specific type. But if you just type `var foo = "bar"`, the compiler will figure out the type for you. All while maintaining type safety and performance.

Generics have never quit reached that level of simplicity though. `foo.map{ $.count }` will figure out that you want to convert an array of strings into an array of integers, but when you start creating really complicated expressions it gets confused fast. Make a mistake in the wrong spot, and you'll get a seemingly completely unrelated error several lines away.

And I've seen SwiftUI get **very** confused. At WWDC, Apple kept advertising how easy it was to break out parts of the view hierarchy into their own smaller views, and how it wouldn't effect performance like creating extra `UIView` wrappers would. But if you **don't** want to break up your views, you'll quickly find that the type you are returning gets **very** complicated, and often the compiler gets confused in the process.

---

SwiftUI definitely seems to be pushing the boundaries of what is possible in Swift. For years we've been talking about what a truly Swift native framework would look like, and I think this is the answer. By dropping interoperability with Objective-C, the framework is free to really dig deep into the language.

I'm curious to see how this plays out. Will we continue down this path? Will we realize that all these fancy features are a bit much and pull back a little? Time will tell.

When object oriented languages became popular, deep levels of subclasses became popular.
One class would subclass another, which would subclass another, and so on and so forth.
With time we realized that it was more useful to have shallow sublcass hierarchies.
Perhaps we'll see a similar trend with generics.
Or maybe we'll figure out a way to tame the generics dragon. 🤷🏻‍♀️