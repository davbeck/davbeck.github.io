# What will Swift Package Manager for apps look like?

The [Swift Package Manager](https://swift.org/package-manager/) (or SPM) was an absalute exciting surprise for Swift developers when it was released. While I apreciate all the work that the maintainers of [Cocoapods](https://cocoapods.org) and [Carthage](https://github.com/Carthage/Carthage) put into making dependencies work at all for iOS and other Apple platforms, I don't think anyone really likes using them. They have a difficult in front of them. They have to integrate with an always changing, and often undocumented, Xcode project structure. Every year, something in Xcode changes that breaks some aspect of those tools. I don't think anyone expected Apple to open source an official package manager, but it certainly gave us all hope that a better solution was on the horizon.

However, SPM has been quit limited and hasn't gotten the same attention that the rest of Swift has so far. The biggest limitation is the fact that it can only be used with command line targets. That's fine for server applications, but completely useless for iOS, tvOS, watchOS, and any Cocoa macOS application.

Initially Apple promised that SPM would eventually come to "Xcode" and app targets, but like [open source Facetime](https://9to5mac.com/2018/06/06/make-facetime-an-open-standard/) and [AirPower](https://www.imore.com/so-where-airpower), we have yet to see it actually released. Unlike those products though, we do seem to get hints each year that it will come eventually, but that it was deprioritized each year.

But what would it even look like to be able to use SPM with Xcode and apps? To be fair, Xcode integration already exists... kind of. SPM can generate an Xcode project from the command line. That's a really convenient feature, but Xcode still doesn't know anything about depencies and SPM. When `swift package generate-xcodeproj` is run, all dependencies are downloaded and arranged just so for Xcode to link and compile everything correctly. If you add a dependency or change a version, you have to regenerate the entire project. And regardless, you still can't use it to build apps.

That model just doesn't make sense for an iOS app. The Xcode project has a ton of information about how exactly to build the app that can't (and probably never will) be representable in the `Package.swift` file. And iOS app is just more complicated than a command line tool.

[Here's what Apple had to say on the subject back in 2016](https://lists.swift.org/pipermail/swift-build-dev/Week-of-Mon-20160815/000613.html):

> We plan for you to be able to write iOS Swift Packages, but rather than creating a complete iOS build system in SwiftPM, this will likely work by leveraging Xcode's build system.