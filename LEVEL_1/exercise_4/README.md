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

You'd better read [these](https://www.chromium.org/blink) to have a preliminary understanding of Blink
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

```mm
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
  

</details>

--------

