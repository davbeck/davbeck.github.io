---
layout: post
title: Managing login and authentication for iOS
date: 2018-08-03
tags: [Swift, iOS]
redirect_from:
  - /blog/2018/08/03/login-app-flow.html.html
---

For many if not most apps these days, some kind of login and authentication is required. Ideally your app should have some kind of benefit without an account. Something to give users value before they make the commitment of creating an account, but for some services that just isn't possible.

Managing this state can be a bit more difficult for iOS apps than web sites though. A website can just redirect to the login page if it realizes a user isn't logged in. But for iOS, we need to maintain a single view hierarchy.

## The naïve approach

For years I used an approach similar to this: setup your UI for a logged in user, making that your root view controller. Then, whenever the user logs out or their session expires, show a modal login screen.

There are a couple of problems with this approach:

First, you have to account for multiple parts of your code trying to show the login screen. At first this can be simple: when your app launches check if the user is logged in and show the login screen if needed and show it if the logout button is ever tapped. But for most apis, your session can expire at any time, either from a timeout or from a password reset. So any api request could trigger a logout, and you may have multiple api requests in progress when your session expires. It's certainly possible to handle this, but it adds complexity.

More pressing however is that when you show your login screen as a modal *over* your regular authenticated UI, you have to think about what happens to that interface when the user isn't logged in. Do you clear out all the data and have a ghost UI hidden from view? In which case you need to be careful not to clear it out until after the login screen animates in completely.

## A stateful approach

An approach I've started using more recently instead puts the control in a root view controller. This view controller owns the login, and switches between views as needed. Here's how you would impliment that for your app. The approach is simple enough, and login workflows are complicated enough, that it makes more sense to impliment this yourself then to pull in a 3rd party library.

Our root view controller will be what manages the login state. It will then set a child view controller based on that state. You could impliment view controller containment here yourself, and doing so has certainly gotten simpler in recent years, but I like to use a `UIPageViewController` for this because it handles those details for us.

```swift
class AuthenticationViewController: UIPageViewController {
	private let sessionPreferenceKey = "LoginSession"
	private(set) var session: String?
	
	private func login(session: String) {
		self.session = session
		UserDefaults.standard.set(session, forKey: sessionPreferenceKey)
	}
	
	private func commonInit() {
		self.session = UserDefaults.standard.value(forKey: sessionPreferenceKey) as? String
	}
	
	init() {
		super.init(transitionStyle: .scroll, navigationOrientation: .horizontal)
		
		commonInit()
	}
	
	required init?(coder: NSCoder) {
		super.init(coder: coder)
		
		commonInit()
	}
}
```

Normally you would want your login state managed by a dedicated controller class and stored in the keychain, but for simplicity sake we are just managing it from our view controller and storing it in preferences. Your apis session might be more complicated than a single string. You might have information about the current user, or permissions, but your will almost always have some sort of token, which is what our session variable represents.

Page view controllers technically support showing multiple view controllers at once, and animating between them. We only want to show a single view controller at a time, and we want to use our own animation. Here's a quick helper to handle that:

```swift
	public var currentViewController: UIViewController? {
		get {
			return viewControllers?.first
		}
		set {
			let viewControllers = newValue.map({ [$0] })
			setViewControllers(viewControllers, direction: .forward, animated: false, completion: nil)
		}
	}
	
	public func set(currentViewController viewController: UIViewController, direction: UIPageViewControllerNavigationDirection, animated: Bool = true, completion: (() -> Void)? = nil) {
		guard currentViewController != viewController else { completion?(); return }
		
		if let window = self.view.window, animated {
			let direction: UIViewAnimationOptions = (direction == .forward) ? .transitionFlipFromRight : .transitionFlipFromLeft
			UIView.transition(with: window, duration: 0.75, options: [.layoutSubviews, direction], animations: {
				self.currentViewController = viewController
			}) { completed in
				completion?()
			}
		} else {
			currentViewController = viewController
			completion?()
		}
	}
```

Notice that we are disabling animation when we call the page view controller's `setViewControllers`, but wrapping it in our own `UIView.transition`. Here I'm using a flip animation, but you could use any transition that makes sense for your app.

We'll trigger this from an update function:

```swift
	private func login(session: String) {
		guard self.session != session else { return }
		self.session = session
		UserDefaults.standard.set(session, forKey: sessionPreferenceKey)
		
		update(animated: true)
	}
	
	func logout() {
		guard self.session != nil else { return }
		self.session = nil
		UserDefaults.standard.removeObject(forKey: sessionPreferenceKey)
		
		update(animated: true)
	}
	
	override func viewDidLoad() {
		super.viewDidLoad()
		
		update()
	}
	
	private func update(animated: Bool = false) {
		if session == nil {
			let viewController = self.storyboard!.instantiateViewController(withIdentifier: "Login")
			set(currentViewController: viewController, direction: .reverse, animated: animated, completion: nil)
		} else {
			let viewController = self.storyboard!.instantiateViewController(withIdentifier: "Authenticated")
			set(currentViewController: viewController, direction: .forward, animated: animated, completion: nil)
		}
	}
```

I'm using storyboards here, but creating view controllers in code would work just as well. Also notice that in login and logout, we are checking to make sure that we aren't actually updating if our session hasn't changed. This will protect against repeated calls for whatever reason. In theory, you could call login with a different session, and what you do in that case, if you allow it at all, depends on your app. Perhaps your token refreshes regularly and you don't need to update your UI at all when it does, you can disable updates if you are already logged in. Or maybe your app supports multiple accounts, this model works well for transitioning between those views as well.

If you were to launch this app right now (assuming you have a proper storyboard setup), it would just show the login screen without any way to transition to the logged in state. There are many ways to wire up communication between the two, but a delegate works well.

```swift
protocol LoginViewControllerDelegate: class {
	func loginViewController(_ loginViewController: LoginViewController, didLoginWithSession session: String)
}

class LoginViewController: UIViewController {
	weak var delegate: LoginViewControllerDelegate?
	
	@IBOutlet weak var emailField: UITextField!
	@IBOutlet weak var passwordField: UITextField!
	
	@IBAction func login(_ sender: Any) {
		let session = emailField.text ?? ""
		delegate?.loginViewController(self, didLoginWithSession: session)
	}
}
```

Make sure to conform to the deleage protocol:

```swift
extension AuthenticationViewController: LoginViewControllerDelegate {
	func loginViewController(_ loginViewController: LoginViewController, didLoginWithSession session: String) {
		self.login(session: session)
	}
}
```

`AuthenticationViewController.update` function:

```swift
let viewController = self.storyboard!.instantiateViewController(withIdentifier: "Login") as! LoginViewController
viewController.delegate = self
```

Using the user's email as a login session isn't a good idea, and you'll probably want to ask your api if their password is correct, but this will work for now.

We also need a way for the authenticated view controllers to logout. We could use a delegate here as well, but the logout action could come from several places, and may be burried in several view controllers. Instead, I like to use an extension on `UIViewController` like this:

```swift
extension UIViewController {
	var authenticationViewController: AuthenticationViewController? {
		if let authenticationViewController = parent as? AuthenticationViewController {
			return authenticationViewController
		}
		
		return parent?.authenticationViewController
	}
}
```

That way any child view controller can just call `self.authenticationViewController?.logout()`. And with that you have a working login and logout management system. You can see a complete working example at [https://github.com/davbeck/LoginExample](https://github.com/davbeck/LoginExample).

## Going further

This model provides a base for all kinds of login and authentication management. For instance, you could show a different authenticated view based on what kind of user has logged in, like if you have basic and pro accounts, or an admin vs a regular user. Or instead of a login screen, your unauthenticated view could be a slimmed down version of your regular UI.