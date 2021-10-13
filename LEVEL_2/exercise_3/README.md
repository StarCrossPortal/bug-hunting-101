# Exercise 3

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2021-21112
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.


### Details

In level 2, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1151298

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset -hard 13aa7b32816e52bf1242d073ada2c892798190e7 
```

### Related code

[third_party/blink/renderer/modules/compression/deflate_transformer.cc](https://chromium.googlesource.com/chromium/src.git/+/13aa7b32816e52bf1242d073ada2c892798190e7/third_party/blink/renderer/modules/compression/deflate_transformer.cc)
[third_party/blink/renderer/modules/compression/inflate_transformer.cc](https://chromium.googlesource.com/chromium/src.git/+/13aa7b32816e52bf1242d073ada2c892798190e7/third_party/blink/renderer/modules/compression/inflate_transformer.cc)
[third_party/blink/renderer/modules/compression/inflate_transformer.h](https://chromium.googlesource.com/chromium/src.git/+/13aa7b32816e52bf1242d073ada2c892798190e7/third_party/blink/renderer/modules/compression/inflate_transformer.h)

You can read this [doc](https://docs.google.com/document/d/1TovyqqeC3HoO0A4UUBKiCyhZlQSl7jM_F7KbWjK2Gcs/edit)  and [this](https://github.com/WICG/compression/blob/main/explainer.md) to get some info about `CompressionStreams` in chrome

### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>
  



</details>

--------
