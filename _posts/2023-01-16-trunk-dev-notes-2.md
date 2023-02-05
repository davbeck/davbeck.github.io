---
layout: post
title: "Trunk Dev Notes #2"
date: 2023-01-17
tags: [Trunk,ThinkSocial,iOS]
---

Over the weekend I was able to get the second beta of Trunk to a place where I was ready to release it. I wanted to get it to a point where the concept of the app was well represented, which meant a fully functional timeline.

I’ve been trying very hard to put off improvements to media in order to focus on the second beta release. I wanted that one to include a “feature complete” timeline experience, but I had bigger changes in mind for media that I didn’t want to get in the way.

But since switching to a UIKit based post page, video was completely broken. Videos just didn’t show up at all. So I needed to at least get it working, even if it wasn’t the best experience for now. But I also didn’t want to put a bunch of work into something that I knew would change quit a bit.

My first attempt at this was to replace VideoPlayer with AVPlayerViewController wrapped in UIViewControllerRepresentable (currently attachments are SwiftUI views embedded inside the UIKit views using UIHostingConfiguration). And… it had the exact same bug. I don’t know why this surprised me, that’s probably exactly how VideoPlayer itself is implemented. Seems like there is a bug with UIViewControllerRepresentable in general and UIHostingConfiguration.

The solution I landed on was to use AVPlayerLayer mostly the same way I do GIFv files (auto played silently) and present AVPlayerViewController when the user taps on it. It has some rough edges but it works.

With that fixed, and some improvements to the all caught up screen, I was ready to open up the beta to a few more people. When I released the first beta I limited it to 100 testers. I wanted to get feedback on the horizontal timeline, but I didn't want to release it to too many people when it was barely usable. Shortly after I released it, [James Thomson](https://mastodon.social/@jamesthomson) boosted the link and those spots filled up almost immediately, but I wasn't sure what the demand would look like when I opened more spots.

After uploading the second beta, I increase the number of testers to 500 and posted the link again. To my surprise those all filled up overnight. I have never seen this much interest in a beta in my entire career. I'm sure most of that is due to the massive migration to Mastodon and everyone looking for new clients, but it's still very encouraging. I don't know how well that will translate into paid subscriptions, but it's very promising.

Something else that was really promising: the #1 feature request I received following this beta was literally the top item on my backlog. I had originally hoped to include an option (defaulted to on) to hide replies that you have already seen, but it proved to be more difficult than I thought and I really wanted to get something out, so I punted it for the next release. I spent Monday getting my predicates in order to make this work. This feature actually makes me a little nervous. It’s pretty easy to hide too much.

Along with hiding previously viewed content, I also addressed a couple of bugs that popped up in the last beta release. On in particular was rather odd: casting to an Int32 was overflowing for someone’s follower count. Certainly there isn’t a user with billions of followers, but the crash logs don’t include any info on what the value actually was, so who knows what was going on there.

Finally before dropping the next beta, I wanted to add in some notices about missing functionality. Several people asked about how to reply to posts, or how to view a profile. To manage expectations I added some placeholder views where those feature will eventually live.