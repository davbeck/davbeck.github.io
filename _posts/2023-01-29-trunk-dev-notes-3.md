---
layout: post
title: "Trunk Dev Notes #3: CI Servers, Cleanup, and Fullscreen Images"
date: 2023-01-29
tags: [Trunk,iOS]
---

At the beginning of last week I spent some time working on CI for Trunk. [After making big claims about having to have CI on every project I work on](https://mastodon.social/@davbeck/109530813220759111), my workflow has been broken for several weeks. It’s admittedly hard to take time away from actually building something to setup support tools.

I was using Xcode cloud, but tests started failing after I added [snapshot tests](https://github.com/pointfreeco/swift-snapshot-testing). Apparently the way Xcode cloud runs tests the snapshot images aren't available.

So I went back to what I know. Going back to what I know seems to be a developing theme of this project. At my last company I had really good success with self hosted GitHub action runners. And I already have a “junk drawer” M1 Mac mini that could do the job well. It will probably help curb costs as well since I’m leaning pretty heavily into UI tests for this project and those eat up CI minutes quickly.

I’m leaving my TestFlight builds on Xcode Cloud though. That is pretty tricky to setup anywhere else because of code signing. To my knowledge, Xcode Cloud is the only CI that can use automatic provisioning.

I tested the full workflow out by putting out a fix for my Sentry logging. Apparently I had forgotten to paste in my api key. Oops.

---

I then spent the evenings last week going through several small improvements.

I learned a new term recently, [“don’t pet your paintings”](https://theoatmeal.com/comics/creativity_petting) and it’s really stuck with me when it comes to this kind of thing. I want to focus on making changes that are objective improvements, not just going around in circles. There’s a time to waffle on the exact font size of a label, but now isn’t it. So right now I only focus on things that stand out to me on a daily basis actually using the app.

But there were a handful of things that were pretty obvious. For instance on link previews, I include the description. For whatever reason I put the description above the title though. And every time I saw that it looked wrong. Those kind of things make sense to fix.

But anything that I'm not sure about, or think I might change back, goes straight to the icebox.

---

At the start of this week I finally tackled something I've been putting off: fullscreen images. I've always been confused by the UIViewController transitions api. But when I think about what the core experience of the app is going to be, I consider excellent media handling to be a top priority.

My approach (and I'm not sure in hindesight this was the right approach) was to start with [NYTPhotoViewer](https://github.com/nytimes/NYTPhotoViewer). I've used it before and it worked well, but I wanted to make some significant changes to the design and I needed it to be able to support Mastodon's GIFs (which are actually videos).

The first step to get it to work was to rewrite it in Swift. I used ObjC for almost a decade, but my brain doesn't think that way anymore. From there I refactored it to use custom views instead of images, and finally made the UI changes I wanted, which mostly involved using a white background in light mode and a navigation bar to match what iOS usually does.

The hardest challenge with this whole process was supporting GIFs. The animation is suppose to imply a thumbnail zooming into place fullscreen. In reality there is more than 1 view involved in that transition. For NYTPhotoViewer, it actually uses 4 views during the animation: the thumbnail, the fullscreen view, and a snapshot of each for the animation transition. But animating snapshots of views won't work for anything with movement. The view would appear to freeze while it animated into place. And for GIFs, you need to make sure their progress is kept in sync between the thumbnail and the fullscreen version.

This was made even more challenging because of limitations with [`AVPlayerLayer`](https://developer.apple.com/documentation/avfoundation/avplayerlayer). It doesn't animate changes to it's transform property like a normal layer does. When you add it to the view hierarchy, even if you are sharing an AVPlayer, even if you wait for [`isReadyForDisplay`](https://developer.apple.com/documentation/avfoundation/avplayerlayer/1389748-isreadyfordisplay) to be `true`, it will stil be blank for a frame or 2.

This process had all the stages of programming. The overconfidence that it would be easier that in was. The despair that it was impossible. The despair that I would never get it to work right and this would be the end of the project. The resolve to leave it alone for a while but then continually "trying one more thing" to get it to work. The overwhelming joy when part of it finally does work, but 30 seconds later you move on to another component only to start the process all over again.

I was driven through this entire process by Ivory. I looked at every Mastodon app I know of and only Ivory handled this UI transition correctly. Many don't even support fullscreen GIFs correctly, instead presenting them as videos. But that dang Ivory app did it perfectly, so it must be possible. I don't know if [Paul](https://tapbots.social/@paul) struggled with this as much as I did, or if he just threw it together effortlessly like the master he is, but if it was possible to get right, I was going to get it right.

I can't emphasize enough just how happy I am with the end result. [Take a look (in slow motion)](/images/2023-01-29-trunk-dev-notes-3/fullscreen-transition-slowmow.mp4). Notice how the corner radius animates, rather than just fading away. Notice how the image doesn't just jump behind the navigation bar but fades in. And [animated GIFs look great too](/images/2023-01-29-trunk-dev-notes-3/fullscreen-transition-gif.mp4). It continues to play while animating into place.