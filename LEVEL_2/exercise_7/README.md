# Exercise 7

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2020-6422
I sugget you don't search any report about it to prevents get too much info like patch.


### Details

In level 2, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1166091

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset --hard a8b9044e5a317034dca14763906aed6fa743ab58
```


### Related code

third_party/blink/renderer/modules/webgl/webgl_rendering_context_base.cc

tips: Not all delete operation set `var` null. In some cases we need save the destoried var for next step.

The bug is in the last quarter of the source code.

### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>

  This cve describes a type of vulnerability for us.
  ```c++
void WebGLRenderingContextBase::PrintWarningToConsole(const String& message) {
  blink::ExecutionContext* context = Host()->GetTopExecutionContext();
  if (context) {                                                 [1]
    context->AddConsoleMessage(MakeGarbageCollected<ConsoleMessage>(
        mojom::ConsoleMessageSource::kRendering,
        mojom::ConsoleMessageLevel::kWarning, message));
  }
}
  ```
  `if (context)` can not check whether the `context` has been destoried, and then it can cause uap. We need check `context->IsContextDestroyed()`.
  ```c++
  // Now that the context and context group no longer hold on to the
  // objects they create, and now that the objects are eagerly finalized
  // rather than the context, there is very little useful work that this
  // destructor can do, since it's not allowed to touch other on-heap
  // objects. All it can do is destroy its underlying context, which, if
  // there are no other contexts in the same share group, will cause all of
  // the underlying graphics resources to be deleted. (Currently, it's
  // always the case that there are no other contexts in the same share
  // group -- resource sharing between WebGL contexts is not yet
  // implemented, and due to its complex semantics, it's doubtful that it
  // ever will be.)
void WebGLRenderingContextBase::DestroyContext() {
  if (!GetDrawingBuffer())
    return;

  clearProgramCompletionQueries();

  extensions_util_.reset();

  base::RepeatingClosure null_closure;
  base::RepeatingCallback<void(const char*, int32_t)> null_function;
  GetDrawingBuffer()->ContextProvider()->SetLostContextCallback(
      std::move(null_closure));
  GetDrawingBuffer()->ContextProvider()->SetErrorMessageCallback(
      std::move(null_function));

  DCHECK(GetDrawingBuffer());
  drawing_buffer_->BeginDestruction();
  drawing_buffer_ = nullptr;
}
  ```

  Do this cve for exercise aims to let you know this type of vulnerability.
  

</details>

--------
