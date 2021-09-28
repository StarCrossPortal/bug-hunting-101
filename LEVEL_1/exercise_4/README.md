# Exercise 4

## CVE-2021-21204
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.

This time we do it by code audit

### Details

> What is happening is that the BlinkScrollbarPartAnimation instance
  passed to BlinkScrollbarPartAnimationTimer is released while
  the BlinkScrollbarPartAnimationTimer::TimerFired method runs as
  part of BlinkScrollbarPartAnimation::setCurrentProgress call,
  during the execution of ScrollbarPainter::setKnobAlpha which ends
  up calling BlinkScrollbarPainterDelegate::setUpAlphaAnimation
  through a chain of observers.
  BlinkScrollbarPainterDelegate::setUpAlphaAnimation releases the
  BlinkScrollbarPartAnimation instance which gets deallocated.
  BlinkScrollbarPartAnimation::setCurrentProgress continues execution
  after ScrollbarPainter::setKnobAlpha returns, but the _scrollbarPointer
  is overwritten with garbage and when SetNeedsPaintInvalidation
  is called the crash happens.

You'd better read [these](https://www.chromium.org/blink) to have a preliminary understanding of Blink and make sure you know a little about `Objective-C`

---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1189926

</details>

--------

### Set environment

Get source code
```sh
git clone https://chromium.googlesource.com/chromium/blink
cd blink
git reset --hard 6c3c857b90ef63822c8e598bdb7aea604ba1688c
```

### Related code
we can analysis the source file [online](https://chromium.googlesource.com/chromium/src/+/6c3c857b90ef63822c8e598bdb7aea604ba1688c/third_party/blink/renderer/core/scroll/mac_scrollbar_animator_impl.mm#414) or offline.

```objective-c
class BlinkScrollbarPartAnimationTimer {
 public:
  BlinkScrollbarPartAnimationTimer(
      BlinkScrollbarPartAnimation* animation,
      CFTimeInterval duration,
      scoped_refptr<base::SingleThreadTaskRunner> task_runner)
      : timer_(std::move(task_runner),
               this,
               &BlinkScrollbarPartAnimationTimer::TimerFired),
        start_time_(0.0),
        duration_(duration),
        animation_(animation),
        timing_function_(CubicBezierTimingFunction::Preset(
            CubicBezierTimingFunction::EaseType::EASE_IN_OUT)) {}
 private:
  void TimerFired(TimerBase*) {
    double current_time = base::Time::Now().ToDoubleT();
    double delta = current_time - start_time_;

    if (delta >= duration_)
      timer_.Stop();
    // This is a speculative fix for crbug.com/1183276.
    if (!animation_)
      return;

    double fraction = delta / duration_;
    fraction = clampTo(fraction, 0.0, 1.0);
    double progress = timing_function_->Evaluate(fraction);
    [animation_ setCurrentProgress:progress];
  }

  TaskRunnerTimer<BlinkScrollbarPartAnimationTimer> timer_;
  double start_time_;                       // In seconds.
  double duration_;                         // In seconds.
  BlinkScrollbarPartAnimation* animation_;  // Weak, owns this.
  scoped_refptr<CubicBezierTimingFunction> timing_function_;
};
```

```objective-c
- (void)setCurrentProgress:(NSAnimationProgress)progress {
  DCHECK(_scrollbar);
  CGFloat currentValue;
  if (_startValue > _endValue)
    currentValue = 1 - progress;
  else
    currentValue = progress;
  blink::ScrollbarPart invalidParts = blink::kNoPart;
  switch (_featureToAnimate) {
    case ThumbAlpha:
      [_scrollbarPainter setKnobAlpha:currentValue];  // call ScrollbarPainter::setKnobAlpha
      break;
    case TrackAlpha:
      [_scrollbarPainter setTrackAlpha:currentValue];
      invalidParts = static_cast<blink::ScrollbarPart>(~blink::kThumbPart);
      break;
    case UIStateTransition:
      [_scrollbarPainter setUiStateTransitionProgress:currentValue];
      invalidParts = blink::kAllParts;
      break;
    case ExpansionTransition:
      [_scrollbarPainter setExpansionTransitionProgress:currentValue];
      invalidParts = blink::kThumbPart;
      break;
  }
  _scrollbar->SetNeedsPaintInvalidation(invalidParts);
}
============================================================================
- (void)scrollerImp:(id)scrollerImp
    animateKnobAlphaTo:(CGFloat)newKnobAlpha
              duration:(NSTimeInterval)duration {
  if (!_scrollbar)
    return;

  DCHECK_EQ(scrollerImp, ScrollbarPainterForScrollbar(*_scrollbar));

  ScrollbarPainter scrollerPainter = (ScrollbarPainter)scrollerImp;
  [self setUpAlphaAnimation:_knobAlphaAnimation    // call BlinkScrollbarPainterDelegate::setUpAlphaAnimation
            scrollerPainter:scrollerPainter
                       part:blink::kThumbPart
             animateAlphaTo:newKnobAlpha
                   duration:duration];
}
============================================================================
- (void)setUpAlphaAnimation:
            (base::scoped_nsobject<BlinkScrollbarPartAnimation>&)
                scrollbarPartAnimation
            scrollerPainter:(ScrollbarPainter)scrollerPainter
                       part:(blink::ScrollbarPart)part
             animateAlphaTo:(CGFloat)newAlpha
                   duration:(NSTimeInterval)duration {
  blink::MacScrollbarAnimator* scrollbar_animator =
      _scrollbar->GetScrollableArea()->GetMacScrollbarAnimator();
  DCHECK(scrollbar_animator);
  // If the user has scrolled the page, then the scrollbars must be animated
  // here.
  // This overrides the early returns.
  bool mustAnimate = [self scrollAnimator].HaveScrolledSincePageLoad();
  if (scrollbar_animator->ScrollbarPaintTimerIsActive() && !mustAnimate)
    return;
  if (_scrollbar->GetScrollableArea()->ShouldSuspendScrollAnimations() &&
      !mustAnimate) {
    scrollbar_animator->StartScrollbarPaintTimer();
    return;
  }
  // At this point, we are definitely going to animate now, so stop the timer.
  scrollbar_animator->StopScrollbarPaintTimer();
  // If we are currently animating, stop
  if (scrollbarPartAnimation) {
    [scrollbarPartAnimation invalidate];
    scrollbarPartAnimation.reset();
  }
  scrollbarPartAnimation.reset([[BlinkScrollbarPartAnimation alloc]
      initWithScrollbar:_scrollbar
       featureToAnimate:part == blink::kThumbPart ? ThumbAlpha : TrackAlpha
            animateFrom:part == blink::kThumbPart ? [scrollerPainter knobAlpha]
                                                  : [scrollerPainter trackAlpha]
              animateTo:newAlpha
               duration:duration
             taskRunner:_taskRunner]);
  [scrollbarPartAnimation startAnimation];
}
```





### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  We can start from `TimerFired` func
  ```objective-c
  class BlinkScrollbarPartAnimationTimer [ ... ]
  void TimerFired(TimerBase*) {
    double current_time = base::Time::Now().ToDoubleT();
    double delta = current_time - start_time_;

    if (delta >= duration_)
      timer_.Stop();
    // This is a speculative fix for crbug.com/1183276.
    if (!animation_)
      return;

    double fraction = delta / duration_;
    fraction = clampTo(fraction, 0.0, 1.0);
    double progress = timing_function_->Evaluate(fraction);
    [animation_ setCurrentProgress:progress];  [1] call setCurrentProgress. Notice `animation_`
  }

  private:
    BlinkScrollbarPartAnimation* animation_;  [2] weak, own this
  ```
  [2] `BlinkScrollbarPartAnimationTimer` own the instance of `BlinkScrollbarPartAnimation` which assignmented by constructor. This make a chance to trigger uaf.

  ```objective-c
  - (void)setCurrentProgress:(NSAnimationProgress)progress {
    DCHECK(_scrollbar);
    CGFloat currentValue;
    blink::ScrollbarPart invalidParts = blink::kNoPart;
    switch (_featureToAnimate) {
      case ThumbAlpha:
        [_scrollbarPainter setKnobAlpha:currentValue];  [3] call ScrollbarPainter::setKnobAlpha
        break;
      case TrackAlpha:
        [_scrollbarPainter setTrackAlpha:currentValue];
        invalidParts = static_cast<blink::ScrollbarPart>(~blink::kThumbPart);
        break;
    }
    _scrollbar->SetNeedsPaintInvalidation(invalidParts);
  }
  ```
  `setCurrentProgress` can call `setKnobAlpha`
  ```objective-c
  - (void)scrollerImp:(id)scrollerImp
      animateKnobAlphaTo:(CGFloat)newKnobAlpha
                duration:(NSTimeInterval)duration {
    if (!_scrollbar)
      return;

    DCHECK_EQ(scrollerImp, ScrollbarPainterForScrollbar(*_scrollbar));

    ScrollbarPainter scrollerPainter = (ScrollbarPainter)scrollerImp;
    [self setUpAlphaAnimation:_knobAlphaAnimation    [4] call BlinkScrollbarPainterDelegate::setUpAlphaAnimation
              scrollerPainter:scrollerPainter
                        part:blink::kThumbPart
              animateAlphaTo:newKnobAlpha
                    duration:duration];
  }
  ```
  ```objective-c
  - (void)setUpAlphaAnimation:
              (base::scoped_nsobject<BlinkScrollbarPartAnimation>&)
                  scrollbarPartAnimation
              scrollerPainter:(ScrollbarPainter)scrollerPainter
                        part:(blink::ScrollbarPart)part
              animateAlphaTo:(CGFloat)newAlpha
                    duration:(NSTimeInterval)duration {
    blink::MacScrollbarAnimator* scrollbar_animator =
        _scrollbar->GetScrollableArea()->GetMacScrollbarAnimator();
    DCHECK(scrollbar_animator);

    // If we are currently animating, stop
    if (scrollbarPartAnimation) {                                       [5]
      [scrollbarPartAnimation invalidate];
      scrollbarPartAnimation.reset();
    }
    scrollbarPartAnimation.reset([[BlinkScrollbarPartAnimation alloc]   [6]
        initWithScrollbar:_scrollbar
        featureToAnimate:part == blink::kThumbPart ? ThumbAlpha : TrackAlpha
              animateFrom:part == blink::kThumbPart ? [scrollerPainter knobAlpha]
                                                    : [scrollerPainter trackAlpha]
                animateTo:newAlpha
                duration:duration
              taskRunner:_taskRunner]);
    [scrollbarPartAnimation startAnimation];
  }
  ```
  The `(base::scoped_nsobject<BlinkScrollbarPartAnimation>&) scrollbarPartAnimation` can be release by `reset()`

  - About `scoped_nsobject`:
    >`scoped_nsobject<>` is patterned after std::unique_ptr<>, but maintains
    ownership of an NSObject subclass object.  Style deviations here are solely
    for compatibility with std::unique_ptr<>'s interface, with which everyone is
    already familiar.

    >scoped_nsobject<> takes ownership of an object (in the constructor or in
    reset()) by taking over the caller's existing ownership claim.  The caller
    must own the object it gives to scoped_nsobject<>, and relinquishes an
    ownership claim to that object.  **scoped_nsobject<> does not call -retain,
    callers have to call this manually if appropriate.**
  - About `void base::scoped_nsprotocol< NST >::reset( NST object = nil )`:
    >
    ```objective-c                            {
      // We intentionally do not check that object != object_ as the caller must
      // either already have an ownership claim over whatever it passes to this
      // method, or call it with the |RETAIN| policy which will have ensured that
      // the object is retained once more when reaching this point.
      [object_ release];
      object_ = object;
      }
    ```


</details>

--------

