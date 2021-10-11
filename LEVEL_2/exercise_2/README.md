# Exercise 2

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2021-21122
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.


### Details

In level 2, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1162131#c13

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset -hard 978994829edb17b9583ab7a6a8b001a5b9dab04e
```

### Related code

[src/third_party/blink/renderer/core/editing/visible_units.cc](https://chromium.googlesource.com/chromium/src/+/978994829edb17b9583ab7a6a8b001a5b9dab04e/third_party/blink/renderer/core/editing/visible_units.cc)


### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>
  
  At first, I analysis the [patched file](https://chromium.googlesource.com/chromium/src/+/978994829edb17b9583ab7a6a8b001a5b9dab04e/third_party/blink/renderer/core/layout/hit_test_result.cc), but have no idea about the bug, so I see more about this cve at issue website. I notice that it was found by [Grammarinator fuzzer](https://github.com/renatahodovan/grammarinator), and when I want to use this fuzzer to continue this analysis, the usage can't run properly at my local. I don't make much time on environment or it's usage, because I don't think I can do the same as the author of this fuzzer :/

  Some bug found by fuzzer are difficult to find by analysis the source files, so I want continue this work with the help of break trace which author [pasted](https://bugs.chromium.org/p/chromium/issues/detail?id=1162131).
  
  I decide to analysis these func from top to bottom, the first
  ```c++
bool EndsOfNodeAreVisuallyDistinctPositions(const Node* node) {
  if (!node)
    return false;

  LayoutObject* layout_object = node->GetLayoutObject();
  if (!layout_object)
    return false;

  if (!layout_object->IsInline())
    return true;

  // Don't include inline tables.
  if (IsA<HTMLTableElement>(*node))
    return false;

  // A Marquee elements are moving so we should assume their ends are always
  // visibily distinct.
  if (IsA<HTMLMarqueeElement>(*node))
    return true;

  // There is a VisiblePosition inside an empty inline-block container.
  return layout_object->IsAtomicInlineLevel() &&
         CanHaveChildrenForEditing(node) &&
         !To<LayoutBox>(layout_object)->Size().IsEmpty() &&  [1]
         !HasRenderedNonAnonymousDescendantsWithHeight(layout_object);
}
  ```


</details>

--------
