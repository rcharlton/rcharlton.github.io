---
title: "Improving Collection View Data Sources"
excerpt: "Syntactic Sugar"

date: 2018-09-22T20:10:00+0000

header:
  teaser: /assets/images/posts/post-improving-collection-view/priscilla-fong-83012-unsplash.jpg
  overlay_image: /assets/images/posts/post-improving-collection-view/priscilla-fong-83012-unsplash.jpg
  overlay_filter: 0.5
  caption: "Photo by Priscilla Fong on [**Unsplash**](https://unsplash.com)"

categories:
  - blog

tags:
  - UIKit
  - Syntactic Sugar

classes: wide
---
Since `UICollectionView` arrived back in iOS 6.0 it’s become the workhorse of UI development. It can be seen everywhere rendering the dynamic content of the interweb’s feeds, lists and stories. Although capable the API suffers from that particular clunkiness that only an Objective-C Cocoa API can give 😆. This post presents a little syntactic sugar to improve the readability of our code and eliminate stupid programming mistakes.

Let’s look at a typical collection view example. First we register our cells:
```swift
collectionView.register(
    MyCollectionViewCell.self, 
    forCellWithReuseIdentifier: "MyCollectionViewCellIdentifier"
)
```

... and then consume those cells in the following data-source method:
```swift
func collectionView(
    _ collectionView: UICollectionView, 
    cellForItemAt indexPath: IndexPath
) -> UICollectionViewCell {
    guard let cell = collectionView.dequeueReusableCell(
        withReuseIdentifier: "MyCollectionViewCellIdentifier", 
        for: indexPath
    ) as? MyCollectionViewCell else {
        fatalError("Gosh, this is embarressing 😭")
    }

    cell.configure(with: myData(for: indexPath))

    return cell
}
```

What are the problems here? We must make an ad hoc association between the cell type `MyCollectionViewCell` and its string reuse identifier and then specify it in multiple places. Value identifiers like these are fragile as they can only be verified at run-time. There’s no way the compiler can catch a simple typo in a string.

Looking at `func collectionView(_ , cellForItemAt indexPath:)` we see that we downcast the cell to configure its subviews. Naturally this cast could fail if the cell were of an unexpected type and that leaves us with an *optional* `UICollectionViewCell` variable. Since the method itself returns a *non-optional* cell we’re left with a contract we cannot fulfil. We can’t return nil nor raise an exception and are left with no way to fail gracefully.

The path to improving this situation is by more formally expressing the relationship of the cell type with its reuse identifier.

```swift
protocol Reusable {
    static var reuseIdentifier: String { get }
}
```

```swift
class MyCollectionViewCell: UICollectionViewCell, Reusable {
    func configure(with data: MyData) {
        ...
    }
}
```
With this in place we can extend `UICollectionView` and eliminate the manual specification of the reuse identifier for all `Reusable` conforming types:

```swift
extension UICollectionView {
    func register<T: UICollectionViewCell>(_: T.Type) where T: Reusable {
        register(T.self, forCellWithReuseIdentifier: T.reuseIdentifier)
    }

    func dequeueReusableCell<T: UICollectionViewCell>(for indexPath: IndexPath) -> T where T: Reusable {
        guard let cell = dequeueReusableCell(withReuseIdentifier: T.reuseIdentifier, for: indexPath) as? T else {
            fatalError(“Failed to dequeue cell: \(T.reuseIdentifier)”)
        }
        return cell
    }
}
```

This leads to a greatly simplified call-site when both registering the cell type and consuming its instances:
```swift
collectionView.register(MyCollectionViewCell.self)
```
```swift
func collectionView(
    _ collectionView: UICollectionView, 
    cellForItemAt indexPath: IndexPath
) -> UICollectionViewCell {
    let cell: MyCollectionViewCell = collectionView.dequeueReusableCell(for: indexPath)
    cell.configure(with: myData(for: indexPath))
    return cell
}
```

For most cases there's little need to manually specify the reuse identifier. We can rely on Swift's type information to supply this.
```swift
extension Reusable {
    static var reuseIdentifier: String {
        return String(describing: type(of: self))
    }
}
```

You may be wondering about that `fatalError` call in the new dequeue method. Whilst we can’t avoid the cell downcast it is now isolated to a single location rather than repeated in every `UICollectionViewDataSource` of our code-base. To actually eliminate the `fatalError` we must satisfying the method's return type. This can be achieved by instantiating a new cell type to indicate the error rather than halting the app. 

```swift
func dequeueReusableCell<T: UICollectionViewCell>(for indexPath: IndexPath) -> T where T: Reusable {
    guard let cell = dequeueReusableCell(withReuseIdentifier: T.reuseIdentifier, for: indexPath) as? T else {
        return MyErrorCollectionViewCell(message: "Failed to dequeue cell: \(T.reuseIdentifier)"))
    }
    return cell
}
```