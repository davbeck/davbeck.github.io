---
layout: post
title: Updating ThinkScore after 4 Years
date: 2019-06-08
tags: [Swift,iOS,AppleWatch,Xcode]
---

Back in 2015, I created an app called ThinkScore. 
It helps you keep score for games, 
like golf or card games. 
Apple Watch was brand new at the time,
and my primary motivation for building this app
was to explore WatchKit.

Since then, I haven't touched the app at all.
I actually still use the iOS app, and it's suprisingly useful still.
Without any updates, an app built for iOS 9 still works really well (as long as you don't have a notch 😅).
I know you're suppose to be discusted by any code you wrote more than 6 months ago, but I'm actually really proud of how well the app works, and how clean the code is. Granted it's a very small project without any other contributers or major changes over time.

The watch app however, hasn't faired so well. [Marco Arment has joked](https://twitter.com/marcoarment/status/1138534770600751110) that he has re-written his watch app every Summer. The original SDK for the watch was... not good. All of the UI on the watch, including responding to UI changes, was controlled from the phone. The user taps a button on the watch, that gets sent to the phone over bluetooth, then the phone responds with any UI changes. If I'm being honest, the ThinkScore watch app, like most of those original watch apps, is unusable.

Almost immediately Apple updated the SDK to support running directly on the watch, but still tethered to the phone for a lot of vital functionality. I never updated ThinkScore to use that model. This year, with watchOS 5, apps can run completely independently on the watch, and there's even a native UI framework to go with it: SwiftUI.

The iOS app also included iAds and a "pro" version to remove those ads. iAd is long gone, and my app hasn't been serving up ads for some time. It never really had a good fill rate anyway. But regardless of the ad platform, I don't think I liked this model. I hate ads, and dealing with StoreKit is a pain. Instead, I plan to charge for the app upfront and eliminate this model altogether.

This seems like a good time to dust off the app and get it up and running with all the latest cool stuff.

---

Before I could make any changes to the app though, I had to get it building in the latest version of Xcode. It's funny how much things can change in 4 years...

Xcode won't build original watch apps anymore. No one should have used that SDK to begin with, and it seems Apple admitted that and banned it entirely 😅. That's ok, I wanted to start from scratch anyway. DELETE.

Next issue was the apps Swift version. It was written in Swift 2 (or maybe 1?). Xcode doesn't support migrating from Swift 2 anymore. I have a copy of Xcode 8, but that crashes on launch on Mojave. That migrator was never very good anyway, so instead I just set the Swift version to 5 and hit build. The process of fixing all the various build errors from doing this is **tedious**. Even though ThinkScore is a very small app, it had hundreds of errors to work through, and as you fix them, others get generated.

To avoid converting code I wouldn't be using, I deleted the iAd code as well as the pro version check. I didn't use very many dependenies with the app, but I decided to remove them all as well. I'm no longer comfortable using Crashlytics because Google owns them and uses them to install trackers in apps. I also used a library I wrote called [TULayoutAdditions](https://github.com/davbeck/TULayoutAdditions), which I don't use anymore in favor of layout anchors.

From there it's a matter of addressing one build error after another, mostly using Xcode's fixits to rename API calls and making sure it doesn't do anything stupid. I really need an intern for that.

At that point, it buids successfully, and works pretty much exactly like it did before. Progress?

---

I paused here to enable Mac support too. Why not 🤷🏽‍♂️.

// TODO: Insert photo

Why would someone want to use this app on the Mac? I don't know, and I probably will never ship this, but it's interesting.

---

Before I re-add a watch app, I needed a way to sync between them. The original watch SDK ran entirely on the phone, so they could just share a file between them. WatchOS supports communicating between the watch and the phone, but in order to be completely independent, I wanted a sync method that worked over the internet. iCloud was the obvious choice.