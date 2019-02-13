---
layout: post
title: Why you should break up your api endpoints
date: 2018-08-10
tags: [API]
redirect_from:
  - /blog/2018/08/10/api-breakup.html.html
---

Years ago I read [Even Faster Web Sites: Performance Best Practices for Web Developers by Steve Souders](https://www.amazon.com/Even-Faster-Web-Sites-Performance/dp/0596522304). It was a simpler time, and the best practices for web development were still being figured out. Almost 10 years later, the advice here seems obvious. Our tools take these practices for granted and impliment them by default. For instance, the idea of bundling all of your JS and CSS into single files for the entire site. It might not be obvious, but if one page uses JS file 1 and 2, and another page uses JS file 2 and 3, the best choice in most cases is to combine all the JS files into a single file and request it on ever single page. That's how Rails and Webpack, along with many other tools work out of the box.

The book had many great gems of wisdom, but one of the driving assumptions of the book was that http requests have a lot of overhead. HTTP connections are limited to 1 request and response at a time. So if you request 3 files from your server, that connection will ask for the first, wait for it to come back, ask for the second, wait for it to come back, then finally ask for the third and wait for that to come back. To combat this limitations browsers will create multiple connections to the same server, but those connections themselves have overhead and browsers will limit the number they create, usually 6. Additonally, each request has to send a complete set of headers, which can be quit large.

But that is HTTP/1.1. We now live in an [HTTP/2 world](https://caniuse.com/#feat=http2). A single HTTP/2 connection can handle nearly unlimted requests at a time. Additionally, it compresses headers into a much smaller request and response. For this reason we need to rethink our best practices.

Let's consider a typical app (single page web, or native) which doesn't think in terms of pages, but instead in terms of data through api requests. In a perfect [RESTful](https://en.wikipedia.org/wiki/Representational_state_transfer) world, each api endpoint represents a single object entity. But in the real world, there is no clean apis. Every api makes tradeoffs to include everything a client would need to know for a given request. Consider a user profile with an address:

`GET /users/123`

```json
{
	"name": "John Doe",
	"address_id": 123
}
```

How cumbersome, both in terms of performance and development time would it be to have to turn around and ask for the address of the user every time after you requested a user. So most apis end up looking more like this:

`GET /users/123`

```json
{
	"name": "John Doe",
	"address": {
		"street": "123 Main St."
		"city": "Anywhere",
		"state": "IDK",
		"zip": "12345"
	}
}
```

Which is fine and correct. But this can go too far when an API gets coupled too closely to the client's view:

`GET /users/123`

```json
{
	"name": "John Doe",
	"address": {
		"street": "123 Main St."
		"city": "Anytown",
		"state": "IDK",
		"zip": "12345"
	},
	"groups": [
		{
			"name": "Anytown woodworkers",
			"member_count": 145,
			"last_meeting": {
				"date": "2018-08-10",
				"address": {
					"street": "123 Main St."
					"city": "Anytown",
					"state": "IDK",
					"zip": "12345"
				}
			}
		}
	],
	"twitter_name": "@jdoe",
	"last_tweet": {
		"body": "I cut it twice and it’s still too short."
	}
}
```

It's not uncommon to see this kind of jumbled up data in an api, especially an internal one. It comes about by a design that needs to pull data from several different places to get everything together. But what if we broke this up into separate api endpoints? Something like this instead:

`GET /users/123/info`

```json
{
	"name": "John Doe",
	"address": {
		"street": "123 Main St."
		"city": "Anytown",
		"state": "IDK",
		"zip": "12345"
	}
}
```

`GET /users/123/groups`

```json
{
	"groups": [
		{
			"name": "Anytown woodworkers",
			"member_count": 145,
			"last_meeting": {
				"date": "2018-08-10",
				"address": {
					"street": "123 Main St."
					"city": "Anytown",
					"state": "IDK",
					"zip": "12345"
				}
			}
		}
	]
}
```

`GET /users/123/twitter_info`

```json
{
	"twitter_name": "@jdoe",
	"last_tweet": {
		"body": "I cut it twice and it’s still too short."
	}
}
```

Because each of these take the user id, we can request each of them immediately in parallel. While something like the address can be joined in a single SQL statement when querying for users, groups would need to be selected separately even in the first example, so there aren't any extra trips to the database with this approach, and the tweet info would likely be coming from an external api anyway, so all 3 of these would need to be independent regardless.

## Advantages

While some backend languages like Node and Go support doing this work in parallel, many langages and tools like Rails don't. And even then, there is some work to be done to make sure that it works in parallel. For instance, this is what it might look like in Node to make sure none of these requests block each other:

```js
async function getUser(id) {
	let infoPromise = getInfo(id);
	let groupsPromise = getGroups(id);
	let twitterPromise = getTwitterInfo(id);
	
	let [info, groups, twitter] = await Promise.all([infoPromise, groupsPromise, twitterPromise]);
	
	return {
		...info,
		...groups,
		...twitter,
	}
}
```

Not overly complicated, but just enough that in practice, it wouldn't be done unless there was an obvious performance problem. But this eats away at performance slowly. Because of the nature of UI development, it is usally simpler for front end clients to impliment this concurrency.

In contrast, when you send multiple http requests in parallel, regardless of what backend service you are running, it will always operate concurrently. A rails backend will use multiple processes or threads as needed, Node will use it's async IO to handle each request, Go will use it's full concurrency to process requests, and all of them will scale out to other servers if they are behind a load balancer.

This also makes the api simpler and easier to understand. Instead of having endpoints that are buckets of random information, you can see what data is being retrieved by it's endpoint. Maybe not pure REST, but something very close to it. This, in turn, makes the api more flexible. When the design is updated to exclude twitter from a profile page, the client can just stop requesting it. In the first, combinded approach, that info would likely need to remain in the api so that apps that haven't been updated yet, or user's that haven't updated yet, will continue to work.

Clients also have the flexibility of progressively loading views as data becomes available. We already do this regularly with profile photos. It is almost unheard of to have a profile screen that doesn't show you information about a person until their profile pic has loaded. Instead we show what we have and display the image when it's available. We can do the same thing with our api responses, especially if the information at the top of the screen loads first (and it often does because the data is simpler). In this case, the data that is still be loaded is likely "below the fold" and not visible until the user scrolls anyway.

Finally, not all data is equal. In a modern client side app, you can (and should) cache api responses for quicker viewing, even if it's just in memory. When a user lands on a page, if there is data already in the cache, you can show that again, and either asynchronously request a more up to date version or just avoid another request entirely if it hasn't been very long since the cached version was loaded. But, some data isn't updated as frequently as other data. Or it might be shared between pages. General site wide information that is needed to render a profile page only needs to be loaded the first time you visit a profile. Then on other profile pages you can just re-use what you already have cached. Or, you might have some data that changes frequently and you want to load that every time the user views a profile, but things like name and address are likely to remain the same for a while.

## When to avoid separate endpoints

Of course, it's not always a great idea to break *everything* up.

If a request is dependent on another request, it's best to combine them. For instance, if you really needed to know the last meeting for each of a users groups, if it wasn't included in the list of groups you would need to first request the list of groups and then independently look up each groups last meeting *after* that requests loads.

If it can be combined into a single SQL satement (or equivalent), you're probably better off just including it. Most db servers are still using a serial protocol just like HTTP/1.1 was, but more importantly can optimized combined queries better than they can individual ones.