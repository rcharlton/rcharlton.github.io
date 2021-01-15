---
title: "UIColor + Transformed"
excerpt: "Extension"

date: 2019-02-16T09:00:00+0000
last_modified_at: 2021-01-09T16:20:02-05:00

header:
  teaser: /assets/images/posts/post-uicolor-transformed/russn_fckr-66974-unsplash.jpg
  overlay_image: /assets/images/posts/post-uicolor-transformed/russn_fckr-66974-unsplash.jpg
  overlay_filter: 0.5
  caption: "Photo by russn_fckr on [**Unsplash**](https://unsplash.com)"

categories:
  - blog

tags:
  - UIKit

classes: wide
---
I really enjoyed reading Erik D. Kennedy’s [post](https://medium.com/@erikdkennedy/color-in-ui-design-a-practical-framework-e18cacd97f9e) on Color in UI Design. In it he discusses how the ability to modify one base colour into many variations is a fundamental skill in colouring interface designs. Erik examines colours in the real world and shows how hue, saturation and brightness vary in different lighting conditions.

[Color in UI Design: A (Practical) Framework – Erik D. Kennedy – Medium](https://medium.com/@erikdkennedy/color-in-ui-design-a-practical-framework-e18cacd97f9e)

He observes how darker colour variations have higher saturation and lower brightness. But if saturation and brightness are unchanged, shifting hue towards red, green or blue will decrease the luminosity, or perceived lightness of the colour. And shifting the hue towards yellow, cyan, or magenta will increase the perceived lightness of the colour.

![Tree shadow image](/assets/images/posts/post-uicolor-transformed/jon-flobrant-229724-unsplash.jpg "Photo by Jon Flobrant on Unsplash"){:height="500px" width="400px"}

![Door image](/assets/images/posts/post-uicolor-transformed/thanos-pal-1146444-unsplash.jpg "Photo by Thanos Pal on Unsplash"){:height="500px" width="400px"}

Armed with this knowledge it seems impossible to resist writing an extension on `UIColor` to encode this.

### UIColor+Transformed.swift
```swift
extension UIColor {

    struct Transform {
        /// The offset applied to hue, in degrees 0...+-360.
        let hue: CGFloat

        /// The % change to the saturation.
        let saturation: CGFloat

        /// The % change to the brightness.
        let brightness: CGFloat

        /// color.transformed(by: identity) is always equal to color.
        static let identity = Transform(hue: 0, saturation: 0, brightness: 0)
    }

    /// Compute the color formed by applying the given transform to this color.
    ///
    /// - Parameter transform: Describes the changes to the color.
    /// - Returns:             The resulting colour.
    func transformed(by transform: Transform) -> UIColor {
        let hsba = UnsafeMutablePointer<CGFloat>.allocate(capacity: 4)
        hsba.initialize(repeating: 0.0, count: 4)

        getHue(&hsba[0], saturation: &hsba[1], brightness: &hsba[2], alpha: &hsba[3])

        let phase = Int(6 * hsba[0])
        let hueDirection: CGFloat = ((phase % 2) == 0) ? 1.0 : -1.0

        let hue = (hsba[0] + (hueDirection * transform.hue / 360)).clamped(to: 0...1)
        let saturation = (hsba[1] + transform.saturation).clamped(to: 0...1)
        let brightness = (hsba[2] + transform.brightness).clamped(to: 0...1)

        let result = UIColor(hue: hue, saturation: saturation, brightness: brightness, alpha: hsba[3])

        hsba.deinitialize(count: 4)
        hsba.deallocate()

        return result
    }
}

private extension Comparable {

    func clamped(to range: ClosedRange<Self>) -> Self {
        return max(min(self, range.upperBound), range.lowerBound)
    }

}

```

With this we can define some colour transformations and apply them to the base colours in our app's UI.
```swift
let baseColor = UIColor(red: 0.2, green: 0.3, blue: 0.4)

let darkerTransform = UIColor.Transform(hue: +6, saturation: +0.2, brightness: -0.3)
let darkerColor = baseColor.transformed(by: darkerTransform)

let lighterTransform = UIColor.Transform(hue: -6, saturation: -0.2, brightness: +0.3)
let lighterColor = baseColor.transformed(by: lighterTransform)
```

### Example App
The screenshot below is from a simple example app that replicates the illustrations in Erik’s article.

![](/assets/images/posts/post-uicolor-transformed/screenshot.png)

The middle column is the base colour and automatically transformed lighter and variations are shown on either side. The latest version of this extension along with the full source code of the app is available [here](https://github.com/rcharlton/ColorTransform). 
