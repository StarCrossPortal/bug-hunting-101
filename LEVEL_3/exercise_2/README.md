# Exercise 2

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21224
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.



### Details

In level 3, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1195777

</details>

--------

### Set environment

Welcome to V8

if you have fetched v8, just
```sh
git reset --hard f87baad0f8b3f7fda43a01b9af3dd940dd09a9a3 
gclient sync
```
But if you not
```sh
# get depot_tools
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
# add to env var
echo 'export PATH=$PATH:"/path/to/depot_tools"' >> ~/.bashrc
# get v8 source code
fetch v8
# chenge to right commit
cd v8
git reset --hard f87baad0f8b3f7fda43a01b9af3dd940dd09a9a3
# download others
gclient sync
```



### Related code

src/compiler/representation-change.cc


### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  ```c++
// The {UseInfo} class is used to describe a use of an input of a node.
//
// This information is used in two different ways, based on the phase:
//
// 1. During propagation, the use info is used to inform the input node
//    about what part of the input is used (we call this truncation) and what
//    is the preferred representation. For conversions that will require
//    checks, we also keep track of whether a minus zero check is needed.
//
// 2. During lowering, the use info is used to properly convert the input
//    to the preferred representation. The preferred representation might be
//    insufficient to do the conversion (e.g. word32->float64 conv), so we also
//    need the signedness information to produce the correct value.
//    Additionally, use info may contain {CheckParameters} which contains
//    information for the deoptimizer such as a CallIC on which speculation
//    should be disallowed if the check fails.
  ```

  When do truncation we need check the `{CheckParameters}` like `use_info.type_check() == TypeCheckKind::kSignedSmall`

  ```c++
enum class TypeCheckKind : uint8_t {
  kNone,
  kSignedSmall,
  kSigned32,
  kSigned64,
  kNumber,
  kNumberOrBoolean,
  kNumberOrOddball,
  kHeapObject,
  kBigInt,
  kArrayIndex
};
  ```

</details>

--------
