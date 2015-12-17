---
layout: post
title:  "Custom Curve UIView Animations"
date:   2014-12-09 16:22:18 +0100
categories: posts
---
I generally favour using the block based view animation methods defined on `UIView` over Core Animation for several reasons. Most importantly, I think that the syntax of these methods is more concise than first defining an animation object instance, then setting all properties and finally adding the animation object instance to the layer of the view that we want to animate.

However, there are some situations where the API exposed by these methods is not sufficient, such as the desire for an animation curve other than the predefined `UIViewAnimationOption` curves. Luckily, Apple also provides a `CADisplayLink` class that allows us to synchronise the execution of code with the refresh rate of the screen.

In this post, I’d like to share a convenience method that I created recently in a category on `UIView`[^1]. With this convenience method, a property of a view can be animated with a custom animation curve. I will first discuss the building blocks of the convenience method and then give an example application.

## Building blocks

To make the convenience method work, there are several building blocks that need to be implemented. First of all, we need to find a way to encode the desired animation curve, and retrieve its value at a specific point in time. While Apple's `CAMediaTimingFunction` class allows us to do the first, it offers no method to evaluate the function described by the curve for a certain input value. Therefore, for my implementation I resorted to [GNUstep QuartzCore's CAMediaTimingFunction class](https://github.com/gnustep/gnustep-quartzcore/blob/master/Source/CAMediaTimingFunction.m), which clones Apple's `CAMediaTimingFunction` implementation. To avoid naming conflicts, I renamed this class to `GSQMediaTimingFunction` and extended its public interface to expose the method `- (CGFloat)evaluateYAtX:(CGFloat)x`.

The second building block, `LLTimedAnimation`, sets up and manages a display-link based animation. The class is initialised as follows:

{% highlight objc %}
- (instancetype)initWithMediaTimingFunction:(GSQMediaTimingFunction *)timingFunction duration:(CGFloat)duration animationStateForProgress:(void(^)(CGFloat progress))animationStateForProgress completion:(void (^)(void))completion
{
    self = [super init];

    if (self) {
        self.duration = duration;
        self.timingFunction = timingFunction;
        self.animationStateForProgress = animationStateForProgress;
        self.completion = completion;
    }
    return self;
}
{% endhighlight %}

After initialisation, our `LLTimedAnimation` instance is set up with the duration of the animation, the timing function to use to calculate the progress, a completion block and, most importantly, a block `animationStateForProgress` which will be called whenever the `CADisplayLink` fires. As a parameter, this block takes the animation's progress normalised to a value in the range `{0,1}`. Note that at any intermediary point in time, the actual value may lay outside this range if some of the parameters passed to `GSQMediaTimingFunction`'s `+ (id) functionWithControlPoints:` are smaller than 0 or greater than 1. However, as per [Apple's documentation](file:///Users/Lars/Library/Developer/Shared/Documentation/DocSets/com.apple.adc.documentation.AppleiOS7.0.iOSLibrary.docset/Contents/Resources/Documents/documentation/Cocoa/Reference/CAMediaTimingFunction_class/Introduction/Introduction.html#//apple_ref/occ/clm/CAMediaTimingFunction/functionWithControlPoints::::), the start and end values are guaranteed to be 0 and 1 respectively.

The `LLTimedAnimation` can be started by calling the `begin` method on an instance:

{% highlight objc %}
- (void)begin
{
    CADisplayLink *displayLink = [CADisplayLink displayLinkWithTarget:self selector:@selector(performAnimationFrame:)];
    [displayLink addToRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
}
{% endhighlight %}

Upon calling this method a `CADisplayLink` is set up with the `LLTimedAnimation` instance as target; ensuring that whenever the display link fires, the method `performAnimationFrame:` will be called with the display link as parameter. The display link is then addeed to the main run loop. Since the display link retains its target, the `LLTimedAnimation` instance will be kept in memory as long as the display link is attached to the run loop.

The final building block is the `performAnimationFrame:` method. The task of this method is straightforward; it uses the timestamp of the display link to calculate the progress of the animation with respect to its total duration and then calls `evaluateYAtX:` on the media timing function to get the actual progress value according to the curve defined earlier. If the time spent within the animation is greater than or equal to the allotted duration, the progress is set to 1.0 to ensure a clean end state. This is required since this method may get called sligthly later than the actual end time due to the nature of a display link.

The block stored in `self.animationStateForProgress` upon initialization is then called with the calculated progress value as a parameter. Finally, if the defined animation duration has finished, and therefore the animation has completed, we remove the display link from the run loop and call the completion block. Note that by removing the display link from the run loop, the class cluster consisting of the `CADisplayLink` and `LLTimedAnimation` instance is cleaned up.

{% highlight objc %}
- (void)performAnimationFrame:(CADisplayLink *)sender
{
    CGFloat durationDone = 0;
    CGFloat progress;

    if (sender) {
        if (self.beginTime && self.beginTime != 0) {
            durationDone = (sender.timestamp - self.beginTime);
        } else {
            self.beginTime = sender.timestamp;
        }

        if (self.duration > 0) {
            progress = [self.timingFunction evaluateYAtX:durationDone/self.duration];
        } else {
            progress = 1.0;
        }
    } else {
        progress = 1.0;
    }

    if (durationDone >= self.duration) {
        // Make sure the end state is 'clean'
        progress = MAX(progress, 1.0);
    }

    self.animationStateForProgress(progress);

    if (durationDone >= self.duration) {
        [sender removeFromRunLoop:[NSRunLoop mainRunLoop] forMode:NSRunLoopCommonModes];
        if (self.completion) {
            self.completion();
        }
    }
}
{% endhighlight %}

## Example application

As an example application, consider a label that may contain a numeric value only. Whenever we set a new value for the label, we do not want the label to update instantly, but rather want it to count toward it's final value starting from it's current value. Using our category as defined above, we can easily achieve such an effect.

{% include image.html file="LLCountingLabel.gif" alt="Example" %}

First of all, we create a class `LLCountingLabel` which subclasses `UILabel`. The label defines a property `currentValue` that is initialized to have the value 0:

{% highlight objc %}
- (instancetype)init
{
    self = [super init];
    if (self) {
        _currentValue = 0;
    }
    return self;
}
{% endhighlight %}

Furthermore, we define a method `- (void)setCounter:(NSUInteger)newCounterValue` in the public interface of our class, and in its implementation, we calculate the difference between the current value of the label and it's new value. We then call our convenience method on `UIView` and in the `animationStateForProgress` callback block, we calculate the value of the label relative to the progress of the animation and set this value as the text of the label.

{% highlight objc %}
- (void)setCounter:(NSInteger)newCounterValue
{
    NSInteger valueAtStart = self.currentValue;
    NSInteger diff = newCounterValue - valueAtStart;

    @weakify(self);
    [UIView animateWithMediaTimingFunction:[GSQMediaTimingFunction functionWithName:kCAMediaTimingFunctionEaseInEaseOut] duration:1.0f animationStateForProgress:^(CGFloat progress) {

        @strongify(self);
        if (! self) {
            return;
        }

        NSInteger valueForProgress = valueAtStart + (diff * progress);
        self.currentValue = valueForProgress;
        [self setText:[NSString stringWithFormat:@"%@", @(valueForProgress)]];

    } completion:nil];
}
{% endhighlight %}

Since the `animationStateForProgress` callback block is called whenever the screen refreshes, the value of the label is updated for every frame that is rendered. The result is as shown above. The code for the example is available on [Github](https://github.com/larslockefeer/CountingLabel).

[^1]:
    I started writing this post in February, but never got to finish it up until now. Meanwhile, Facebook released their awesome POP Animation Framework, which contains a `POPCustomAnimation` class that is very similar to what I'll describe in this post. I still think, though, that this post is a nice illustration of how to simplify animation code. -->
