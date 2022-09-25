---
title: "Experimenting With AsyncStream"
excerpt: "Streaming locations in modern Swift"

date: 2022-09-25T09:00:00+0000 

header:
  teaser: /assets/images/posts/post-experimenting-with-async-stream/jean-frederic-fortier-RkBTPqPEGDo-unsplash.jpg
  overlay_image: /assets/images/posts/post-experimenting-with-async-stream/jean-frederic-fortier-RkBTPqPEGDo-unsplash.jpg
  overlay_filter: 0.5
  caption: "Photo by Jean-Frederic Fortier on [**Unsplash**](https://unsplash.com)"

categories:
  - blog

tags:
  - Swift

classes: wide
---
`AsyncSequence` and `AsyncStream` are mechanisms to model an asynchronous stream of values using Swift's native structured concurrency. As a fan of functional reactive programming and functional composition but one who winces at the syntactic complexity of Combine or Rx then this has a lot of appeal.

This post is a brain dump that documents an afternoon spent noodling about with `CLLocationManager` and `AsyncStream`. Credit to Andy Ibanez who's comprehensive [post](https://www.andyibanez.com/posts/understanding-actors-in-the-new-concurrency-model-in-swift/) on concurrency inspired this effort.

The goal was to write code like:

```swift
let altitudes = locations()
    .filter { $0.verticalAccuracy < kCLLocationAccuracyNearestTenMeters }
    .map { $0.altitude }

for await altitude in altitudes {
    print(altitude)
}

```

This is achieved with the following function. The use of a function rather than e.g. a wrapper class is to eliminate shared state (within my code at least - who knows what Apple is doing?!) and reduce the potential for side-effects.

```swift
// CLLocationManager doesn't appear to function correctly off the main thread.
@MainActor
func locations() -> AsyncStream<CLLocation> {
    let locationManager = CLLocationManager()
    let locationHandler = LocationHandler()
    locationManager.delegate = locationHandler

    var shouldStart = false
    var continuation: AsyncStream<CLLocation>.Continuation?

    locationHandler.didChangeAuthorization = { locationManager in
        if locationManager.authorizationStatus == .denied {
            continuation?.finish()
        } else if shouldStart {
            shouldStart = false
            locationManager.startUpdatingLocation()
        }
    }

    locationHandler.didFailWithError = { locationManager, error in
        if locationManager.authorizationStatus != .notDetermined {
            continuation?.finish()
        }
    }

    locationHandler.didUpdateLocations = { locationManager, locations in
        locations.forEach { continuation?.yield($0) }
    }

    return AsyncStream<CLLocation> {
        continuation = $0

        $0.onTermination = { @Sendable _ in
            // Use this escaping closure to retain these objects whilst the stream is active.
            // This appears to break the Sendable constraint since these reference types are not isolated.
            _ = locationHandler
            _ = locationManager
        }

        if locationManager.authorizationStatus == .notDetermined {
            shouldStart = true
            locationManager.requestWhenInUseAuthorization()
        } else {
            locationManager.startUpdatingLocation()
        }
    }

}

// MARK: -

private class LocationHandler: NSObject, CLLocationManagerDelegate {

    var didChangeAuthorization: (CLLocationManager) -> Void = { _ in }
    var didFailWithError: (CLLocationManager, Error) -> Void = { _, _ in }
    var didUpdateLocations: (CLLocationManager, [CLLocation]) -> Void = { _, _ in }

    override init() {
    }

    deinit {
        debugPrint(type(of: self), #function)
    }

    func locationManagerDidChangeAuthorization(_ manager: CLLocationManager) {
        didChangeAuthorization(manager)
    }

    func locationManager(_ manager: CLLocationManager, didFailWithError error: Error) {
        didFailWithError(manager, error)
    }

    func locationManager(_ manager: CLLocationManager, didUpdateLocations locations: [CLLocation]) {
        didUpdateLocations(manager, locations)
    }

}
```

The above really isn't intended to be production code. 

Suggestions for future work include:
- Replacing the use of `CLLocation` with an immutable value type.
- Devising a mechanism to throw errors; a rather severe limitation of `AsyncStream` when compared with Combine or Rx.
- Investigate the behaviour of awaiting streams when the containing `Task` is cancelled.