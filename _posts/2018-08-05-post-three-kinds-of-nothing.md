---
title: "Three Kinds of Nothin"
excerpt: "nil, Void & Never"

date: 2018-08-05T09:00:00+0000

header:
  teaser: /assets/images/posts/post-three-kinds-of-nothing/thierry-meier-218997-unsplash.jpg
  overlay_image: /assets/images/posts/post-three-kinds-of-nothing/thierry-meier-218997-unsplash.jpg
  overlay_filter: 0.5
  caption: "Photo by Thierry Meier on [**Unsplash**](https://unsplash.com)"

categories:
  - blog

tags:
  - Swift

classes: 
	- wide
---
A gem from Advanced Swift [objc.io](https://www.objc.io/books/advanced-swift/) (credited to David Smith on [Twitter](https://twitter.com/Catfish_Man/status/825080948555292672)) is the appreciation of Swift’s three representations of nothing. Although the language's syntactic shortcuts give us succinct code they sometimes mask the details and in doing so blunt our understanding of how things work. 

This post explores and contrasts the meaning of the three kinds of nothing: nil, Void and Never.


## nil — The absence of a thing
Everyone is familiar with the value of `nil`, or more specifically `Optional.none`. It's the complement to `Optional.some` and the semantic analogue of C languages' `NULL`.

```swift
var aThing: Optional<Thing> = .none
````


## Void —  The Presence of nothing
The Standard Library describes the type `Void` as *"the return type of functions that don’t explicitly specify a return type"*.  So we might deduce that functions and methods either return *something* or they return the *presence of nothing*:
```swift
func f() -> String {
	print("Shall we play a game?")
	return "something"
}
```
```swift
func f() -> Void {
	print("Shall we play a game?")
	return Void()
}
```

Or equivalently:

```swift
func f() {
	print("Shall we play a game?")
}
```

Since `Void` is a type we can instantiate an instance of it.
```swift
var nothing: Void = f()
```
And we can also test its equality to show that all kinds of nothing are the same!
```swift
assert(nothing == Void())
```

The Standard Library actually defines `Void` as equivalent to the empty tuple; giving rise to another equivalent function declaration.
```swift
public typealias Void = ()
````
```swift
func f() -> () {	
	print("Shall we play a game?")
	return ()
}
```

But now things are strange. The empty tuple expression `()` is both the type and the value of *nothing*.

```swift
var nothing: () = ()
```

## Never — The thing which cannot be
Never is a type that has no valid values and cannot be instantiated. 

Consider this enum type with no cases. 
```swift
public enum Never { 
}
```

Such types are termed *uninhabited*. The set of all possible instances of `Never` is empty.

This leads us to a situation where we can declare a variable of the type but have no values with which to initialise it.
```swift
var never: Never = ???
````

OK, perhaps we can declare a function that returns type `Never` and assign that? 

```swift
func f() -> Never {
	print("Shall we play a game?")
	// return ???
}
```

But how can a function return a value of a type that cannot be instantiated? 

Swift gives the compilation error `Function with uninhabited return type 'Never' is missing call to another never-returning function on all paths` for this declaration. To silence the error the function must call another function that returns `Never` like e.g. `fatalError`.

`Never` tells the compiler that the function will **not return**. This contradicts the previous idea that *"functions and methods either return something or nothing"*. In fact, sometimes they don't return at all!


```swift
func f() -> Never {
	fatalError("The only winning move is not to play")

	// Execution does not continue.
}
```

And now we can compile code that initialises a value of type `Never`. 

```swift
var never: Never = f()
````
Of course this initialisation is guaranteed to **never** run.
