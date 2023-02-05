---
layout: post
title: "Trunk Dev Notes #1"
date: 2023-01-14
tags: [SwiftUI,Trunk,ThinkSocial,iOS]
---

The last few weeks working on Trunk have been fairly frustrating, but I think it's coming around the corner.

I had hoped to spend the holiday break knocking out a punch list of features before releasing a second beta to a wider audience. However as more features got added, 2 very big issues emerged that brought development to a stop.

First was a performance issue with my caching layer. I had gone with an approach that I had used before where caching was based on api endpoints. Views would first show the cached result and then if needed, refresh the endpoint to get the latest data. While that simple approach worked great for apps where the data is mostly isolated between each call, in Mastodon (and any social app) the same data can be returned by multiple api requests.

For instance, you might have a status that was loaded from the timeline. If the user favorites that, you want to update the cache to remember that change. But that same status may then come back as a reply to another status, or as a boosted post, etc. I was trying to update every cached value when something like that changed, but not only was it incredibly buggy, but it was also slowing the app down to a complete crawl.

After several false starts and alternatives considered, I eventually went back to CoreData. I’ve been trying to get away from using CoreData for the last several years (mostly because of its limitations with Swift, but also its tendency to crash the app), but this experience really showed me how much it really has to offer.

So, with the entire backend rewritten, I then turned my attention to the other showstopper: scrolling.

Trunks UI is centered around a horizontal paged interface where the user swipes between posts from their timeline. As they scroll to each post, the “context” (the conversation before and after the post) get loaded in. But it would be really weird if you went to view a post from your timeline and what you saw at the top of the screen was actually the original post that someone replied to. So instead I show the “primary” post (the one from the timeline) and you can scroll up to see older posts in the conversation.

The problem is that adding content to the top of a scroll view while maintaining the previous content’s position is actually really tricky. UIScrollView (and by extension SwiftUI.ScrollView) really wants to maintain the offset. That gets even more tricky to deal with when using views that don’t render their entire contents, like UITableView or LazyVStack. In those cases the position might not even be consistent, because visible content is placed based on estimated heights.

In the first beta I had some pretty simple logic that would scroll to the primary posts position when the context loaded. This worked at first, but as I implemented all the various content posts can have (like media and polls), the screen started flashing as the new content loaded in and then was scrolled out of the way.

I tried several approaches to get this to work. [A recent episode of Swift Talk](https://talk.objc.io/episodes/S01E336-scroll-view-with-tabs-part-2) made me hopeful that I could implement something in SwiftUI that would work. But there just isn’t a direct enough connection to the layout process to get it to be reliable and performant.

Eventually I gave up and realized I would need to rewrite the post page in UIKit. While that was a hard decision to make, ultimately I had to ask myself if I wanted to make an app in pure SwiftUI, or the best app possible. I attempted to use the existing post view (using either UIHostingConfiguration or UIHostingController) but animating size changes with either was non functional. I was able to use a lot of SwiftUI views within each post row, but the overall container needed to be a UIView.

I concidered using UITableView or UICollectionView, but if something's worth doing, it's worth overdoing. Both of those can have their own challenges with maintaining content's position. I think that's why [Ivory is using manual layout with pre-computed heights](https://tapbots.social/@paul/109564775494812308) and why [Mastoot needs to scroll to content twice](https://mastodon.social/@libei/109597383746473808). So instead I wrote my own UIScrollView subclass that handles reusing views itself. What this gives me is autolayout based views without any inconsistent positioning. So far it seems to be working really well.

With those changes, the next beta is getting really close. There are a few bugs to work out and one last feature that I want to include. The goal is to have a “feature complete” timeline. Meaning you won’t miss anything if you browse mastodon exclusively in the app.