---
layout: post
title:  "Please stop displaying \"No Internet\" Alert Views!"
---
One of the things you still see a lot in iOS apps is the "No Internet" popups displayed whenever an outgoing request fails because network is not reachable anymore. This is poor UX!

Reachability is a global matter that should be handled asynchronously. When network is not reachable your app should go in a limited mode, avoiding any request that will collapse even before it is started. In the mean time users should be warned that Internet is not available anymore.

When working with [AFNetworking](https://github.com/AFNetworking/AFNetworking) just set the reachability status change block when you instantiate your API Client:

{% highlight Objective-C %}
[[AFNetworkReachabilityManager sharedManager] setReachabilityStatusChangeBlock:^(AFNetworkReachabilityStatus status) {
            NSLog(@"Reachability: %@", AFStringFromNetworkReachabilityStatus(status));
            dispatch_async(dispatch_get_main_queue(), ^{
                [[NSNotificationCenter defaultCenter] postNotificationName:SKYReachabilityChangedNotification object:nil userInfo:@{@"status": @(status)}];
            });
}];
{% endhighlight %}

I like to work with notifications when reachability is concerned because all my view controllers can be observers for this notification and respond to it with the proper UI in their own context.

The observers of this SKYReachabilityChangedNotification are responsible for toggling on and off the "No Internet" UI based on the reachability changes (Alert Views are Evil, please don't use them).
These same observers can also disable any send button, display placeholders if needed, handle offline data, stop further outgoing requests or do anything that you need to do whenever Internet is not available.

![Some UI](http://planning.omts.fr/SugarFree/nointernetconnection.png){: .center-image }

Now that your app is set for responding to reachability changes with a more subtle UI the only thing left is to silence errors of any pending requests that started before network has dropped. Remember that these are the only requests that need to be taking care of because your view controllers won't allow any other request to start if Internet is no longer available.

Again I like to let the view controllers decide what is the right way of displaying an applicative error and the only thing that should be centralized is to silence the "No Internet" error on a per request basis.
It can be done using a class method (defined in whatever utility class you want) available to all your controllers. Call this method on all your request failure completions and pass in the error you get and a display block.

{% highlight Objective-C %}
+ (void)handleError:(NSError *)error withDisplayBlock:(void (^)())displayBlock {
    if ([error.domain isEqualToString:NSURLErrorDomain] && error.code == NSURLErrorNotConnectedToInternet) {
        NSLog(@"Internet availability error => Silenced");
     }
     else {
         if (displayBlock) {
             //Always run the display block in the UI queue
             dispatch_async(dispatch_get_main_queue(), ^{
                 displayBlock();
             });
         }
     }
}
{% endhighlight %}

This method analyzes the error and if it's not a reachability error executes whatever display block is passed in. Otherwise it does nothing and you'll have to trust the asynchronous mechanism you put in place earlier to display Internet reachability errors to the users.

{% include twitter_plug.html %}
