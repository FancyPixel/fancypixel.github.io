---
layout: post
title: "iOS9 notifications and Text Input"
date: 2015-06-11 11:03:06 +0200
comments: true
categories: ios swift
---

When iOS8 opened the API to have actionable notifications many of us were disappointed that the text input action used in Messages wasn't available. It turns out that with iOS9 this API is now public, although as I'm writing this the documentation doesn't mention it. After some digging in the UIKit changelog I was able to build a sample making use of this new behavior.  
Like with iOS8 the NDA is more relaxed this year, but it still forbids showing screenshots. In any case, you'll find the full sample [on GitHub](https://github.com/FancyPixel/text-notification-sample).  
Let's take a step by step look at what changed.
<!-- More -->

# Requesting the permission
First of all we need to ask the permission to the user before we can start sending notifications. We start off by building a new `UIMutableUserNotificationAction`, that alongside the main properties of the notification will also specify the desired behavior:

~~~swift
let textAction = UIMutableUserNotificationAction()
textAction.identifier = "TEXT_ACTION"
textAction.title = "Reply"
textAction.activationMode = .Background
textAction.authenticationRequired = false
textAction.destructive = false
textAction.behavior = .TextInput
~~~

The `behavior` property is a new addition of iOS9 and holds the `UIUserNotificationActionBehavior` enum:

~~~swift
enum UIUserNotificationActionBehavior : UInt {
  case Default // the default action behavior
  case TextInput // system provided action behavior, allows text input from the user
}
~~~

With that out of the way the rest of the configuration didn't change much from iOS8:

~~~swift
let category = UIMutableUserNotificationCategory()
category.identifier = "CATEGORY_ID"
category.setActions([textAction], forContext: .Default)
category.setActions([textAction], forContext: .Minimal)

let categories = NSSet(object: category) as! Set<UIUserNotificationCategory>
let settings = UIUserNotificationSettings(forTypes: [.Alert, .Badge, .Sound], categories: categories)
UIApplication.sharedApplication().registerUserNotificationSettings(settings)
~~~

One minor difference here is due to the new Swift 2 syntax, instead of:

~~~swift
UIUserNotificationType.Alert | UIUserNotificationType.Sound | UIUserNotificationType.Badge
~~~

we need to use the much nicer

~~~swift
[.Alert, .Sound, .Badge]
~~~

That's it, once the user grants the permission we can schedule a new local notification to test this out:

~~~swift
static func scheduleNotification() {
    let now: NSDateComponents = NSCalendar.currentCalendar().components([.Hour, .Minute], fromDate: NSDate())

    let cal = NSCalendar(calendarIdentifier: NSCalendarIdentifierGregorian)!
    let date = cal.dateBySettingHour(now.hour, minute: now.minute + 1, second: 0, ofDate: NSDate(), options: NSCalendarOptions())
    let reminder = UILocalNotification()
    reminder.fireDate = date
    reminder.alertBody = "You can now reply with text"
    reminder.alertAction = "Cool"
    reminder.soundName = "sound.aif"
    reminder.category = "CATEGORY_ID"

    UIApplication.sharedApplication().scheduleLocalNotification(reminder)

    print("Firing at \(now.hour):\(now.minute+1)")
}
~~~

#Receiving the response
Once the user activates the action and writes the reply our `AppDelegate` is called, receiving a new delegate message introduced in iOS9:

~~~swift
func application(application: UIApplication, handleActionWithIdentifier identifier: String?, forLocalNotification notification: UILocalNotification, withResponseInfo responseInfo: [NSObject : AnyObject], completionHandler: () -> Void) {
~~~

The text is stored in the `responseInfo` under the `UIUserNotificationActionResponseTypedTextKey` key:

~~~swift
let reply = responseInfo[UIUserNotificationActionResponseTypedTextKey]
~~~

And that's it. A welcome addition to the API, let's hope that all the major messaging apps will adopt it soon.  
You can find the full sample [here](https://github.com/FancyPixel/text-notification-sample). Once you run the app and schedule the notification, hop back to the home screen and you'll be able to reply with some text, that will be displayed in the app's view controller.    

Until next time.  
Andrea - [@theandreamazz](https://twitter.com/theandreamazz)
