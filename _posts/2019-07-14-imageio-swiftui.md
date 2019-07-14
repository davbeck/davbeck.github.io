---
layout: post
title: Animating and Downloading Images Incrementally with SwiftUI
date: 2019-07-14
tags: [SwiftUI,iOS,ImageIO,Xcode]
---

When Apple released their new view framework, SwiftUI, at WWDC 2019, they had several sessions throughout the week to breakdown and explain how to use the framework. There was a big hole that people are still trying to figure out though: how do you load images! To be fair, this has long been a void in the Apple frameworks. UIKit doesn't have any builtin way to download and show images either. But given the radical new design of SwiftUI, it's not obvious how it should be done.

I have a library called [ImageIO.Swift](https://github.com/davbeck/ImageIOSwift) that uses [Image I/O](https://developer.apple.com/documentation/imageio) to animated and load images incrementally. Basically it gives you a UIView that works like an [html `img`](https://www.w3schools.com/tags/tag_img.asp) tag. So I figured why not add support for SwiftIU! How hard could it be 😅.

<span class="side-note">
I've been thinking of changing the name of this library. Originally the idea was that it would be a lightweight wrapper around Image I/O similar to the way GCD gets adapted into a more friendly Swift interface. However the feature that I use the most is the image source view, which handles downloading and animating images. There is no equivalent in Image I/O. If you have any suggestions, let me know.
</span>

If you're just interested in using the library, check out the [beta 1.0.0 release](https://github.com/davbeck/ImageIOSwift/tree/1.0.0). To learn more about how the SwiftUI integration was implemented, read on!

---

The simplest way to display an image source in SwiftUI would be to use a [UIViewRepresentable](https://developer.apple.com/documentation/swiftui/uiviewrepresentable) view and just wrap the existing UIView implementation of `ImageSourceView`. That would have been pretty quick and simple. But who wants that!

The big drawback there though is that it would only work with UIKit apps. To get the real cross platform benefit of SwiftUI, including watchOS and AppKit Mac apps, I'd need to use a more native view implementation.

## Downloading images

The first challenge I ran into with SwiftUI was handling derived state. ImageIO.Swift uses `Task`s derived from urls to track downloading. We don't really want tasks to be replaced for fear of the download starting again.

We can use `@State` to get SwiftUI to track our task for us. The task isn't actually changing though, which feels weird, but is the only way I see to accomplish this.

```swift
struct RemoteImageSourceView: View {
	@State var task: ImageSourceDownloader.Task
	var body: some View {
		return ImageSourceView(imageSource: task.imageSource)
		.onAppear {
			self.task.resume()
		}
	}
}

struct URLImageSourceView: View {
	var url: URL
	var body: some View {
		return RemoteImageSourceView(task: self.imageSourceDownloader.task(for: self.url))
		.id(url)
	}
}
```

Each time `URLImageSourceView` gets evaluated, it will create a new task, but it will actually get discarded in favor of the previous state. I made some changes to the downloader to make tasks lazy, similar to the way `URLSession` tasks work. So creating extra tasks that get thrown out immediately shouldn't be a performance issue.

That's how we can maintain a derived state, but if the url changes, we actually want to reset the task and track a new one. That's what [`id(url)`](https://developer.apple.com/documentation/swiftui/view/3278578-id) does. When the value you pass to `id` changes, it will reset any state.

Note that we don't need to worry about something like `@ObjectBinding` here because the changes to a task are primarily in it's `imageSource`. That gets created immediately and updated as data becomes available. We can bind to that instead.

It's a good idea to wait until `onAppear` to actually start any work. As people are finding out, things like [NavigationLink](https://developer.apple.com/documentation/swiftui/navigationlink) evaluate their children immediately, but don't call `onAppear` until it's actually shown. But beware, `onAppear` and `onDisappear` can be called more often than you might think. Even if the view you attach them too isn't being added and removed, if their only visible child changes down the hierarchy, these will get triggered. For that reason, I don't explicitly call cancel on the task because it might "disappear" and "reappear" without the task ever actually changing. Instead, the task will be cancelled implicitly when it gets released (a pattern Swift is using more and more).

## Animating

Animation also needs a kind of derived state. The animations are driven by a [CADisplayLink](https://developer.apple.com/documentation/quartzcore/cadisplaylink) (or a simple timer if it's not available).

We could adapt a display link into a [`BindableObject`](https://developer.apple.com/documentation/swiftui/bindableobject), but the bigger challenge is where do we create this object. Every time the parent gets re-evaluated, a new instance of the object would get created if we just created a default:

```swift
struct AnimatedImageSourceView: View {
	// WRONG!
	@ObjectBinding var displayLink = DisplayLink()
}
```

Instead, I created a custom [`Publisher`](https://developer.apple.com/documentation/combine/publisher) that waits until the user subscribes to it before creating the display link. It then invalidates it when it gets cancelled.

```swift
struct AnimatedImageSourceView: View {
	@ObjectBinding var imageSource: ImageSource
	var displayLink: DisplayLink
	@State var startTimestamp: TimeInterval? = .none
	@State var animationFrame: Int = 0
	var label: Text
	
	init(imageSource: ImageSource, label: Text) {
		self.imageSource = imageSource
		self.label = label
		self.displayLink = DisplayLink(preferredFramesPerSecond: imageSource.preferredFramesPerSecond)
	}
	
	var body: some View {
		return StaticImageSourceView(imageSource: imageSource, animationFrame: self.animationFrame, label: label)
			.onAppear {
				self.startTimestamp = CACurrentMediaTime()
			}
			.onReceive(displayLink) { targetTimestamp in
				if let startTimestamp = self.startTimestamp {
					let timestamp = targetTimestamp - startTimestamp
					self.animationFrame = self.imageSource.animationFrame(at: timestamp)
				}
			}
	}
}
```

We don't need to worry about this being re-created. It is a struct with only one variable. It only does work when it gets subscribed to, and then all of its state is tracked within a subscription object, which SwiftUI keeps track of.

This is something that is a little hard for me to wrap my head around, but based on [this forum post](https://forums.swift.org/t/a-uicontrol-event-publisher-example/26215/9) I think is the right way to think about publishers. The actual publisher is just a template for a subscription, which then manages the state and triggers work to be done.

## Handling orientation

One of the sticking points with using a [CGImageSource](https://developer.apple.com/documentation/imageio/cgimagesource-r84) directly instead of a [UIImage](https://developer.apple.com/documentation/uikit/uiimage) is that images with a non-standard [exif orientation](https://www.daveperrett.com/articles/2012/07/28/exif-orientation-handling-is-a-ghetto/) don't get adjusted. Exif orientation is used to avoid re-encoding images from a camera. Instead of taking a JPEG, rotating the pixels and then re-compressing it (which would lead to artifacts), software, especially phones, will just mark the image as rotated from the original.

When creating a UIImage from a CGImage, you can specify the orientation and things like UIImageView will adjust it appropriately. But funny enough, SwiftUI doesn't handle this, even with images loaded from an asset catalog. So at least until that bug gets fixed (and yes, I did file a <s>radar</s> feedback) ImageIO.Swift has yet another advantage over plain `Image`s 😆.

But there were a few challenges with correcting the orientation. Generally, making the adjustment is just a matter of combining -1 transformations in the x and y directions with rotations left or right. The transformations were easy enough:

```swift
Image(cgImage, scale: 1, label: Text(""))
.scaleEffect(x: properties.scaleX,
             y: properties.scaleY,
             anchor: .center)
```

That's all that's needed for half of the orientations (1-4). But the other half actually involve a 90° rotation left or right. This actually changes the frame of the image from portrait to landscape or vice versa. Here's where it gets tricky. SwiftUI has a [`rotationEffect`](https://developer.apple.com/documentation/swiftui/image/3269732-rotationeffect) modifier, but it doesn't effect its bounding box. So if you had a landscape image that was rotated into portrait and you set its width to 200 pixels, it would actually be smaller than expected because the bounding box would be set to that width, and the the image rotated.

I came up with an interesting solution to this: using [`overlay`](https://developer.apple.com/documentation/swiftui/view/3278622-overlay) to place the actual image above a placeholder view. Using the image source's size as a preferred frame:

```swift
struct ImageSourceBase: View {
	var properties: ImageProperties
	
	var body: some View {
		Rectangle()
			.fill(Color.clear)
			.frame(idealWidth: properties.imageSize?.width, idealHeight: properties.imageSize?.height)
	}
}
```

The actual image can then be placed over it with it's various transforms:

```swift
struct StaticImageSourceView: View {
	@ObjectBinding var imageSource: ImageSource
	var animationFrame: Int = 0
	var label: Text
	
	var body: some View {
		let image = self.imageSource.cgImage(at: animationFrame)
		let properties = imageSource.properties(at: animationFrame)
		
		return ImageSourceBase(properties: properties)
			.overlay(
				image.map { Image($0, scale: 1, label: self.label)
					.resizable()
					// adjust based on exif orientation
					.rotationEffect(properties.rotateZ)
					.scaleEffect(x: properties.scaleX,
					             y: properties.scaleY,
					             anchor: .center)
				}
			)
	}
}
```

While the image source is available immediately from the download task, it doesn't necessarily have enough data to render anything, or even know what size it is. I struggled a big to figure out how to conditionally show and hide the image when it wasn't available. But, as it turns out, an optional `View` is itself a valid `View` thanks to conditional conformance. When its nil, nothing will be shown, when it's non-nil, it will be shown.

## Resizable? 🤷🏽‍♂️

Another thing you might notice in that code snippet is that I'm marking the image as resizable. I'll be honest, I'm not sure what Apple was going for here. Views can have an "ideal" size, which seems to be equivalent to [`intrinsicContentSize`](https://developer.apple.com/documentation/uikit/uiview/1622600-intrinsiccontentsize), which is what `UIImageView` uses.

However [`resizable`](https://developer.apple.com/documentation/swiftui/image/3269730-resizable) is specific to `Image`s. Most view modifiers like that return a wrapper around their original view, but `resizable` returns a plain `Image`. This means that it must be the first in a chain of modifiers:

```swift
// this won't compile
Image("foo")
	.background(Color.blue)
	.resizable()
// Value of type '_ModifiedContent<Image, _BackgroundModifier<Color>>' has no member 'resizable'

// instead, you have to order it like this
Image("foo")
	.resizable()
	.background(Color.blue)
```

It also means that we can't easily wrap an `Image` and maintain the ability to mark it as resizable or not.

Until I understand the purpose of this more, I'm always including it, because as far as I can tell, that's what you would always want. For icons its perhaps more useful to have a fixed size, but images sources are primarily used for other types of images.

## Packaging it all up

At WWDC, Apple pointed out that SwiftUI doesn't include prefixes for it's types. It's a true Swift only framework! Ironically though they are able to get away with that because all of their other view frameworks use prefixes. You don't confuse `View` for a `UIView` for instance, both in your mind and in your compiler. 

For Apple's other frameworks like MapKit, this hasn't been an issue in the past either. Even though there are 2 versions of `MKMapView` (one for UIKit and one for AppKit), they are only available on different platforms. When they eventually do add a SwiftUI version of the view, it will probably just be named `MapView`.

That's great for Apple, but for existing Swift native frameworks like ImageIO.Swift, we are already using non-prefixed classes. The natural name for the SwiftUI view here would be `ImageSourceView`. But that's exactly what I'm already using for the `UIView` subclass.

After some deliberation, I decided to break up the library into several packages/podspecs. ImageIOUIKit has all the UIKit adaptations, including an `ImageSourceView`, which is a subclass of `UIView`. Meanwhile, `ImageIOSwiftUI` includes a `SwiftUI.View` implementation of `ImageSourceView`. And technically these can be used together with their full namespace: `ImageIOSwiftUI.ImageSourceView` and `ImageIOUIKit.ImageSourceView`. But I think most people will end up only using one or the other.

---

The final result works really well. Images are downloaded correctly, appearing incrementally as data becomes available, and start to animate once they are loaded.

Here's what the public API ends up looking like:

```swift
URLImageSourceView(url: url, isAnimationEnabled: true, label: Text("alt text"))
	.aspectRatio(contentMode: .fit)
	.frame(width: 300)
```

Or, if you already have a reference to an `ImageSource`:

```swift
ImageSourceView(imageSource: imageSource, isAnimationEnabled: true, label: Text("alt text"))
	.aspectRatio(contentMode: .fit)
	.frame(width: 300)
```

I'm including this integration in a [1.0.0](https://github.com/davbeck/ImageIOSwift/tree/1.0.0) update I'm planning on releasing in the Fall after the Xcode GM is available, but in the meantime you can give it a try in your own SwiftUI apps using SwiftPM.

There has been quit a bit of talk about [some additional api in Image I/O](https://developer.apple.com/documentation/imageio/3333271-cganimateimageaturlwithblock) that supposedly adds support for animated images. It is currently undocumented and unavailable in Swift (not to mention its iOS 13 only). It will be interesting to see if it is using any special tricks when it's finally available, but for now, ImageIO.Swift has got your back!