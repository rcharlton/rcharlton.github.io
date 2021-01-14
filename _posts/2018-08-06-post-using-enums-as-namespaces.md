---
title: "Using Enums as Namespaces"
excerpt: "Syntactic Sugar"

date: 2018-08-06T09:00:00+0000

header:
  teaser: /assets/images/posts/post-using-enums-as-namespaces/patrick-fore-389419-unsplash.jpg
  overlay_image: /assets/images/posts/post-using-enums-as-namespaces/patrick-fore-389419-unsplash.jpg
  overlay_filter: 0.5
  caption: "Photo by Patrick Fore on [**Unsplash**](https://unsplash.com)"

categories:
  - blog

tags:
  - Swift

classes: wide
---
A few code-bases I’ve come across have used instance properties to hold constants, as follows.

```swift
class LoginViewController: UIViewController {
    private let headerSpace: CGFloat = 24
}
```

This is convenient since the value can be referenced within the class without any additional prefix or scoping. 

However this approach is problematic because it suggests that the value varies between instances of the type when in fact it does not. A `static` property better represents the real semantics of the situation.

```swift
class LoginViewController: UIViewController {
    private static let headerSpace: CGFloat = 24
}
```

This is less convenient to use since the property must be qualified with the classname when referenced within instance methods it , e.g. `LoginViewController.headerSpace`. I don’t know about you but I don’t usually need reminding what class I’m working in!

A better approach is to embed the property within a case-less enum, as follows:

```swift
class LoginViewController: UIViewController {
    private enum LayoutMetric {
        static let headerSpace: CGFloat = 24
    }
}
```

Then the property can be referenced internally as `LayoutMetric.headerSpace`. With this convention we can group semantically related constants and the qualified name tells us what it relates to.

Using an enum in this way is similar to the `namespace` concept in C++ which provides a first class way of collecting together type names.

An enum with no cases is known as an **uninhabited** type. It is a type that cannot be instantiated. As a result there’s no way a sleep deprived developer could start using LayoutMetric variables by mistake.
