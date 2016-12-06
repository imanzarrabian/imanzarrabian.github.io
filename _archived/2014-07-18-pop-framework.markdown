---
layout: post
title:  "Pop framework: A practical use case"
---
It's been a while since I've been thinking about writing some tech articles again. The first blog was such an adventure and I didn't have the shoulders to handle it. Lack of time, lack of knowledge definitely.
It's been 3 or 4 years now and I have even less time today, I think I'm ready though. This might seem very paradoxal, it's not to me.

Anyway I decided to create this blog and write again when I started to use [**pop framework**](https://github.com/facebook/pop) for iOS. Almost everything I read about the new Facebook opensource framework was great but what an everyday iOS developer really needs is to experiment, right? So I did that.

Even better I've decided to include it and use it as a partial replacement for Apple's Core Animation in one of my [**Company's**](http://www.onemorethingstudio.com) project : [**Cheers!**](https://itunes.apple.com/app/cheers-selfie-party/id779619406?mt=8)

This post is about a practial case study of how I used pop and other cool animation stuffs to create some part of Cheers V2 UI/UX (Note that Cheers V2 is not live yet, we're working on it).

Basically the idea of all the animations behind Cheers V2 is to have something very organic and fluid. [**Paper**](https://itunes.apple.com/us/app/paper-stories-from-facebook/id794163692?mt=8) was a great source of inspiration but we don't want to be as interactive as paper is. So we'll focus on one screen here, the login screen.

Our animations can be decomposed into two categories: the launch animations and the interactive animations.


<img src="http://data.bloggif.com/distant/user/store/6/e/d/6/a73b8ed6f127a5b726a8a2cfe0796de6.gif" alt="Mounting created Bloggif" />{: .center-image }

The launch animations are the background Image View getting "squizzed" and the bottom view getting in the screen with a nice bouncy effect without any user interaction. I've added to that some [Shimmering](https://github.com/facebook/Shimmer) effect as well on the "Wanna find out?" label.

For this post I'll work on a demo project (available on [**github**](https://github.com/OMTS/CocoaSugarFree))

From now on I consider that you have some basic knowledge of iOS Dev.

### Launch Animation
![Minion](http://data.bloggif.com/distant/user/store/8/2/f/c/1653782d567ff619ad8bb4f3a32fcf28.gif){: .center-image }

The idea here is to start off with an image bigger than the screen and resize it after the view is displayed. Sounds easy enough. I use pop framework to make the image view small and reposition it on the top of the screen.

{% highlight Objective-C %}
//1 - Create animations
POPSpringAnimation *imageViewBoundsAnimation = [POPSpringAnimation animation];
POPSpringAnimation *imageViewPositionAnimation = [POPSpringAnimation animation];

//2 - Set the animatable properties
imageViewBoundsAnimation.property = [POPAnimatableProperty propertyWithName:kPOPLayerBounds];
imageViewPositionAnimation.property = [POPAnimatableProperty propertyWithName:kPOPLayerPositionY];

//3 - set the destination values
imageViewBoundsAnimation.toValue = [NSValue valueWithCGRect:CGRectMake(0, 0, self.view.bounds.size.width, self.view.bounds.size.height)];
imageViewPositionAnimation.toValue = @(self.view.center.y);

//4 - set the animaions configurations values
imageViewBoundsAnimation.springBounciness = 2;
imageViewBoundsAnimation.springSpeed = 15;
imageViewPositionAnimation.springBounciness = 3;
imageViewPositionAnimation.springSpeed = 12;

//5 - Finally start the animation
[self.bgImageView pop_addAnimation:imageViewBoundsAnimation forKey:@"bounds"];
[self.bgImageView pop_addAnimation:imageViewPositionAnimation forKey:@"position"];

{% endhighlight %}

That's it. We create our animations, select the right property to animate within the pop framework, set the destination values and configure the animation behaviour. The final two lines of code kick off the animations

Note that in this project I'm not using Autolayout but feel free to animate NSLayoutConstraints instead of bounds and position. NSLayoutConstraints are animatable properies that you can select as a parameter of the **propertyWithName:** method

 I'm also using different values of bounciness for each of these animations making things fall in place with a little cool "imperfect random organic parallax effect". Feel free to play with thoses speed and bounciness values to find the right feeling.

We still have to move our bottom view making it visible to users. I'm doing that by adding the following code to the previous animation method.

{% highlight Objective-C %}
POPSpringAnimation *interactionViewBottomAnimation = [POPSpringAnimation animation];

//Set the animatable properties
interactionViewBottomAnimation.property = [POPAnimatableProperty propertyWithName:kPOPLayerPositionY];

//set the destination values
interactionViewBottomAnimation.toValue = @(self.view.bounds.size.height - self.interactionView.bounds.size.height/2.0);

//set the animaions configurations valuessugar
interactionViewBottomAnimation.springBounciness = 3;
interactionViewBottomAnimation.springSpeed = 12;

//Finally set the animation
[self.interactionView pop_addAnimation:interactionViewBottomAnimation forKey:@"interactionViewAnimation"];
{% endhighlight %}



### Interaction Animation

<img src="http://data.bloggif.com/distant/user/store/0/0/d/c/1c742d585024bd13b745026a8803cd00.gif" alt="Mounting created Bloggif"/>{: .center-image }

Let's deal with the bounce effect of the bottom view first. There are tons of way to accomplish that. You can code everything by hand, or use a scroll view which will give it to you for free.
In my case I decided to use UIKit Dynamics and a UISnapBehaviour both introduced in iOS7 because I want to have easy control on the bounce effect and on the actual movement of the bottom view. And because I think UIKit Dynamics are cool :)

First we have to add a pan gesture to the bottom view:

{% highlight Objective-C %}

UIPanGestureRecognizer *gestureRecognizer = [[UIPanGestureRecognizer alloc] initWithTarget:self action:@selector(handlePanGesture:)];
    [self.interactionView addGestureRecognizer:gestureRecognizer];  
{% endhighlight %}

and deal with the gesture when it is triggered by the user:
{% highlight Objective-C %}

- (void)handlePanGesture:(UIPanGestureRecognizer *)gestureRecognizer {

    if (gestureRecognizer.state == UIGestureRecognizerStateBegan) {
        // We'll fill that later
    }
    else if (gestureRecognizer.state == UIGestureRecognizerStateChanged) {

        CGPoint newCenter = self.interactionView.center;

        //This is why I choosed to implement this with UIKit Dynamics
        newCenter.y += [gestureRecognizer translationInView:self.view].y/2.5;

        //Zooming the top imageview in or out
        //You can set some max zooming in or out here to avoid too much scaling
        self.bgImageView.bounds = CGRectMake(0, 0, self.bgImageView.bounds.size.width* (newCenter.y/self.interactionView.center.y), self.bgImageView.bounds.size.height* (newCenter.y/self.interactionView.center.y));

        //Actually moving the view
        self.interactionView.center = newCenter;
        [gestureRecognizer setTranslation:CGPointZero inView:self.view];

    }
    else if (gestureRecognizer.state == UIGestureRecognizerStateEnded) {
        //We'll fill that later
    }
}
{% endhighlight %}

The important thing here is that as long as the user moves his finger on the bottom view we just move it's center and we change the background image bounds.

You can of course tweak the code here to set some boundaries to the zooming.

Ok, let's add the actual bounce effect to resist the movement. This initialization should be done on the screen appearance for example.

{% highlight Objective-C %}
//Creating the animator and the Snap Behaviour
self.animator = [[UIDynamicAnimator alloc] initWithReferenceView:self.view];
self.snapBehaviour = [[UISnapBehavior alloc] initWithItem:self.interactionView snapToPoint:CGPointMake(self.view.bounds.size.width/2.0,self.view.bounds.size.height - self.interactionView.bounds.size.height/2.0)];
self.snapBehaviour.damping = .7;
{% endhighlight %}

Note that the anchor point for our UISnapBehaviour is the center of our bottom view in the superview coordinates system.

One thing left is to add this behaviour to our UIDynamicAnimator and we're doing that as soon as our user lets the view loose (aka as the UIGestureRecognizerStateEnded Gesture state).
This is the new implementation of our previous handlePanGesture: method

{% highlight Objective-C %}
- (void)handlePanGesture:(UIPanGestureRecognizer *)gestureRecognizer {

    if (gestureRecognizer.state == UIGestureRecognizerStateBegan) {
        // Let's avoid any interference here
        [self.animator removeAllBehaviors];
    }
    else if (gestureRecognizer.state == UIGestureRecognizerStateChanged) {

        CGPoint newCenter = self.interactionView.center;

        //This is why I choosed to implement this with UIKit Dynamics
        newCenter.y += [gestureRecognizer translationInView:self.view].y/2.5;

        //Zooming the top imageview in or out
        //You can set some max zooming in or out here to avoid too much scaling
        self.bgImageView.bounds = CGRectMake(0, 0, self.bgImageView.bounds.size.width* (newCenter.y/self.interactionView.center.y), self.bgImageView.bounds.size.height* (newCenter.y/self.interactionView.center.y));

        //Actually moving the view
        self.interactionView.center = newCenter;
        [gestureRecognizer setTranslation:CGPointZero inView:self.view];

    }
    else if (gestureRecognizer.state == UIGestureRecognizerStateEnded) {
    	//Add the Behaviour to the animator
         [self.animator addBehavior:self.snapBehaviour];

        UIDynamicItemBehavior *itemBehaviour = [[UIDynamicItemBehavior alloc] initWithItems:@[self.interactionView]];
        itemBehaviour.allowsRotation = NO;
        [self.animator addBehavior:itemBehaviour];

        //This will simply set the image view animate back to it's original size
        [self animateBackToTheRightTopImageSize];
    }
}
{% endhighlight %}

Notice the allowsRotation property set to NO on the itemBehaviour. You can comment that line out and see that the bottom view will come back to it's position in a funky way. I think this a little too much and for Cheers I've decided to keep it simple (at least for this specific animation).

Finally the only thing left is to implement this animateBackToTheRightTopImageSize method where pop framework is used again to set the background image's bounds back to it's original value.

{% highlight Objective-C %}
- (void)animateBackToTheRightTopImageSize {
    POPSpringAnimation *sizeAnimation = [POPSpringAnimation animation];

    sizeAnimation.property = [POPAnimatableProperty propertyWithName:kPOPLayerBounds];

    sizeAnimation.toValue = [NSValue valueWithCGRect:CGRectMake(0, 0, self.view.bounds.size.width, self.view.bounds.size.height)];

    sizeAnimation.springBounciness = 4;
    sizeAnimation.springSpeed = 20;

    [self.bgImageView pop_addAnimation:sizeAnimation forKey:@"sizeBack"];
}
{% endhighlight %}

A little something remains as an excercise here. Did you notice that we use a constant speed for the "back to it's place" animation? pop crew's advice is to have some continuity in the animations. Here continuity would be for example bypassing the snap behaviour and get the actual speed of the bottom view when the user lift his fingers and pass it as the initial velocity for some decay animation (another animation type shipped with pop) that moves our bottom view back to it's original position from the current view position. There's a little something here. Please feel free to experiment on that.

That's it for this small use case study.

Reminder : [**Source Code**](https://github.com/OMTS/CocoaSugarFree)

{% include twitter_plug.html %}
