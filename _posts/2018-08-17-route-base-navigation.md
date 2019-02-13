---
layout: post
title: Using URLs, paths and routes for navigation in an iOS app
date: 2018-08-17
tags: [iOS]
redirect_from:
  - /blog/2018/08/17/route-base-navigation.html.html
---

Many of our apps are mirrors, at least at some level, of a website. The interface might use iOS or Android specific UI or layout that is optimized for smaller screens, but the data the user is interacting with is the same accross platforms. Navigation for these 2 platforms however are fairly different. On the web you rely on the browsers back button heavily. In iOS you might use a navigation controller or a cancel button on a modal, but you have to be intentional about how things are presented and intentional about rolling back those transitions. For instance, if you just kept pushing modal view controllers on top of each other to go back to "home", you would end up with a large stack of views and view controllers all taking up memory indefinitely.

These worlds have to interact though. There has always been ways (sometimes hacks) to move between a web page and the same content in an app. First there were [custom url schemes](https://developer.apple.com/documentation/uikit/core_app/allowing_apps_and_websites_to_link_to_your_content/defining_a_custom_url_scheme_for_your_app) (`myapp://`) then [smart app banners](https://developer.apple.com/library/archive/documentation/AppleApplications/Reference/SafariWebContent/PromotingAppswithAppBanners/PromotingAppswithAppBanners.html) and finally Apple gave us [universal links](https://developer.apple.com/ios/universal-links/). That last one is by far the closest connection between a website and an app. The recommendation from Apple is to have things like emails include links to content on your website, and register your app to handle them if it's installed. But how your app handles a url and figures out what to show and how to show it, is up to us to figure out.

My approach to this is to create a "router" similar to what you would have in a web app. The job of the router is to map incoming urls (mostly just their path) into a view controller and a transition. In this way, a route is a lot like a Storyboard segue. Segues define the destination view controller and how it gets presented (navigation push, modal, or something custom). One big difference is that Segues always have a source, but our router needs to be able to handle urls opened when any view controller is already visible. I also like to use a router for in app navigation just to keep things consistent, but you certainly could continue to use more traditional navigation methods and just use routing for handling urls.

You can check out the [sample code on Github](https://github.com/davbeck/RoutesExample) to see the finished example.

## The Router

Let's start by building a generic router class. This class will just lookup an object for a given url and should be very reusable. It will use a generic type to represent the destination:

```swift
class Router<Destination> {
	init() {
	}
}
```

Let's write a test to verify some basic behavior:

```swift
class RouterTests: XCTestCase {
	func testReturnsMatchedParameters() {
		let router = Router<String>()
		router.register("/posts/:postID", destination: "Post detail")
    }
}
```

Each route will be represented by a `Route` type that has some path components and it's destination object.

```swift
	struct Route {
		enum Component {
			case constant(String)
			case parameter(String)
		}
		
		var components: Array<Component>
		var destination: Destination
	}
```

To make things easier and cleaner, we'll define routes using a pattern like "/posts/:postID". We'll break components on backslashes ("/") and any component starting with ":" will be treated as a parameter.

```swift
	struct Route {
		// ...
		
		init(pattern: String, destination: Destination) {
			self.components = pattern
				.components(separatedBy: "/")
				.map({ part -> Component in
					if part.first == ":" {
						return .parameter(String(part.dropFirst()))
					} else {
						return .constant(part)
					}
				})
			self.destination = destination
		}
	}
	
	var routes: Array<Route> = []
	
	func register(_ pattern: String, destination: Destination) {
		routes.append(Route(pattern: pattern, destination: destination))
	}
```

Go ahead and run your tests at this point just to make sure everything compiles and runs without exceptions.

Next, we'll add some matching logic to get a destination for a given url, starting with expanding our test:

```swift
	func testReturnsMatchedParameters() {
		let router = Router<String>()
		router.register("/posts/:postID", destination: "Post detail")
		
		guard let match = router.match(for: URL(string: "myapp:///posts/123")!)  else { XCTFail(); return }
		
		XCTAssertEqual(match.destination, "Post detail")
		XCTAssertEqual(match.parameters["postID"], "123")
    }
```

We'll impliment this by checking each route if they match. Note that this has some implications on how routes are handled, in particular the first route registered wins if 2 routes might match a url (ie /posts/:postID and /posts/123).

```swift
	struct Match {
		var destination: Destination
		var parameters: Dictionary<String, String>
	}
	
	struct Route {
		// ...
		
		func matches(_ url: URL) -> Match? {
			guard components.count == url.pathComponents.count else { return nil }
			
			var parameters: Dictionary<String, String> = [:]
			for (component, input) in zip(components, url.pathComponents) {
				switch component {
				case .constant(let value):
					guard input == value else { return nil }
				case .parameter(let name):
					parameters[name] = input
				}
			}
			
			return Match(destination: self.destination, parameters: parameters)
		}
	}
	
	func match(for url: URL) -> Match? {
		// url.pathComponents produces a slightly different result that doesn't match our pattern
		let pathComponents = url.path.components(separatedBy: "/")
		for route in routes {
			if let match = route.matches(pathComponents) {
				return match
			}
		}
		
		return nil
	}
```

You should be able to run tests and verify that the basic router logic works. We can add a few more tests as well. These should pass right away, but it makes sure we don't break anything in the future, especially if we want to make performance improvements.

```swift
class RouterTests: XCTestCase {
	var router: Router<String>!
	
	override func setUp() {
		super.setUp()
		
		router = Router<String>()
		router.register("/", destination: "Root")
		router.register("/posts", destination: "Post index")
		router.register("/posts/abc", destination: "Post detail constant")
		router.register("/posts/:postID", destination: "Post detail")
		router.register("/posts/:postID", destination: "Post detail alternative")
	}
	
	override func tearDown() {
		router = nil
		
		super.tearDown()
	}
	
	
	// Tests
	
	func testReturnsMatchedParameters() {
		guard let match = router.match(for: URL(string: "myapp:///posts/123")!) else { XCTFail(); return }
		
		XCTAssertEqual(match.destination, "Post detail")
		XCTAssertEqual(match.parameters["postID"], "123")
    }
	
	func testRootRoute() {
		guard let match = router.match(for: URL(string: "myapp:///")!) else { XCTFail(); return }
		
		XCTAssertEqual(match.destination, "Root")
		XCTAssertEqual(match.parameters, [:])
	}
	
	func testUsesFirstRoute() {
		guard let match = router.match(for: URL(string: "myapp:///posts/abc")!) else { XCTFail(); return }
		
		XCTAssertEqual(match.destination, "Post detail constant")
		XCTAssertEqual(match.parameters, [:])
	}
}
```

## Navigation Router

Next, let's turn our attention towards the actual navigation part of this. I'm breaking these up because how you handle the navigation part of routing will likely be different between apps, but the basic routing logic above can be reused. For instnace, do you want to use Storyboards, create view controllers in code, or both? You could create a protocol for the destination and conform view controllers to it, but for this example I'm going to build the navigation router around storyboard ids.

We'll start by subclassing `Router` using our own concrete route destination. For convenience we'll also add a custom `register` method.

```swift
protocol RoutedViewController {
	func present(url: URL, parameters: [String: String])
}

struct NavigationDestination {
	enum Transition {
		case push
		case modal(UIModalPresentationStyle)
	}
	var transition: Transition
	var storyboardIdentifier: String
	
	init(storyboardIdentifier: String, transition: Transition = .push) {
		self.storyboardIdentifier = storyboardIdentifier
		self.transition = transition
	}
}

class NavigationRouter: Router<NavigationDestination> {
	let storyboard: UIStoryboard
	init(storyboard: UIStoryboard) {
		self.storyboard = storyboard
	}
	
	func register(_ pattern: String, storyboardIdentifier: String, transition: NavigationDestination.Transition = .push) {
		self.register(pattern, destination: NavigationDestination(storyboardIdentifier: storyboardIdentifier, transition: transition))
	}
}
```

This navigation router will not just serve up storyboard ids though, it will also handle actually instantiating them and routing to them. In order to create view controllers though, it will need a storyboard. When routing to a destination, we'll also need a source view controller.

```swift
class NavigationRouter: Router<NavigationDestination> {
	static var shared: NavigationRouter?
	
	let storyboard: UIStoryboard
	init(storyboard: UIStoryboard) {
		self.storyboard = storyboard
	}
	
	// ...
	
	@discardableResult
	func route(to url: URL, from: UIViewController) -> UIViewController? {
		guard let match = self.match(for: url) else { return nil }
		let viewController = storyboard.instantiateViewController(withIdentifier: match.destination.storyboardIdentifier)
		
		if let viewController = viewController as? RoutedViewController {
			viewController.present(url: url, parameters: match.parameters)
		}
		
		from.show(viewController, using: match.destination.transition)
		
		return viewController
	}
}

extension UIViewController {
	@discardableResult
	func route(to url: URL) -> UIViewController? {
		return NavigationRouter.shared?.route(to: url, from: self)
	}
	
	func show(_ viewController: UIViewController, using transition: NavigationDestination.Transition) {
		switch transition {
		case .push:
			if let navigationController = self.navigationController {
				navigationController.pushViewController(viewController, animated: true)
			} else {
				// if we aren't in a navigation controller, fallback to a modal presentation style
				let navigationController = UINavigationController(rootViewController: viewController)
				viewController.navigationItem.leftBarButtonItem = UIBarButtonItem(barButtonSystemItem: .cancel, target: self, action: #selector(cancel))
				
				self.present(navigationController, animated: true, completion: nil)
			}
		case .modal(let style):
			viewController.modalPresentationStyle = style
			self.present(viewController, animated: true, completion: nil)
		}
	}
	
	@IBAction func cancel() {
		self.presentingViewController?.dismiss(animated: true, completion: nil)
	}
}
```

We can setup our router in the app delegate, along with it's routes:

```swift
	func application(_ application: UIApplication, didFinishLaunchingWithOptions launchOptions: [UIApplicationLaunchOptionsKey: Any]?) -> Bool {
		let router = NavigationRouter(storyboard: window!.rootViewController!.storyboard!)
		router.register("/", storyboardIdentifier: "root")
		router.register("/posts", storyboardIdentifier: "postsIndex")
		router.register("/posts/:postID", storyboardIdentifier: "postDetail", transition: .modal(.formSheet))
		NavigationRouter.shared = router
		
		return true
	}
```

Of course, this assumes that those view controllers exist in your storyboard. The view controller's don't necessarily need to impliment `RoutedViewController`, but if they do they can update their state based on the url.

With all that in place, we can route to a url from another view controller like this:

```swift
self.route(to: URL(string: "/posts")!)
```

I'm not going to go in to all the details of how I setup the Storyboard in the example app, you can check out the sample code for more details on that, but here is a UI test I wrote to verify that our in app navigation is working with our router:

```swift
    func testUINavigation() {
        let app = XCUIApplication()
		
		app.buttons["Posts"].tap()
		app.tables.cells.staticTexts["Post 2"].tap()
		XCTAssert(app.staticTexts["Post: 2"].exists)
		app.navigationBars["Post"].buttons["Posts"].tap()
		app.navigationBars["Posts"].buttons["Done"].tap()
		XCTAssert(app.staticTexts["First View"].exists)
    }
```

Next lets add a custom url scheme. This isn't strictly necessary if you also have universal links setup, but I have found that eventually you will want to explicitly open the app instead of potentially opening a website. It is also a lot easier to test and demo because universal links aren't supported in the simulator and require a domain name and some work on your server.

To add a custom url scheme, open up your Xcode project and go to Info > URL Types. You can add a scheme there. The only field that is actually requires is "URL Scheme" and I am going to use "routesexample".

We can write another UI test that uses Safari to open a url that should open in our app:

```swift
	func testURLScheme() {
		let safari = XCUIApplication(bundleIdentifier: "com.apple.mobilesafari")
		safari.launch()

		safari.buttons["URL"].tap()
		safari.typeText("routesexample:///posts\n")
		safari.otherElements["WebView"].buttons["Open"].tap()
		

		let app = XCUIApplication()
		XCTAssert(app.navigationBars["Posts"].buttons["Done"].exists)
	}
```

If you run that, it will open our app, but it won't navigate to posts, and will fail. We need to handle the open url event in our app delegate:

```swift
	func application(_ app: UIApplication, open url: URL, options: [UIApplicationOpenURLOptionsKey : Any] = [:]) -> Bool {
		let result = window?.rootViewController?.route(to: url)
		return result != nil
	}
```

And with that, our url scheme UI tests should pass. We just use our root view controller as the source of the route.

## Other sources of urls

I'm not going to go into how to setup universal links here, but you can [read all about it from Apple](https://developer.apple.com/library/archive/documentation/General/Conceptual/AppSearch/UniversalLinks.html). The delegate method to handle a universal link is a little different, instead using the user activity mechanism:

```swift
	func application(_ application: UIApplication, continue userActivity: NSUserActivity, restorationHandler: @escaping ([Any]?) -> Void) -> Bool {
		if let url = userActivity.webpageURL {
			let result = window?.rootViewController?.route(to: url)
			return result != nil
		}
		
		return false
	}
```

You're method might be a bit more complex here if you are also handling other user activity types. You can check for [NSUserActivityTypeBrowsingWeb](https://developer.apple.com/documentation/foundation/nsuseractivitytypebrowsingweb) if you want to handle universal links different from other user activities that might also have a webpage url. Although using routes is a great way to impliment handoff.

There are a lot of other ways a url based navigation system can be useful as well. I personally use this for our push notifications. We include a path for the content in our payload that the app looks for and routes to.