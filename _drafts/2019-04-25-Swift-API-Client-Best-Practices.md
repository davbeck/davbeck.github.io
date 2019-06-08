---
layout: post
title: Swift API Client Best Practices
date: 2019-04-25
tags: [iOS, Swift, API]
---

Almost every app these days connects to some kind of backend api over HTTP. Sometimes many different APIs.

Swift of course has access to Foundation with builtin support for making these requests using URLSession. In the past, many developers reached for libraries on top of Foundation like [AFNetworking](https://github.com/AFNetworking/AFNetworking) and [Alamofire](https://github.com/Alamofire/Alamofire) to build their API clients. Those are both great libraries, but you probably don't need either! URLSession is quit capable and usable all on it's own. Let's look at how you should approach connecting to APIs from your apps.

## Making a basic request

Let's use the Github api as an example. Here's how you might get issues for a given repo:

```swift
struct GithubIssue: Codable {
	// just a placeholder
}

let decoder = JSONDecoder()
let url = URL(string: "https://api.github.com/repos/davbeck/VectorIO/issues")!
let task = URLSession.shared.dataTask(with: url, completionHandler: { (data: Data?, response: URLResponse?, error: Error?) -> Void in
	guard let data = data, let issues = try? decoder.decode([GithubIssue].self, from: data) else { return }
	print("issues: \(issues)")
})
task.resume()
```

<span class="side-note">
I'm force unwrapping the URL here. Of course force unwrapping is usually a bad practice in Swift, but IMHO is acceptable for compile time constants. The only way this initializer will fail is if a new version of Foundation changes what it considers a valid URL to be. In most of my apps, I add `ExpressibleByStringLiteral` to `URL` like in [this tweet](https://twitter.com/johnsundell/status/886876157479616513?lang=en).
</span>

A wee bit verbose to be sure...

But more importantly, it doesn't handle a lot of important stuff. Like errors (those are important remember?) and invalid status codes. And when we look at another requests, say for pull requests, you'll notice that *a lot* of that code is exactly the same:

```swift
struct GithubPullRequest: Codable {
	// just a placeholder
}

let decoder = JSONDecoder()
let url = URL(string: "https://api.github.com/repos/davbeck/VectorIO/pulls")!
let task = URLSession.shared.dataTask(with: url, completionHandler: { (data: Data?, response: URLResponse?, error: Error?) -> Void in
	guard let data = data, let pulls = try? decoder.decode([GithubPullRequest].self, from: data) else { return }
	print("pulls: \(pulls)")
})
task.resume()
```

## Creating our own client

Of course, as programmers we know that when we see code repeated, we abstract it into something like a function or class.

Let's pause for a minute though and think philosophically about what exactly should be abstracted. The motivation behind libraries like Alamofire is that you can abstract this code generally for *all* API requests. And to some extend you can. However, APIs come in all shapes and sizes, but usually (unfortunately not always) they are consistent within themselves.

Take for example the concept of a version number. Lots of APIs include a way to specify what version you want to use, but in many different ways. Some prepend their domain like `v1.api.service.com`. Others include it in the path: `api.service.com/v1/`. In the case of Github, it's included as part of the `Accept` header. Authentication varries similarly. And at the very least, every api is going to almost certainly have a common base URL.

Our best approach is to create a custom API client class for each api that we interact with that takes all of the APIs assumptions into consideration. Users of the client then don't have to worry about the services conventions for each request.

Here's what that might look like for our first 2 requests:

```swift
class GithubClient {
	let decoder = JSONDecoder()
	let baseURL = URL(string: "https://api.github.com/")!
	
	private func send<ResponseBody: Decodable>(_ path: String, completion: @escaping (_ data: ResponseBody?, _ error: Error?) -> Void) {
		guard let url = URL(string: path, relativeTo: baseURL) else { return }
		var request = URLRequest(url: url)
		request.addValue("application/vnd.github.v3+json", forHTTPHeaderField: "Accept")
		
		let task = URLSession.shared.dataTask(with: url, completionHandler: { (data: Data?, response: URLResponse?, error: Error?) -> Void in
			guard let data = data, let responseBody = try? self.decoder.decode(ResponseBody.self, from: data) else { return }
			completion(responseBody, nil)
		})
		task.resume()
	}
	
	func getIssues(completion: @escaping (_ data: [GithubIssue]?, _ error: Error?) -> Void) {
		self.send("repos/davbeck/VectorIO/issues", completion: completion)
	}
	
	func getPullRequests(completion: @escaping (_ data: [GithubIssue]?, _ error: Error?) -> Void) {
		self.send("repos/davbeck/VectorIO/pulls", completion: completion)
	}
}

let client = GithubClient()
client.getIssues { (issues, error) in
	guard let issues = issues else { return }
	print("issues: \(issues)")
}
```

Notice that I'm adding an `Accept` header to every request. That wouldn't be correct for any other API, but is for every one of our Github requests. Also notice our use of `URL(string:relativeTo:)`. I think this is an underutilized tool. It intelligently creates a relative URL, a lot like the `href` attribute in HTML does. For instance, if you include a full URL, it will override every part of the base URL. If you provide an absolute path (one that starts with a "/") it will wipe out the path of the base URL and replace it with the new path. And in our case here, we are only including a relative path with will be appended onto the end of our URL.

I'm not including the response data (which includes things like headers) in our completion block. In the case of the Github API, it's not relavent, so we can simplify *our* client by not including it.

## Error handling

Remember that errors matter! In the current implimentation, any issue at any point is being ignored and the completion block is never called. Let's fix that.

First, let's define our won error:

```swift
class GithubClient {
	enum ClientError: Swift.Error {
		// error cases go here
	}
	
	// ...
}
```

We'll start by catching any `nil` urls:

```swift
	enum ClientError: Swift.Error {
		case invalidURL
	}
	
	// ...
	
	private func send<ResponseBody: Decodable>(_ path: String, completion: (_ data: ResponseBody?, _ error: Error?) -> Void) {
		guard let url = URL(string: path, relativeTo: baseURL) else { return completion(nil, ClientError.invalidURL) }
		// ...
	}
```

Here we add our first error case and then throw that error if for whatever reason URL fails to initialize.

Next let's handle errors in our completion handler:

```swift
// ...
if let error = error {
	completion(nil, error)
}
	
guard let data = data, let responseBody = try? self.decoder.decode(ResponseBody.self, from: data) else { return }
completion(responseBody, nil)
// ...
```

This is just passing on the error from the task. But this will only include an error if something really catastrophic happens, like the internet being down (which is of course, *very* catastrophic). But there are lots of other kinds of errors an API can return that URLSession will not create an error for. This is of course because all APIs are different and URLSession is conservative. It won't assume that a 404 status code means that something went wrong.

<span class="side-note">
I've worked with at least one API that uses 404 when requesting an array of items returns an empty set. Don't do this, but if you do, URLSession has your back.
</span>

Github is a sane API though, and anything outside of the 200 range of status codes can be considered an error:

```swift
enum ClientError: Swift.Error {
	// ... 
	case unexpected
	case invalidStatusCode(Int)
}

// ...

do {
	if let error = error {
		throw error
	}
	guard let response = response as? HTTPURLResponse else { throw ClientError.unexpected }
	guard (200..<300).contains(response.statusCode) else { throw ClientError.invalidStatusCode(response.statusCode) }
	
	guard let data = data, let responseBody = try? self.decoder.decode(ResponseBody.self, from: data) else { return }
	completion(responseBody, nil)
} catch {
	completion(nil, error)
}
```

We've added 2 new error cases. `.unexpected` is used for cases that probably shouldn't happen. We are making an HTTP request so we should expect to get a `HTTPURLResponse` back (again, URLSession is conservative and technically [supports `data`, `file`, and `ftp`](https://developer.apple.com/documentation/foundation/urlsession#2934752) URLs). But just in case, we handle that and hope the error gets bubbled up to a point we can find later.

`.invalidStatusCode` is more obvious. Anything outside fo a 2XX status code is considered an error. We aren't implimenting authentication for this example (although Github does support it for things like private repos and editing) you could conditionally catch `ClientError.invalidStatusCode(401)` errors and handle them in a special way.

The Github API usually provides some more details in the response body. Let's make sure to grab that if it's available:

```swift
enum ClientError: Swift.Error {
	// ...
	case invalidStatusCode(Int, APIError?)
}
	
struct APIError: Decodable {
	var message: String
	var documentationURL: String
	
	enum CodingKeys: String, CodingKey {
		case message
		case documentationURL = "documentation_url"
	}
}
	
// ...

guard (200..<300).contains(response.statusCode) else {
	throw ClientError.invalidStatusCode(
		response.statusCode,
		try? self.decoder.decode(APIError.self, from: data ?? Data())
	)
}
```

We don't want to throw a decoding error and lose our status code if that body is missing (or it's in a format we don't understand) so I'm using `try?` on the decoder.

Next we can throw any errors around parsing our response body:

```swift
let responseBody = try self.decoder.decode(ResponseBody.self, from: body ?? Data())
```

I'm not explicitly checking for a nil body here. If we have a valid status code, we *should* have a non-nil body. And if we don't the decoder will catch the issue and throw an error for us.

## Using Result

[Swift 5 introduced a standard Result type](https://github.com/apple/swift-evolution/blob/master/proposals/0235-add-result.md) (finally). It's a simple construct, but makes our completion blocks more clear:

```swift
class GithubClient {
	// ...
	
	private func send<ResponseBody: Decodable>(_ path: String, completion: @escaping (_ result: Result<ResponseBody, Error>) -> Void) {
		guard let url = URL(string: path, relativeTo: baseURL) else { return completion(.failure(ClientError.invalidURL)) }
		// ...
		
		let task = URLSession.shared.dataTask(with: url, completionHandler: { (data: Data?, response: URLResponse?, error: Error?) -> Void in
			do {
				// ...
				completion(.success(responseBody))
			} catch {
				completion(.failure(error))
			}
		})
		task.resume()
	}
	
	func getIssues(completion: @escaping (_ result: Result<[GithubIssue], Error>) -> Void) {
		self.send("repos/davbeck/VectorIO/issues", completion: completion)
	}
	
	func getPullRequests(completion: @escaping (_ result: Result<[GithubPullRequest], Error>) -> Void) {
		self.send("repos/davbeck/VectorIO/pulls", completion: completion)
	}
}

let client = GithubClient()
client.getIssues { (result) in
	guard let issues = try? result.get() else { return }
	print("issues: \(issues)")
}
```

I'm not a fan of the explicit type error (I've just used Swift.Error here) but otherwise `Result` adds clarity to the interface of our class. You can see the difference if you compare it to how URLSession presents it's completion handler. It provides 3 arguments, all of them optional, but it's not clear in what combination those could be provided.

## Using Promises

Alternatively (really on top of) we could use Promises to return our data. Promises take the idea of a result type one step further. Instead of providing a completion block, it returns an object that eventually is either sucessful or a failure. This works well with the typesystem: you can only ever return a promise once, and it can only ever be completed once. A completion block can be called multiple times by accident, or as we saw in our early examples, not at all.

But where promises really shine is in composition and error handling. If you want to tag on an additional action to a promise, you use something like `map` or `then`. In these cases you don't have to worry about the error case. If the previous promise is successful, it calls your chained method and uses the result. If the previous promise has an error, it skips your chain and continues on with the error. Then at the very end you can hand any error that occurs in the entire chain. Consider this contrived callback pyramid of doom:

```swift
foo() { (data1, error) in
	if let error = error {
		completion(nil, error)
		return
	}
	
	// assuming data1 is not nil if error is nil
	bar(data1!) { (data2, error) in
		if let error = error {
			completion(nil, error)
			return
		}
		
		car(data2!) { (data3, error) in
			if let error = error {
				completion(nil, error)
				return
			}
			
			// use data3
		}
	}
}
```

At each step of the way, we have to check if there was an error and bail. And it's pretty easy to mess this up. In writting this example I almost forgot to return after calling the completion handler. Here's a similarly contrived example using Promises:

```swift
foo()
.then { data1 in
	return bar(data2)
}
.then { data2 in
	return car(data2)
}
.done { data3 in
	// use data3
}
.catch { error in
	// handle any error that occurs at any point in the chain
}
```

Here's what our client might look like using [PromiseKit](https://github.com/mxcl/PromiseKit):

```swift
class GithubClient {
	// ...
	
	private func send<ResponseBody: Decodable>(_ path: String) -> Promise<ResponseBody> {
		guard let url = URL(string: path, relativeTo: baseURL) else { return Promise(error: ClientError.invalidURL) }
		// ...
		
		return URLSession.shared.dataTask(.promise, with: url)
			.validate()
			.map {
				try self.decoder.decode(ResponseBody.self, from: $0.data)
			}
	}
	
	func getIssues() -> Promise<[GithubIssue]> {
		return self.send("repos/davbeck/VectorIO/issues")
	}
	
	func getPullRequests() -> Promise<[GithubPullRequest]> {
		return self.send("repos/davbeck/VectorIO/pulls")
	}
}

let client = GithubClient()
client.getIssues()
	.done { issues in
		print("issues: \(issues)")
	}
```

PromiseKit includes extensions for URLSession that return promises instead of taking completion blocks. It even has a convenience method `validate` that checks the status code and throws an error for us so we don't have to handle that.

## Next steps

At some point we would need to add the ability to customize the HTTP method (`GET`, `POST`, `PUT` etc) used for each requests. Depending on the API you are connecting to, there will probably be others like query params, request body (hopefully in JSON, but maybe in multipart/form encoding), and headers.

That's the nice thing about writting our own client instead of using a generalized approach. Each API is different, and has different needs. We are able to use a pretty basic implimentation here because the Github API is straightforward, but also we are only using a subset of the API that doesn't require much input. When you roll your own API client, you get to add what you need and nothing more.

Could you generalize this architechure out a bit? Sure, there are some common patterns that any client would use. But a lot of that comes down to preference. Do you like Promises? How detailed do want your erros to be? Personally I use a code snippet as a starting point rather than create a library.