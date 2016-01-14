---
layout: post
title: Why I Love Swift
comments: true
tags: Swift
---

Because.

This would take a lot more lines in Objective-C, and be less clear.

![]({{ site.url }}/assets/SwiftLove/swift_love.png)

Here's the extension for it:

{% highlight swift %}
extension Array {
    func partitionBy(predicate: (Element -> Bool)) -> ([Element], [Element]) {
        var left = [Element]()
        var right = [Element]()
        
        for elem in self {
            if predicate(elem) {
                left.append(elem)
            } else {
                right.append(elem)
            }
        }
        
        return (left, right)
    }
  }
{% endhighlight %}