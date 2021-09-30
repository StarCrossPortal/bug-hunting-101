# Exercise 6

## CVE-2021-21188
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.

This time we do it by code audit

### Details

> Test for persistent execution context during Animatable::animate.
> 
> Prior to the patch, the validity of the execution context was only
> checked on entry to the method; however, the execution context can
> be invalidated during the course of parsing keyframes or options.
> The parsing of options is upstream of Animatable::animate and caught by
> the existing check, but invalidation during keyframe parsing could fall
> through triggering a crash.


You can read [this](https://chromium.googlesource.com/chromium/src/+/af77c20371d1418300cefbc5fa6779067b7792cf/third_party/blink/renderer/core/animation/#core_animation) to know what about `animation`



---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1161739

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset -hard 710bae69e18a9b086795cf79d849bd7f6e9c97fa
```

### Related code
[`third_party/blink/renderer/core/animation/animatable.cc`](https://chromium.googlesource.com/chromium/src/+/db032cf0a96b0e7e1007f181d8ce21e39617cee7/third_party/blink/renderer/core/animation/animatable.cc) and others



### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>
  
  Let's start analysis the source code
  ``` c++
  Animation* Animatable::animate(
      ScriptState* script_state,
      const ScriptValue& keyframes,
      const UnrestrictedDoubleOrKeyframeAnimationOptions& options,
      ExceptionState& exception_state) {
    if (!script_state->ContextIsValid())
        return nullptr;
    Element* element = GetAnimationTarget();
    if (!element->GetExecutionContext())        [1] call `GetExecutionContext` to check whether the validity of this ptr 
        return nullptr;
    KeyframeEffect* effect =
        KeyframeEffect::Create(script_state, element, keyframes,    [2] call create and element is the second parameter
                                CoerceEffectOptions(options), exception_state);
    if (exception_state.HadException())
        return nullptr;

    ReportFeaturePolicyViolationsIfNecessary(*element->GetExecutionContext(),
                                            *effect->Model());
    if (!options.IsKeyframeAnimationOptions())
      return element->GetDocument().Timeline().Play(effect);

    Animation* animation;
    const KeyframeAnimationOptions* options_dict =
        options.GetAsKeyframeAnimationOptions();
    if (!options_dict->hasTimeline()) {
      animation = element->GetDocument().Timeline().Play(effect);
    } else if (AnimationTimeline* timeline = options_dict->timeline()) {
      animation = timeline->Play(effect);
    } else {
    animation = Animation::Create(element->GetExecutionContext(), effect, [3] If we delete `element` in [2], this trigger crash 
                                    nullptr, exception_state);
    }
    [ ... ]
  ```
  What happen in Create
  ```c++
  KeyframeEffect* KeyframeEffect::Create(
    ScriptState* script_state,
    Element* element,
    const ScriptValue& keyframes,
    const UnrestrictedDoubleOrKeyframeEffectOptions& options,
    ExceptionState& exception_state) {
    Document* document = element ? &element->GetDocument() : nullptr;
    Timing timing = TimingInput::Convert(options, document, exception_state);
    if (exception_state.HadException())
      return nullptr;

    EffectModel::CompositeOperation composite = EffectModel::kCompositeReplace;
    String pseudo = String();
    // [ ... ]
    KeyframeEffectModelBase* model = EffectInput::Convert(      [4] call Convert, and element is the first parameter
        element, keyframes, composite, script_state, exception_state);
    if (exception_state.HadException())
      return nullptr;
    KeyframeEffect* effect =
        MakeGarbageCollected<KeyframeEffect>(element, model, timing);

    if (!pseudo.IsEmpty()) {
      effect->target_pseudo_ = pseudo;
      if (element) {
        element->GetDocument().UpdateStyleAndLayoutTreeForNode(element);
        effect->effect_target_ = element->GetPseudoElement(
            CSSSelector::ParsePseudoId(pseudo, element));
      }
    }
    return effect;
  }
  ================================================================================
  KeyframeEffectModelBase* EffectInput::Convert(
    Element* element,
    const ScriptValue& keyframes,
    EffectModel::CompositeOperation composite,
    ScriptState* script_state,
    ExceptionState& exception_state) {
  StringKeyframeVector parsed_keyframes =
      ParseKeyframesArgument(element, keyframes, script_state, exception_state);  [5] call ParseKeyframesArgument and element is the first parameter
  if (exception_state.HadException())
    return nullptr;
  [ ... ]
  ```
  I wander what is Keyframe? Then I found [this](https://developer.mozilla.org/en-US/docs/Web/CSS/@keyframes) and [this](https://developer.mozilla.org/en-US/docs/Web/CSS/animation). Although I posted the animation link last time, I didn’t read it myself... But this time we must know what is animation and keyframe.
  > The animation property is specified as one or more single animations, separated by commas.
  I don't know what the word animation mean, so I don't know what it mean for chrome :/ Alright, you can know them detailed from the two link or you can search yourself.

  What can we do to delete this `element` during  `ParseKeyframesArgument`? We can see its def and how animation be constructed.
  ```c++
  StringKeyframeVector EffectInput::ParseKeyframesArgument(
      Element* element,
      const ScriptValue& keyframes,
      ScriptState* script_state,
      ExceptionState& exception_state) {
    // Per the spec, a null keyframes object maps to a valid but empty sequence.
    v8::Local<v8::Value> keyframes_value = keyframes.V8Value();
    if (keyframes_value->IsNullOrUndefined())
      return {};
    v8::Local<v8::Object> keyframes_obj = keyframes_value.As<v8::Object>();

    // 3. Let method be the result of GetMethod(object, @@iterator).
    v8::Isolate* isolate = script_state->GetIsolate();
    auto script_iterator =
        ScriptIterator::FromIterable(isolate, keyframes_obj, exception_state);
    if (exception_state.HadException())
      return {};

    // TODO(crbug.com/816934): Get spec to specify what parsing context to use.
    Document& document = element
                            ? element->GetDocument()
                            : *LocalDOMWindow::From(script_state)->document();

    StringKeyframeVector parsed_keyframes;
    if (script_iterator.IsNull()) {
      parsed_keyframes = ConvertObjectForm(element, document, keyframes_obj,
                                          script_state, exception_state);
    } else {
      parsed_keyframes =
          ConvertArrayForm(element, document, std::move(script_iterator),   [6]  if keyframes is sorted by array, do convert
                          script_state, exception_state);
    }
  [ ... ]
  ```
  Parse the parameter of animatable (parse keyframes), If we transform an Array composed of keyframs, need call ConvertArrayForm for convert step.
  ```c++
StringKeyframeVector ConvertArrayForm(Element* element,
                                      Document& document,
                                      ScriptIterator iterator,
                                      ScriptState* script_state,
                                      ExceptionState& exception_state) {
  v8::Isolate* isolate = script_state->GetIsolate();

  // This loop captures step 5 of the procedure to process a keyframes argument,
  // in the case where the argument is iterable.
  HeapVector<Member<const BaseKeyframe>> processed_base_keyframes;
  Vector<Vector<std::pair<String, String>>> processed_properties;
  ExecutionContext* execution_context = ExecutionContext::From(script_state);
  while (iterator.Next(execution_context, exception_state)) {
    if (exception_state.HadException())
      return {};

    // The value should already be non-empty, as guaranteed by the call to Next
    // and the exception_state check above.
    v8::Local<v8::Value> keyframe = iterator.GetValue().ToLocalChecked();

    BaseKeyframe* base_keyframe = NativeValueTraits<BaseKeyframe>::NativeValue(
        isolate, keyframe, exception_state);
    Vector<std::pair<String, String>> property_value_pairs;

    if (!keyframe->IsNullOrUndefined()) {
      AddPropertyValuePairsForKeyframe(                           [7]   call AddPropertyValuePairsForKeyframe
          isolate, v8::Local<v8::Object>::Cast(keyframe), element, document,
          property_value_pairs, exception_state);
      if (exception_state.HadException())
        return {};
    }
  [ ... ]
===========================================================================
  void AddPropertyValuePairsForKeyframe(
      v8::Isolate* isolate,
      v8::Local<v8::Object> keyframe_obj,
      Element* element,
      const Document& document,
      Vector<std::pair<String, String>>& property_value_pairs,
      ExceptionState& exception_state) {
    Vector<String> keyframe_properties =
        GetOwnPropertyNames(isolate, keyframe_obj, exception_state);

      // By spec, we are only allowed to access a given (property, value) pair
      // once. This is observable by the web client, so we take care to adhere
      // to that.
      v8::Local<v8::Value> v8_value;
      if (!keyframe_obj
              ->Get(isolate->GetCurrentContext(), V8String(isolate, property))    [8] call get
              .ToLocal(&v8_value)) {
        exception_state.RethrowV8Exception(try_catch.Exception());
        return;
      }
  }
  ```
  We can delete this `element` in `getter`, these `element` and `document` is the target of `html`, if you can get if you have learned how js control DOM of html.
  We use `element.animate(keyframes, options);` to trigger, we can make a `arr` whose `getter` delete `element_1`. and do `element_2.animate(arr,{...})`, you can see detail at [Poc](./poc.html).



</details>

--------
