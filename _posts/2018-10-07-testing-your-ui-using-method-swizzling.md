---
layout: post
title: Testing your UI using method swizzling
---

Bugs come in many forms, and one of the classics is when our app's UI is messed up by rendering data that is
either much larger or much smaller than we expected during development. For example, if you've ever done work on an app, chances are good you've come across a label which overlaps other elements when it's text gets too long, or clips the text in some unexpected, unwanted way. In this post, I'm going to show a neat trick on how to easily test the entire UI for this without adding lots of code, using a technique called *method swizzling*.

### Method swizzling

In short, swizzling is a technique available through the *Objective C runtime* library, which allows you to make a swap so that the *selector*, or name, of method *m1* points to the implementation of another method *m2*, and vice versa. I won't go into detail about swizzling or the Objective C runtime in general in this article, but I really recommend reading [Apple's documentation](https://developer.apple.com/library/archive/documentation/Cocoa/Conceptual/ObjCRuntimeGuide/Introduction/Introduction.html#//apple_ref/doc/uid/TP40008048) on the matter, to get a feeling for the extremely dynamic nature of what goes on under the hood when you develop using iOS and macOS frameworks. More of that stuff in a future post!

To give an example of how swizzling could be used: With a few lines of code we could replace the ```viewDidLoad()``` implementation of every single instance of UIViewController, so that every time it is called on a UIViewController or any of it's subclasses, our customized version that we might wanna call ```swizzledViewDidLoad()``` would be executed instead. Note that this is different from overriding a method in a subclass, since we by swizzling can replace the implementations of the actual base classes in Apple's closed source frameworks such as UIKit and Foundation. Cool stuff for sure!

### Testing labels

To help us debug our UI in order to detect nasty UILabel bugs, we are going to replace the method that is called when setting the text property of a UILabel

```swift
label.text = "This text will be swizzled away!"
```

The reason for doing this is that we want to define a really long text with great potential to mess up the UI, and we want to be able to make every single label in the app show this text simultaneously, whether a base class or a subclass instance. In this way we can easily see how long strings are handled as our UI evolves. What's nice about using swizzling in this case, is that we can accomplish what we want without having to make any changes to the call sites where our labels are used. This of course saves us time and prevents us from polluting our code with debug-specific stuff.

Basically, we're just going to use 2 methods from the Objective C Runtime library:

```swift
//Gets a reference to the method in class 'cls' with the name 'name'
func class_getInstanceMethod(_ cls: AnyClass?,
                           _ name: Selector) -> Method?

//swaps m1 and m2, so that calling m1 executes the implementation of m2, and vice versa
func method_exchangeImplementations(_ m1: Method,
                                  _ m2: Method)                        
```

Since we might want to do a whole lot of swizzling in our project, we'll wrap these methods in a generic ```Swizzler``` struct to have a bit more pleasant API to work with

```swift
struct Swizzler<T: NSObject> {
    static func swizzle(_ originalSelector: Selector,
                        _ swizzledSelector: Selector) {
        //The backticks let us use 'class' a variable name,
        //despite it being a reserved keyword
        let `class`: AnyClass! = T.self

        guard
            let originalMethod = class_getInstanceMethod(`class`, originalSelector),
            let swizzledMethod = class_getInstanceMethod(`class`, swizzledSelector) else {
            return
        }

        method_exchangeImplementations(originalMethod, swizzledMethod)
    }
}
```

We'll then extend UILabel with a method called ```setReallyLongText```, which will replace the ```label.text``` implementation. We'll also add a static func which performs the actual implementation swap via our Swizzler struct  

```swift
extension UILabel {
    @objc func setReallyLongText(_ text: String) {
        let replacementText = """
        Bacon ipsum dolor amet turducken ground round cow
        pastrami prosciutto cupim picanha, sirloin ham
        bresaola leberkas corned beef ball tip. Spare ribs
        pork chop short ribs pork tenderloin sausage
        tongue biltong.
        """
        self.setReallyLongText(replacementText)
    }

    static func swizzleSetReallyLongText() {
        Swizzler<UILabel>.swizzle(#selector(setter: text), #selector(setReallyLongText(_:)))
    }
}
```

We can then call ```UILabel.swizzleSetReallyLongText()``` somewhere early in our app's lifecycle, and by doing so, all code in the app using the ```.text``` setter will instead call our ```setReallyLongText``` method with the given string passed as an argument.

Note that our method seems to call itself with the *replacementText* we've defined, which is actually not as recursive as it might come across
```swift
@objc func setReallyLongText(_ text: String) {
    ...
    self.setReallyLongText(replacementText)
}
```

Remember that we swapped names for UILabel's text setter, and our own method. So when ```label.text``` is invoked in our code, the implementation of ```setReallyLongText``` is executed, and when ```setReallyLongText``` is invoked in our code, the implementation of ```label.text``` is executed. So the above method hijacks the text setter method, replaces the string passed to it, and then passes this new string along to the original setter.

### With great power comes great responsibility

Although method swizzling might seem bordering on black magic, using it in your code should not be a problem as far as getting your app to App Store is concerned. However, it's still something that could seriously mess up your code if you don't tread with caution. Since we have no idea what the actual implementation of methods in UIKit or Foundation look like, there's a risk of swizzling away some very important functionality which will cause trouble down the road. It's likely a good idea to call the original implementation in your hijacked method unless you have a very good reason not too. And unless you *really* know what you're doing, it's probably a good idea *not* use this stuff in production.

 When having code that is debug/test specific, it's good practice to create a separate target in Xcode. By doing this you can add custom flags to your target which will activate test-specific functionality when building that target only. In this way you don't have to manually activate/deactivate debug features in your code whenever you want to test things, which is both tedious and risky, should you ever forget to change something back before shipping your app.

 In this case, I added a specific *Test* target to my project and added a *TEST* flag under *Build Settings -> Swift Compiler - Custom Flags -> Active Compilation Conditions*. I then put the following code in my AppDelegate's ```didFinishLaunching...``` method to make sure swizzling is left out whenever we're not testing

 ```swift
#if TEST
UILabel.swizzleSetReallyLongText()
#endif
```

### Other uses of swizzling

Hopefully this post can serve as an inspiration for coming up with other ways to use swizzling for debugging purposes. One use case could be if you wanted to log something whenever a certain method is invoked. For example, you might want to know whenever ```viewDidLoad()``` is called in your app's different view controllers. Instead of adding logging code to every single view controller class, you could easily hijack ```viewDidLoad()``` on UIViewController and add your logging code there.

This might look something like this

```swift
extension UIViewController {
    @objc func logViewDidLoad() {
        Logger.log("viewDidLoad executed for \(type(of: self))")
        logViewDidLoad()
    }

    static func swizzleLogViewDidLoad() {
        Swizzler<UIViewController>.swizzle(#selector(viewDidLoad), #selector(logViewDidLoad))
    }
}
```

I really hope you enjoyed this post, and I would love to hear from you on Twitter. Thank you so much for reading!
