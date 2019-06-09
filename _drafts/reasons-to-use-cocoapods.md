Xcode 11 introduced Swift Package Manager support (#finally) allowing us to use it with our iOS apps instead of Cocoapods. It works really well and feels well designed, but there are still some really good reasons why you'll still be using Cocoapods, at least for now.

## You want to use a framework with file resources

If you have a framework with an image or other kind of non-code resource, there's no way to represent that in SwiftPM. It's unclear how this would work with the typical SwiftPM executable, which only builds a binary executable and not an app package. If SwiftPM does add support for this, it will probably only be supported by Xcode, which would be a bit weird.

## You want to use a binary library

Most of the time you are using dependency managers to import open source projects that you build locally. But for closed source dependencies like Crashlytics, you're only option is a pre-build binary. Cocoapods handles this gracefully.

You probably can just manually link a library in this case,
since most of the work is already done for you.
But it might also be nice to have all your dependencies in one spot.

## You want to use vended scripts

Cocoapods has a really convenient feature where it will vend scripts from your dependencies. This is invaluable for things like Crashlytics which requires a build script. It's also really convenient for things like SwiftFormat. In that case, you aren't really importing code to use, but a tool. It may seem odd to track tools with your dependency manager, but if you're working with a team, it's important to keep the version of those tools in sync as well.
