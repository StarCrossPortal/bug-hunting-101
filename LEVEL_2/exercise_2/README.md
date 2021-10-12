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
You'd better read some introduce for [DOM](https://chromium.googlesource.com/chromium/src/+/8689d5f68d3ce081fb0b81230a4f316c03221418/third_party/blink/renderer/core/dom/#dom) and [layout](https://chromium.googlesource.com/chromium/src/+/8689d5f68d3ce081fb0b81230a4f316c03221418/third_party/blink/renderer/core/layout/#blink-layout) in chrome

tips: CanContainRange[xxxxxxxx]() is virtual func, you can find all its def carefully for the true one.
### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>
  
  At first, I analysis the [patched file](https://chromium.googlesource.com/chromium/src/+/978994829edb17b9583ab7a6a8b001a5b9dab04e/third_party/blink/renderer/core/layout/hit_test_result.cc), but have no idea about the bug, so I see more about this cve at issue website. I notice that it was found by [Grammarinator fuzzer](https://github.com/renatahodovan/grammarinator), and when I want to use this fuzzer to continue this analysis, the usage can't run properly at my local. I don't make much time on environment or it's usage, because I don't think I can do the same as the author of this fuzzer in just two days :/

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
         !To<LayoutBox>(layout_object)->Size().IsEmpty() &&         [1]
         !HasRenderedNonAnonymousDescendantsWithHeight(layout_object);
}
  ```
  We can know [1] trigger the uaf from break trace. So the `layout_object` can be free before call [1]. `layout_object->IsAtomicInlineLevel()` seems like just a judgement, so we can analysis `CanHaveChildrenForEditing(node)`. Because `layout_object` get from `node`, so `layout_object` can be deleted by `node`.

  ```c++
inline bool CanHaveChildrenForEditing(const Node* node) {
  return !node->IsTextNode() && node->CanContainRangeEndPoint();
}
  ```
  `node->CanContainRangeEndPoint()` is a virtual func which can be override. At first I ignore this point and just notice this func return false...
  ```c++
bool HTMLMeterElement::CanContainRangeEndPoint() const {
  GetDocument().UpdateStyleAndLayoutTreeForNode(this);          [2]
  return GetComputedStyle() && !GetComputedStyle()->HasEffectiveAppearance();
}
  ```
  Notice this UpdateStyle, I guess it can delete object.
  ```c++
void Document::UpdateStyleAndLayoutTreeForNode(const Node* node) {
  [ ... ]
  DisplayLockUtilities::ScopedForcedUpdate scoped_update_forced(node);
  UpdateStyleAndLayoutTree();  [3]
}
  ```
  in [3] and later will call delete, we can get this by call tree.
  ```shell
  #1 0x563e4438c880 in Free base/allocator/partition_allocator/partition_root.h:673
  #2 0x563e4438c880 in operator delete third_party/blink/renderer/core/layout/layout_object.cc:240   [4]
  #3 0x563e443c643f in blink::LayoutObject::Destroy() third_party/blink/renderer/core/layout/layout_object.cc:3826
  #4 0x563e443c6169 in blink::LayoutObject::DestroyAndCleanupAnonymousWrappers() layout_object.cc:?
  #5 0x563e42da53d3 in blink::Node::DetachLayoutTree(bool) third_party/blink/renderer/core/dom/node.cc:1714
  #6 0x563e42c3b542 in blink::Element::DetachLayoutTree(bool) element.cc:?
  #7 0x563e42a818bd in blink::ContainerNode::DetachLayoutTree(bool) third_party/blink/renderer/core/dom/container_node.cc:1014
  #8 0x563e42c3b534 in blink::Element::DetachLayoutTree(bool) third_party/blink/renderer/core/dom/element.cc:2807
  #9 0x563e42a818bd in blink::ContainerNode::DetachLayoutTree(bool) third_party/blink/renderer/core/dom/container_node.cc:1014
  #10 0x563e42c3b534 in blink::Element::DetachLayoutTree(bool) third_party/blink/renderer/core/dom/element.cc:2807
  #11 0x563e42da4968 in blink::Node::ReattachLayoutTree(blink::Node::AttachContext&) third_party/blink/renderer/core/dom/node.cc:1679
  #12 0x563e42c43106 in blink::Element::RebuildLayoutTree(blink::WhitespaceAttacher&) third_party/blink/renderer/core/dom/element.cc:3163
  #13 0x563e42a8660a in blink::ContainerNode::RebuildLayoutTreeForChild(blink::Node*, blink::WhitespaceAttacher&) third_party/blink/renderer/core/dom/container_node.cc:1378
  #14 0x563e42a869ca in blink::ContainerNode::RebuildChildrenLayoutTrees(blink::WhitespaceAttacher&) third_party/blink/renderer/core/dom/container_node.cc:1403
  #15 0x563e42c43428 in blink::Element::RebuildLayoutTree(blink::WhitespaceAttacher&) third_party/blink/renderer/core/dom/element.cc:3192
  #16 0x563e4293af00 in blink::StyleEngine::RebuildLayoutTree() third_party/blink/renderer/core/css/style_engine.cc:2071
  #17 0x563e4293c4d7 in blink::StyleEngine::UpdateStyleAndLayoutTree() third_party/blink/renderer/core/css/style_engine.cc:2110
  #18 0x563e42aee703 in blink::Document::UpdateStyle() third_party/blink/renderer/core/dom/document.cc:2540
  #19 0x563e42ade9f6 in blink::Document::UpdateStyleAndLayoutTree() third_party/blink/renderer/core/dom/document.cc:2493
  #20 0x563e42af049b in blink::Document::UpdateStyleAndLayoutTreeForNode(blink::Node const*)    [5]
  ```
  [5] is the func we mentioned above. [4] delete object and we can delete layout_object there by set content-visibility to hidden.
  
  So in `EndsOfNodeAreVisuallyDistinctPositions` after `CanHaveChildrenForEditing(node)` will trigger uaf


</details>

--------
