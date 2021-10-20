# Exercise 7


## CVE-2021-30565
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.


### Details

> cppgc: Fix ephemeron iterations
>
> If processing the marking worklists found new ephemeron pairs, but
processing the existing ephemeron pairs didn't mark new objects, marking
would stop and the newly discovered ephemeron pairs would not be
processed. This can lead to a marked key with an unmarked value.


An ephemeron pair is used to conditionally retain an object.
The `value` will be kept alive only if the `key` is alive.


### Set environment

set v8 environment
```sh
# get depot_tools
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
# add to env var
echo 'export PATH=$PATH:"/path/to/depot_tools"' >> ~/.bashrc
# get v8 source code
fetch v8
# chenge to right commit
cd v8
git reset --hard f41f4fb4e66916936ed14d8f9ee20d5fb0afc548
# download others
gclient sync
# get ninja for compile
git clone https://github.com/ninja-build/ninja.git
cd ninja && ./configure.py --bootstrap && cd ..
# set environment variable
echo 'export PATH=$PATH:"/path/to/ninja"' >> ~/.bashrc

# compile debug
tools/dev/v8gen.py x64.debug
ninja -C out.gn/x64.debug d8
# compile release (optional)
tools/dev/v8gen.py x64.release
ninja -C out.gn/x64.release d8
```

### Related code
`src/heap/cppgc/marker.cc`
`src/heap/cppgc/marking-state.h`
```c++
bool MarkerBase::ProcessWorklistsWithDeadline(
    size_t marked_bytes_deadline, v8::base::TimeTicks time_deadline) {
  StatsCollector::EnabledScope stats_scope(
      heap().stats_collector(), StatsCollector::kMarkTransitiveClosure);
  do {
    if ((config_.marking_type == MarkingConfig::MarkingType::kAtomic) ||
        schedule_.ShouldFlushEphemeronPairs()) {
      mutator_marking_state_.FlushDiscoveredEphemeronPairs();
    }

    // Bailout objects may be complicated to trace and thus might take longer
    // than other objects. Therefore we reduce the interval between deadline
    // checks to guarantee the deadline is not exceeded.
    [ ... ]
    {
      StatsCollector::EnabledScope inner_stats_scope(
          heap().stats_collector(), StatsCollector::kMarkProcessEphemerons);
      if (!DrainWorklistWithBytesAndTimeDeadline(
              mutator_marking_state_, marked_bytes_deadline, time_deadline,
              mutator_marking_state_.ephemeron_pairs_for_processing_worklist(),
              [this](const MarkingWorklists::EphemeronPairItem& item) {
                mutator_marking_state_.ProcessEphemeron(
                    item.key, item.value, item.value_desc, visitor());
              })) {
        return false;
      }
    }
  } while (!mutator_marking_state_.marking_worklist().IsLocalAndGlobalEmpty());
  return true;
}
```

```c++
void MarkingStateBase::ProcessEphemeron(const void* key, const void* value,
                                        TraceDescriptor value_desc,
                                        Visitor& visitor) {
  // Filter out already marked keys. The write barrier for WeakMember
  // ensures that any newly set value after this point is kept alive and does
  // not require the callback.
  if (!HeapObjectHeader::FromObject(key)
           .IsInConstruction<AccessMode::kAtomic>() &&
      HeapObjectHeader::FromObject(key).IsMarked<AccessMode::kAtomic>()) {
    if (value_desc.base_object_payload) {
      MarkAndPush(value_desc.base_object_payload, value_desc);
    } else {
      // If value_desc.base_object_payload is nullptr, the value is not GCed and
      // should be immediately traced.
      value_desc.callback(&visitor, value);
    }
    return;
  }
  // |discovered_ephemeron_pairs_worklist_| may still hold ephemeron pairs with
  // dead keys.
  discovered_ephemeron_pairs_worklist_.Push({key, value, value_desc});
}
```

### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  https://chromium.googlesource.com/v8/v8.git/+/e677a6f6b257e992094b9183a958b67ecc68aa85
  ```c++
void MarkingStateBase::ProcessEphemeron(const void* key, const void* value,
                                        TraceDescriptor value_desc,
                                        Visitor& visitor) {
  // Filter out already marked keys. The write barrier for WeakMember
  // ensures that any newly set value after this point is kept alive and does
  // not require the callback.
  if (!HeapObjectHeader::FromObject(key)
          .IsInConstruction<AccessMode::kAtomic>() &&   [1]
      HeapObjectHeader::FromObject(key).IsMarked<AccessMode::kAtomic>()) { [2]
    if (value_desc.base_object_payload) {
      MarkAndPush(value_desc.base_object_payload, value_desc);
    } else {
      // If value_desc.base_object_payload is nullptr, the value is not GCed and
      // should be immediately traced.
      value_desc.callback(&visitor, value);
    }
    return;
  }
  discovered_ephemeron_pairs_worklist_.Push({key, value, value_desc});
}
  ```
  [1] `IsInConstruction == false` means the the data of `HeapObjectHeader` all setted up.

  [2] `IsMarked == true` means this `HeapObjectHeader` has been marked.

  `ProcessEphemeron` means if key has been marked and `value_desc.base_object_payload`(may be write barrier?) not null, we need to mark value, else we find new  `ephemeron_pairs`.

  But if `value_desc.base_object_payload` have not been set, we need to search old space for what object prt to this value. During this process, the `ProcessEphemeron` can be called recursively. Finally,

  ```c++
const HeapObjectHeader& HeapObjectHeader::FromObject(const void* object) {  [3]
  return *reinterpret_cast<const HeapObjectHeader*>(
      static_cast<ConstAddress>(object) - sizeof(HeapObjectHeader));
}
============================================================
template <AccessMode mode>
bool HeapObjectHeader::IsInConstruction() const {
  const uint16_t encoded =
      LoadEncoded<mode, EncodedHalf::kHigh, std::memory_order_acquire>();
  return !FullyConstructedField::decode(encoded);         [4]
}
============================================================
template <AccessMode mode>
bool HeapObjectHeader::IsMarked() const {
  const uint16_t encoded =
      LoadEncoded<mode, EncodedHalf::kLow, std::memory_order_relaxed>();
  return MarkBitField::decode(encoded);                    [5]
}
  ```
  [3] return the ptr to `HeapObjectHeader`, you can treat it as `addrOf(Chunk) - sizeof(ChunkHeader)` can get the addr of ChunkHeader

  [4] and [5] can get info of the `HeapObjectHeader` which be organized in some regular pattern. The following content explains this clearly
  ```c++
// Used in |encoded_high_|.
using FullyConstructedField = v8::base::BitField16<bool, 0, 1>;        [6]
using UnusedField1 = FullyConstructedField::Next<bool, 1>;
using GCInfoIndexField = UnusedField1::Next<GCInfoIndex, 14>;
// Used in |encoded_low_|.
using MarkBitField = v8::base::BitField16<bool, 0, 1>;
using SizeField = void;  // Use EncodeSize/DecodeSize instead.
==============================================
// Extracts the bit field from the value.
static constexpr T decode(U value) {
  return static_cast<T>((value & kMask) >> kShift);           [7]
}
  ```
  you can understand [6] better by this.

   ```c++
  HeapObjectHeader contains meta data per object and is prepended to each
  object.

  +-----------------+------+------------------------------------------+
  | name | bits | |
  +-----------------+------+------------------------------------------+
  | padding | 32 | Only present on 64-bit platform. |
  +-----------------+------+------------------------------------------+
  | GCInfoIndex | 14 | |
  | unused | 1 | |
  | in construction | 1 | In construction encoded as |false|. |
  +-----------------+------+------------------------------------------+
  | size | 15 | 17 bits because allocations are aligned. |
  | mark bit | 1 | |
  +-----------------+------+------------------------------------------+
  
  Notes:
  - See |GCInfoTable| for constraints on GCInfoIndex.
  - |size| for regular objects is encoded with 15 bits but can actually
  represent sizes up to |kBlinkPageSize| (2^17) because allocations are
  always 4 byte aligned (see kAllocationGranularity) on 32bit. 64bit uses
  8 byte aligned allocations which leaves 1 bit unused.
  - |size| for large objects is encoded as 0. The size of a large object is
  stored in |LargeObjectPage::PayloadSize()|.
  - |mark bit| and |in construction| bits are located in separate 16-bit halves
  to allow potentially accessing them non-atomically.
  ```
  So [7] can get specific bit of `HeapObjectHeader` | meta data, means the Object status information

</details>

--------
