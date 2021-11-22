# Exercise 4

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2021-30565
I sugget you don't search any report about it to prevents get too much info like patch.


### Details

In level 2, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1210985  

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset --hard e382f185aaee6d4f4a5f8762f1a1ae89bcc0d046
```

### Related code

[chrome/browser/ui/tabs/tab_strip_model.cc](https://crrev.com/e382f185aaee6d4f4a5f8762f1a1ae89bcc0d046/chrome/browser/ui/tabs/tab_strip_model.cc)

This time we analysis [`tab`](https://www.chromium.org/user-experience/tabs), a module of chrome. You can read [this](https://www.chromium.org/developers/design-documents/tab-strip-mac) to get info of `tab strip`.

tips: You can get help from [CVE-2021-30526](https://bugs.chromium.org/p/chromium/issues/detail?id=1198717) 

### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>

  In my cognition, we can focus on `pinned tab`, because I notice this comment
  ```c++
// Each tab may be pinned. Pinned tabs are locked to the left side of the tab
// strip and rendered differently (small tabs with only a favicon). The model
// makes sure all pinned tabs are at the beginning of the tab strip. 
  ```
  If we can make a pinned tab is not at the left side of the tab strip, what happen?

  ```c++
int TabStripModel::MoveWebContentsAt(int index,
                                     int to_position,
                                     bool select_after_move) {
  to_position = ConstrainMoveIndex(to_position, IsTabPinned(index)); [1]

  if (index == to_position)
    return to_position;

  MoveWebContentsAtImpl(index, to_position, select_after_move);
  EnsureGroupContiguity(to_position);

  return to_position;
}
======================================================
int TabStripModel::ConstrainMoveIndex(int index, bool pinned_tab) const {
  return pinned_tab
             ? base::ClampToRange(index, 0, IndexOfFirstNonPinnedTab() - 1) [2]
             : base::ClampToRange(index, IndexOfFirstNonPinnedTab(),
                                  count() - 1);
}
======================================================
template <class T>
constexpr const T& ClampToRange(const T& value, const T& min, const T& max) {
  return std::min(std::max(value, min), max);
}
  ```
  If we make pinned tab at index 1, and non-pinned tab at 0. Then we move pinned tab to 0, this will trigger `MoveWebContentsAt`.

  [1] `ConstrainMoveIndex(0, true);`, and [2] `min(1, 0, 0 - 1)`. This will result in an OOB write

  How can we make a pinned tab that's not at the start of the tab strip?
  ```c++
 void TabStripModel::MoveTabRelative(bool forward) {
  const int offset = forward ? 1 : -1;

  // TODO: this needs to be updated for multi-selection.
  const int current_index = active_index();
  absl::optional<tab_groups::TabGroupId> current_group =
      GetTabGroupForTab(current_index);

  int target_index = std::max(std::min(current_index + offset, count() - 1), 0);
  absl::optional<tab_groups::TabGroupId> target_group =
      GetTabGroupForTab(target_index);

  // If the tab is at a group boundary and the group is expanded, instead of
  // actually moving the tab just change its group membership.
  if (current_group != target_group) {
    if (current_group.has_value()) {
      UngroupTab(current_index);
      return;
    } else if (target_group.has_value()) {
      // If the tab is at a group boundary and the group is collapsed, treat the
      // collapsed group as a tab and find the next available slot for the tab
      // to move to.
      const TabGroup* group = group_model_->GetTabGroup(target_group.value());
      if (group->visual_data()->is_collapsed()) {
        const gfx::Range tabs_in_group = group->ListTabs();
        target_index =
            forward ? tabs_in_group.end() - 1 : tabs_in_group.start();
      } else {
        GroupTab(current_index, target_group.value());
        return;
      }
    }
  }
  MoveWebContentsAt(current_index, target_index, true);
}
  ```
  `TabStripModel::MoveTabRelative` doesn't check whether a tab is pinned, so we can move pinned tab to a group. But a pinned tab typically can't be placed in a group, so the `Groups.move` operation have no check about move pinned tab to index 1 or other. Then the `IndexOfFirstNonPinnedTab` can be 0 at the same time pinned tab index 1.



</details>

--------
