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
 
  ```

</details>

--------
