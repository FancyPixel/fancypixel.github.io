---
layout: post
title: "Fun with iBeacon"
date: 2014-07-01 20:47
comments: true
categories: ios beacon rails 
---
You probably know already what [iBeacon](https://developer.apple.com/library/ios/documentation/userexperience/conceptual/LocationAwarenessPG/RegionMonitoring/RegionMonitoring.html) is, but just to reiterate, iBeacon is the Apple way of saying Bluetooth 4 Low Energy. At the cost of sounding like a mindless drone, by 'the Apple way of' I mean 'it just works and shows a lot of potential'. An iBeacon is a simple B4LE device that keeps broadcasting its presence. Other B4LE devices can sense when they reach the beacon without draining the battery (hence the LE) and making the user scream in agony. "Oook, what do I do with it?". The best thing you can do is locating a user without the GPS, which means locating a user inside a building. The cool thing is that it's fast, it takes seconds to detect a beacon and to react to its vicinity (or lack there of), and it works within the reach of Bluetooth technology (let's say around a 50 meters radius). I should also mention that it works fine with Android too. 

This week an [Estimote](http://estimote.com) developer kit arrived in our offices, so we took the chance to play around with it. 
<!-- More -->
I already tried my hand with iBeacons in the not so distant past. Using [BeaconEmitter](https://github.com/lgaches/BeaconEmitter) you can easily turn your Mac into a beacon, with no extra hardware required. When I experimented with iBeacons I had a couple of ideas on my mind, that involved being able to send a local notification to the user that enters in range of a device acting as a beacon. My dreams were crushed by the limits of the iOS 7.0 implementation, as I found out that: 

- you can't react when the user's screen is turned off 
- you can't perform any action when your app is in background, even if you request the `location` background state
- detecting when the user leaves a region takes quite a lot of time (at least 10/15 minutes)

The most exciting thing about playing around with the Estimote SDK, besides the nifty packaging and well designed piece of hardware, is that my devices now have iOS 7.1. It turns out that with version 7.1, iOS is way more flexible and it's taking care of all the problems I faced with 7.0:

- you can show a local notification when the screen is off
- you can perform operations when the user enters a region (even if the app was killed)
- it takes second to detect when the user is out of range

This turns everything around, iBeacons aren't just a gimmick now, but an exciting tool to experiment with. 

##Building a sample
First thing that came to our mind was to build a simple system to automatically check people in and out of the office. Really, as simple as it gets, it took a couple of hours to build, but it works surprisingly well. 

###Rails backend
To check people in and out we need a backend and an authentication system. Rails makes it easy, a model, a basic API and the help of Devise for the authentication process.

~~~ruby
# app/model/checkin.rb lang:ruby
class Checkin < ActiveRecord::Base

  belongs_to :user  
  enum direction: { in: 0, out: 1 }

end
~~~
That's a pretty basic model, taking advantage of Rails 4.1 enums. 

The routes are scoped as APIs, just to be fancy:

~~~ruby
# config/routes.rb
namespace :api, defaults: {format: :json} do
  namespace :v1 do
    post 'checkin', to: 'checkins#checkin'
    post 'checkout', to: 'checkins#checkout'
  end
end
~~~

And the controller does pretty much just this:

~~~ruby
# checkins_controller.rb
def checkin
  checkin = Checkin.new(user: current_user, direction: :in)
  if checkin.save
    head :no_content
  else
    render json: {errors: checkin.errors}, status: :unprocessable_entity
  end
end
~~~

The authentication is handled by Devise, and for simplicity we opted for HTTP Basic Authentication.

###iOS Client
The iOS app needs to look for our trusty beacon, and once the user is in range of our region, it needs to make a POST call to our API. When the user walks out of the office the phone needs to do the same to the checkout API. The iOS APIs for handling beacons are inside CoreLocation, in this sample I'll be using two main delegate methods: 

~~~objectivec
- (void)locationManager:(CLLocationManager *)manager didEnterRegion:(CLRegion *)region;
- (void)locationManager:(CLLocationManager *)manager didExitRegion:(CLRegion *)region;
~~~

The good guy calling these two methods is a `CLLocationManager` instance. The location manager needs an instance of `CLBeaconRegion` to start doing its magic though. We can define a region by specifying a UUDID, an identifier, a major and a minor. It might sound confusing at first, but all those things boil down to this: 

- `UUDID`: A unique identifier of our beacon network. It's best practice to have one UUDID per App. Each beacon will share the same UUDID.
- `identifier`: It's a string representation of our network. It usually is the reverse URI of our App, something along the line of com.something.awesome.
- `major`: It's an integer that specifies the major group of our beacons. Think of it as a common number that can identify a bunch of beacons inside a building.
- `minor`: It's an integer that specifies the single beacon inside of a major group.

So our basic config would be one UUDID and identifier per App, one major per building, and one minor per beacon. For the purposes of this sample we only have a beacon, so we can either disregard this info, or just specify whatever major and minor that we want, as long as it matches the ones configured in the beacon itself. 
Now that all that is out of the way, let's get to the code:

~~~objectivec
- (CLBeaconRegion *)region
{
    if (_region == nil) {
        NSUUID *proximityUUID = [[NSUUID alloc] initWithUUIDString:self.settings[@"udid"]];
        _region = [[CLBeaconRegion alloc] initWithProximityUUID:proximityUUID
                                                          major:[self.settings[@"major"] intValue]
                                                          minor:[self.settings[@"minor"] intValue]
                                                     identifier:self.settings[@"identifier"]];
        
        [_region setNotifyOnExit:YES];
        [_region setNotifyOnEntry:YES];
        [_region setNotifyEntryStateOnDisplay:YES];
    }
    return _region;
}
~~~
There we go, our lazy loaded region that reads the parameters from an NSDictionary. Cool, let's start monitoring:

~~~objectivec
[self.manager startMonitoringForRegion:self.region];
[self.manager stopRangingBeaconsInRegion:self.region];  
~~~

The first line is pretty self explanatory, the second one just tells the system that I don't really care for the single beacons, I just need the region updates. 

Now that we are monitoring the region, we just need to decide what to do when we are in and out of range:

~~~objectivec
- (void)locationManager:(CLLocationManager *)manager didEnterRegion:(CLRegion *)region
{
    if ([region isKindOfClass:[CLBeaconRegion class]]) {
        CLBeaconRegion *beaconRegion = (CLBeaconRegion *)region;
        if ([beaconRegion.identifier isEqualToString:self.settings[@"identifier"]] && [beaconRegion.major intValue] == [self.settings[@"major"] intValue] && [beaconRegion.minor intValue]== [self.settings[@"minor"] intValue]) {
            UILocalNotification *notification = [[UILocalNotification alloc] init];
            notification.userInfo = @{@"identifier": region.identifier};
            notification.alertBody = [NSString stringWithFormat:@"Entering %@", region.identifier];
            notification.soundName = @"Default";
            [[UIApplication sharedApplication] presentLocalNotificationNow:notification];
            [self remoteCheckin:FPCheckDirectionIn];
        }
    }
}

- (void)locationManager:(CLLocationManager *)manager didExitRegion:(CLRegion *)region
{
    if ([region isKindOfClass:[CLBeaconRegion class]]) {
        CLBeaconRegion *beaconRegion = (CLBeaconRegion *)region;
        if ([beaconRegion.identifier isEqualToString:self.settings[@"identifier"]] && [beaconRegion.major intValue] == [self.settings[@"major"] intValue] && [beaconRegion.minor intValue]== [self.settings[@"minor"] intValue]) {
            UILocalNotification *notification = [[UILocalNotification alloc] init];
            notification.userInfo = @{@"identifier": region.identifier};
            notification.alertBody = [NSString stringWithFormat:@"Exiting %@", region.identifier];
            notification.soundName = @"Default";
            [[UIApplication sharedApplication] presentLocalNotificationNow:notification];
            [self remoteCheckin:FPCheckDirectionOut];
        }
    }
}
~~~

As you can see, we're just checking against our beacon, when we are in range or when we get out of range, we push a local notification and we perform a remote call to our Rails backend. You can find the full source on our Github account, don't worry.

##Who's Fancy?
And there we go, the iOS app:

{% img center /images/posts/2014-07-01/iOS.png 'Who's Fancy iOS' %}

and the web page:

{% img center /images/posts/2014-07-01/rails.png 'Who's Fancy Rails' %}

You can find the rails and iOS code [here](https://github.com/FancyPixel/whosfancy-rails) and [here](https://github.com/FancyPixel/whosfancy-ios).  

####Android version
We also pushed the Android version on our Github page, you can find it [here](https://github.com/FancyPixel/whosfancy-android). 

Until next time. 

Andrea

