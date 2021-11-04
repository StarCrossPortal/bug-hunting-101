# Exercise 4

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21207
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.



### Details

In level 3, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1185732

</details>

--------

### Set environment

after fetch chromium
```sh
git reset --hard 86a37b3c8fdc47b0ba932644fd61bbc791c82357
```



### Related code

mojo/public/cpp/bindings/receiver_set.h

read this [doc](https://chromium.googlesource.com/chromium/src/+/668cf831e91210d4f23e815e07ff1421f3ee9747/mojo/public/cpp/bindings#Receiver-Sets) get info about Receiver Sets

### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  
</details>

--------
