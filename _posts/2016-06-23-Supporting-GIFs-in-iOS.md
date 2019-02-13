---
layout: post
title: Supporting GIFs in iOS
date: 2016-06-23
tags: [iOS, UIKit, NSFileAttachment, GIF]
redirect_from:
  - /blog/2016/06/23/Supporting-GIFs-in-iOS.html.html
---

A few weeks ago I got a bug report from my tester. I quote: "Animated GIFs don't animate. This is inconsistent with the web experience—and it's just not *fun*." We definitely want our app to be fun! But we never really built animated GIF support into our web app. It just kind of happened. If you display a GIF on a web browser, it's going to animate it. It's just what it does. It would be harder to keep them from animating.

GIFs are awesome, obviously. Emoji may have taken over the world, but when people start to look further, to take their playful graphics to the next level, they turn to dem moving pictures that have been popular for decades. Sure there are [more efficient formats available](http://blog.imgur.com/2014/10/09/introducing-gifv/), and [higher quality formats are pushed by Apple](https://en.wikipedia.org/wiki/APNG). But the common GIF still dominates in one form or another across the entire internet.

In iOS 8 Apple introduced support for [custom keyboard extensions](https://developer.apple.com/library/ios/documentation/General/Conceptual/ExtensibilityPG/CustomKeyboard.html). As with many of their APIs, it is clear that they had one idea for them, but developers immediately saw their potential and started creating [tons](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=2&cad=rja&uact=8&ved=0ahUKEwiaq-7u76jNAhWDKGMKHVvSA3AQFggmMAE&url=https%3A%2F%2Fwww.riffsy.com%2F&usg=AFQjCNFIPuYEB0-QgjKKupqg08kpBlStZQ&sig2=UMX6k9XE5dkP2Gm2uxvdEQ) [and](https://giphy.com/gifkeyboard) [tons]((https://popkey.co)) of GIF keyboards. There's a catch though. Apple only supports entering text from a keyboard. So all these GIF keyboards rely on the pasteboard to get the GIF into the app. Typically a user taps on the GIF they want, it is copied and the keyboard instructs the user to paste the new image into the app they are using.

But there's another catch, iOS doesn't even come close to supporting animated GIFs out of the box[^1] [^2].

But, it is possible. Let's walk through what it takes to add awesome GIF support to our apps. We can even take things a step further than apps like Messages and actually automatically paste the GIF when it is copied by the keyboard.

---

To start, we are going to use the fantastic [FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage). There are a few different frameworks floating around on GitHub to support animated GIFs, and I've used a few of them, but this library is hands down the one you should be using. Check their README for more information on why, but I can vouch that the library is performant and stable without taking over your app's resources.

For the purpose of this post, I'm also going to use [SlackTextViewController](https://github.com/slackhq/SlackTextViewController) for the text input UI, but you can use anything based on `UITextView`.

You can view the [sample code for this post on GitHub](https://github.com/davbeck/GIFFun). To get the starting point for this post, check out the [starting-point tag](https://github.com/davbeck/GIFFun/releases/tag/starting-point) tag. I'm using CocoaPods to include both FLAnimatedImage and SlackTextViewController. The app is just a sample chat interface, without any backend. You can type text into the field and hit send to add it to the chat stream. I've disabled the [autohide send button feature](http://cocoadocs.org/docsets/SlackTextViewController/1.8/Classes/SLKTextInputbar.html#//api/name/autoHideRightButton) in SlackTextViewController, because it can cause some weird animations when using text attachments.

Let's start by adding support for pasted GIFs. While there is [a property on `UITextView` to enable editing attributed text](https://developer.apple.com/reference/uikit/uitextview/1618622-allowseditingtextattributes), it only adds support for bold, italic, underline in the edit menu, and similar transformations via paste. It does not add support for pasting attachments. We need to intercept the `paste:` action in the `UITextView`. To do this with SlackTextViewController, we call [registerClassForTextView](http://cocoadocs.org/docsets/SlackTextViewController/1.8/Classes/SLKTextViewController.html#//api/name/registerClassForTextView:) with our custom [SLKTextView](http://cocoadocs.org/docsets/SlackTextViewController/1.8/Classes/SLKTextView.html) subclass. Again, if you aren't using SlackTextViewController, you can just subclass UITextView directly and use that in your project.

```swift
override func canPerformAction(action: Selector, withSender sender: AnyObject?) -> Bool {
	// if it's an action our superclasss can handle, let it
	if super.canPerformAction(action, withSender: sender) {
		return true
	}
	
	// if the pasteboard has images on it, make sure paste gets enabled
	return UIPasteboard.generalPasteboard().containsPasteboardTypes([ kUTTypeImage as String ]) && action == #selector(paste)
}
	
override func paste(sender: AnyObject?) {
	// we want to handle pasting images ourselves
	if UIPasteboard.generalPasteboard().containsPasteboardTypes([ kUTTypeImage as String ]) {
		// the pasteboard can contain any number of items, we want to handle them all
		for index in 0..<UIPasteboard.generalPasteboard().numberOfItems {
			let itemSet = NSIndexSet(index: index)
			
			// for GIFs, make sure we retain their type, for all other image types we can be generic
			let textAttachment: NSTextAttachment
			if let data = UIPasteboard.generalPasteboard().dataForPasteboardType(kUTTypeGIF as String, inItemSet: itemSet)?.first as? NSData {
				textAttachment = NSTextAttachment(data: data, ofType: kUTTypeGIF as String)
			} else if let data = UIPasteboard.generalPasteboard().dataForPasteboardType(kUTTypeImage as String, inItemSet: itemSet)?.first as? NSData {
				textAttachment = NSTextAttachment(data: data, ofType: kUTTypeImage as String)
			} else {
				continue
			}
			
			
			// this is how you add an attachment to an NSAttributedString
			// unfortunately, NSAttributedString(attachment:) isn't available for NSMutableAttributedString, so we need to copy it
			let attachmentString = NSMutableAttributedString(attributedString: NSAttributedString(attachment: textAttachment))
			// for convenience, we are adding a line break after an image
			// we will trim this out of our text when we send it later
			attachmentString.appendAttributedString(NSAttributedString(string: "\n"))
			
			let startingSelectedRange = self.selectedRange
			
			// if there is something selected, paste should replace it
			self.textStorage.replaceCharactersInRange(
				startingSelectedRange,
				withAttributedString: attachmentString
			)
			
			// move selection to the end of what we just inserted
			let endingSelectedRange = NSRange(location: startingSelectedRange.location + attachmentString.length, length: 0)
			self.selectedRange = endingSelectedRange
			
			
			// adding the attachment to an empty text view can wipe out any formatting set on it
			// you'll need to reset the basic attributes you want here, such as font, text color and so on
			self.typingAttributes = [
				NSFontAttributeName : UIFont.preferredFontForTextStyle(UIFontTextStyleBody),
			]
			self.textStorage.addAttributes(self.typingAttributes, range: NSRange(location: 0, length: self.textStorage.length))
		}
	} else {
		// all other pasted content can be handled by our superclass
		super.paste(sender)
	}
}
```

The way that the pasteboard works, an app can add any number of items, each with any number of representations. For instance, when the photos app copies an image, it includes a custom internal type that represents the id of the photo, and the JPEG representation of the photo. If the user selects multiple photos, each item would then have 2 representations. Further, UTIs (what `UIPasteboard` calls "`pasteboardTypes`") can inherit from other UTIs. `kUTTypeJPEG` inherits from `kUTTypeImage` for instance.

For our purposes, we only care about 2 types: `kUTTypeGIF`, and `kUTTypeImage`. `kUTTypeGIF` inherits from `kUTTypeImage`, but we need to know if the file is specifically a GIF, so we can handle it in a way that will maintain it's animations. Any other type (for instance, text) we simply pass on to the super implementation.

In order to display the images as attachments in the text view, we need to use `NSTextAttachment`. The class is a little bit of a dark corner in iOS, originating on OS X and being ported back. It has 2 main properties, `contents` and `fileType`. Something that wasn't immediately apparent to me was that the `image` property is not a way to customize the display of the attachment, but a convenience for the `contents` and `fileType` properties. On OS X, you can use a cell to customize the appearance, but that isn't available on iOS. *Unfortunately, that means that our attachments will not be animated until they are sent. This is consistent with the way the Messages app works.*

You can however customize the size the attachment is displayed at. The `bounds` property will set a constant size, but if you just want it to be the width of the text view, you can subclass NSTextAttachment and override [`attachmentBoundsForTextContainer`](https://developer.apple.com/library/ios/documentation/UIKit/Reference/NSTextAttachmentContainer_Protocol/index.html#//apple_ref/occ/intfm/NSTextAttachmentContainer/attachmentBoundsForTextContainer:proposedLineFragment:glyphPosition:characterIndex:). Just make sure to use that subclass when you are creating your attachments in the `paste` method.

```swift
class TextAttachment: NSTextAttachment {
	private var _imageSize: CGSize?
	
	/// A cached size for the image represented by the attachment.
	var imageSize: CGSize? {
		if let image = self.image {
			return image.size
		} else {
			if let size = _imageSize {
				return size
			}
			
			guard let data = self.contents ?? self.fileWrapper?.regularFileContents else { return nil }
			guard let image = UIImage(data: data) else { return nil }
			
			_imageSize = image.size
			
			return image.size
		}
	}
	
	override func attachmentBoundsForTextContainer(textContainer: NSTextContainer?, proposedLineFragment lineFrag: CGRect, glyphPosition position: CGPoint, characterIndex charIndex: Int) -> CGRect {
		guard let imageSize = self.imageSize else { return lineFrag }
		
		let width = lineFrag.size.width - 14
		let adjustedSize = CGSize(width: width, height: floor(imageSize.height / imageSize.width * width))
		return CGRect(origin: .zero, size: adjustedSize)
	}
}
```

This all looks lovely, but if you hit send at this point, you will only get text that is included with the images, and not the images themselves. How you convert the `NSAttributedString` into something that can be sent to your backend will depend on what that backend wants. In our case, our chat service creates a separate message for each image and each chunk of text. They all then get sent and grouped together. Here's how we break up the attributed string into those parts:

```swift
private func messagesWithAttributedString(attributedText: NSAttributedString) -> [Message] {
	var messages = [Message]()
	
	attributedText.enumerateAttributesInRange(NSRange(location: 0, length: self.textView.attributedText.length), options: []) { (attributes, range, stop) in
		if let attachment = attributes[NSAttachmentAttributeName] as? NSTextAttachment {
			guard let contents = attachment.contents ?? attachment.fileWrapper?.regularFileContents else { return } // block continue
			guard let image = UIImage(data: contents) else { return } // block continue
			
			let message = Message(senderName: "You", photo: image)
			messages.append(message)
		} else {
			// because we add in line breaks to put each photo on it's own line, we need to trim that out of the text
			let body = attributedText.attributedSubstringFromRange(range).string.stringByTrimmingCharactersInSet(.whitespaceAndNewlineCharacterSet())
			
			// text between images may just be whitespace, and can be ignored
			guard !body.isEmpty else { return } // block continue
			
			let message = Message(senderName: "You", body: body)
			messages.append(message)
		}
	}
	
	return messages
}
```

It's important to note here that we don't handle any other attributes here like bold or italics. We assume everything is either an image or text.

If you try everything out now, you should be able to paste in various types of images, type in text, and send a combination of the 2 to our highly advanced example chat stream. But wait, nothing is animating! That's no fun.

![No Fun](http://i.giphy.com/3orif8Uufh4J2lRKaQ.gif)

This is where [FLAnimatedImage](https://github.com/Flipboard/FLAnimatedImage) comes in. It provides 2 primary components. The first is `FLAnimatedImage`, which is similar to `UIImage`, but also includes animation information from GIFs. However, it does not support other file types, so we will need to conditionally use either `UIImage` or `FLAnimatedImage` depending on our file type. The second is `FLAnimatedImageView`, which is a subclass of `UIImageView`. Because it is a subclass, we can safely use it for both still and animated images. So we can just change our `photoView` class (in the case of our sample, `AspectImageView` just needs a new superclass). The only sticky point here is that you must set `animatedImage` to nil when setting a plain image, since the next frame will replace whatever you set if you are reusing the view.

For the animated vs. still image issue, we will use a Swift enum to represent the 2 options:

```swift
enum Photo {
	case image(UIImage)
	case animatedImage(FLAnimatedImage)
}
```

And create one or the other based on our attachment type:

```swift
if attachment.fileType == kUTTypeGIF as String {
	photo = .animatedImage(FLAnimatedImage(animatedGIFData: contents))
} else {
	photo = .image(UIImage(data: contents)!)
}
```

Now animated GIFs play in our chat stream!

![Excited for animated GIFs](http://i.giphy.com/xT1XGCLSBt98CqmXe0.gif)

This is clearly more fun now. I would say our app has increased at least 74% on the fun index.

There's one last thing we can do to take our GIF support to the next level. In most apps, you have to select the GIF in a keyboard and then manually paste it into the app. But what if we could detect when a user selected a GIF from a keyboard and *automatically* paste it in.

![What did you say?](http://i.giphy.com/5VKbvrjxpVJCM.gif)

But here's the thing, we don't want to auto paste just any old thing. Not even just any GIF. If the user opens the app with a GIF on their pasteboard, we can't know for sure that GIF is for us. So we want to detect when a GIF is pasted, but only from a keyboard.

`UIPasteboard` is a funny little class. It has a notification that it posts when something is copied, but only if it was copied from within the app. The assumption is that if something was copied from another app on iOS, you can just check for it when the application becomes active. That's less and less true these days in iOS though. In particular, when a keyboard copies something, it is not done in our app's process, and the user doesn't have to switch back to our app since it's already open. But we can use this to our advantage. If we didn't copy it, and it wasn't on the pasteboard when we become active, we can assume it came from a keyboard and that the user would want us to paste it for them.

The only way left to detect when something is added to the pasteboard is to create a repeating timer and check each second for changes. `UIPasteboard` has a `changeCount` property that we can use to quickly check if anything has changed before doing anything with those changes. The details of how to avoid pasted content and avoid a retain cycle can get a little complicated, so make sure to take a look at the sample code, but the timer handler looks like this:

```swift
@objc func checkPasteboardChanged() {
	guard UIPasteboard.generalPasteboard().changeCount > self.lastChangeCount else { return }
	self.lastChangeCount = UIPasteboard.generalPasteboard().changeCount
	
	guard UIPasteboard.generalPasteboard().containsPasteboardTypes([ kUTTypeGIF as String ]) else { return }
	
	// let our paste action handle inserting the new image
	textView?.paste(nil)
}
```

We just check if the pasteboard has changed, verify that it's a GIF, and if it is activate paste programatically. Now, when a user taps on an animated GIF in their keyboard, it will immediately appear in the text view without them needing to manually paste them!

![Oh yeah!](http://i.giphy.com/7LO7q5KcXawaQ.gif)

[^1]: I discussed the topic of this post for quit a while with an Apple engineer at WWDC this year and while he is only a sample set of 1, he was absolutely shocked that UIImage didn't support animated GIFs out of the box.

[^2]: Some will point out that `UIImage` does actually have support for animated images. However, out of the box it is mostly limited to their asset catalog format that just uses multiple PNGs for each frame. There are libraries that will use the low level APIs to convert a GIF to one of these animated `UIImage`s, but `UIImage` doesn't support variable timed frames. So if you accept GIFs as user input, many will appear choppy and broken.
