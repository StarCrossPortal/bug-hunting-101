# Exercise 5

##  CVE-2021-21203
I sugget you don't search any report about it to prevents get too much info like patch.

This time we do it by code audit

### Details

> Don't erase InterpolationTypes used by other documents
>
> A registered custom property in one document caused the entry for the
> same custom property (unregistered) used in another document to be
> deleted, which caused a use-after-free.
>
> Only store the CSSDefaultInterpolationType for unregistered custom
> properties and never store registered properties in the map. They may
> have different types in different documents when registered.

You can read [this](https://chromium.googlesource.com/chromium/src/+/af77c20371d1418300cefbc5fa6779067b7792cf/third_party/blink/renderer/core/animation/#core_animation) to know what about `animation`



---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1192054

</details>

--------

### Set environment

Just like exercise_4, we need chromium. I recomend you do as [offical gudience](https://chromium.googlesource.com/chromium/src/+/refs/heads/main/docs/linux/build_instructions.md). If you have installed `depot_tools` ago, you just need `fetch chromium`.

When you finish the above
```sh
git reset --hard 7e5707cc5f46b0155b9e42b121c8e2128c05f178 
```

### Related code
we can analysis the source file [online](https://chromium.googlesource.com/chromium/src/+/af77c20371d1418300cefbc5fa6779067b7792cf/third_party/blink/renderer/core/animation/css_interpolation_types_map.cc) or offline.

This time you need to analysis entire file [`third_party/blink/renderer/core/animation/css_interpolation_types_map.cc`](https://source.chromium.org/chromium/chromium/src/+/af77c20371d1418300cefbc5fa6779067b7792cf:third_party/blink/renderer/core/animation/css_interpolation_types_map.cc), this bug can be easily found if you read `Details` carefully ;)

### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>

  `Details` has clearly told us the cause of the vulnerability. **A registered custom property in one document caused the entry for the same custom property (unregistered) used in another document to be deleted, which caused a use-after-free**
  This mean if we register a `custom property` and then the `entry` of the same `custom property` in another document which `unregistered` will be deleted by `erase`.
  ```c++
  const InterpolationTypes& CSSInterpolationTypesMap::Get(
    const PropertyHandle& property) const {
  using ApplicableTypesMap =
      HashMap<PropertyHandle, std::unique_ptr<const InterpolationTypes>>;  
  // TODO(iclelland): Combine these two hashmaps into a single map on
  // std::pair<bool,property>
  DEFINE_STATIC_LOCAL(ApplicableTypesMap, all_applicable_types_map, ());
  DEFINE_STATIC_LOCAL(ApplicableTypesMap, composited_applicable_types_map, ());

  ApplicableTypesMap& applicable_types_map =
      allow_all_animations_ ? all_applicable_types_map
                            : composited_applicable_types_map;

  auto entry = applicable_types_map.find(property);               [1] find entry (HashMap)
  bool found_entry = entry != applicable_types_map.end();

  // Custom property interpolation types may change over time so don't trust the
  // applicableTypesMap without checking the registry.
  if (registry_ && property.IsCSSCustomProperty()) {
    const auto* registration = GetRegistration(registry_, property);  [2] registr
    if (registration) {
      if (found_entry) {
        applicable_types_map.erase(entry);          [3] delete entry
      }
      return registration->GetInterpolationTypes();
    }
  }

  if (found_entry) {
    return *entry->value;
  }
  [ ... ]
  ============================================================================
  static const PropertyRegistration* GetRegistration(
    const PropertyRegistry* registry,
    const PropertyHandle& property) {
    DCHECK(property.IsCSSCustomProperty());
    if (!registry) {
      return nullptr;
    }
    return registry->Registration(property.CustomPropertyName());
  }
  ```

</details>

--------

