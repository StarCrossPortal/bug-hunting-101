# Exercise 7

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21155
I sugget you don't search any report about it to prevents get too much info like patch.



### Details

In level 3, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1175500

</details>

--------

### Set environment

after fetch chromium
```sh
git reset --hard c57ba0a5dacc78c7a1954c99d381b77ec771fba6
```



### Related code

chrome/browser/ui/views/tabs/tab_drag_controller.cc

About TabDragController:
```c++
// TabDragController is responsible for managing the tab dragging session. When
// the user presses the mouse on a tab a new TabDragController is created and
// Drag() is invoked as the mouse is dragged. If the mouse is dragged far enough
// TabDragController starts a drag session. The drag session is completed when
// EndDrag() is invoked (or the TabDragController is destroyed).
//
// While dragging within a tab strip TabDragController sets the bounds of the
// tabs (this is referred to as attached). When the user drags far enough such
// that the tabs should be moved out of the tab strip a new Browser is created
// and RunMoveLoop() is invoked on the Widget to drag the browser around. This
// is the default on aura.
```

### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>


  ```c++
void TabDragController::RevertDragAt(size_t drag_index) {
  DCHECK_NE(current_state_, DragState::kNotStarted);
  DCHECK(source_context_);

  base::AutoReset<bool> setter(&is_mutating_, true);
  TabDragData* data = &(drag_data_[drag_index]);
  int target_index = data->source_model_index;
  if (attached_context_) {
    int index = attached_context_->GetTabStripModel()->GetIndexOfWebContents(
        data->contents);
    if (attached_context_ != source_context_) {
      // The Tab was inserted into another TabDragContext. We need to
      // put it back into the original one.
      std::unique_ptr<content::WebContents> detached_web_contents =
          attached_context_->GetTabStripModel()->DetachWebContentsAt(index);
      // TODO(beng): (Cleanup) seems like we should use Attach() for this
      //             somehow.
      source_context_->GetTabStripModel()->InsertWebContentsAt(
          target_index, std::move(detached_web_contents),
          (data->pinned ? TabStripModel::ADD_PINNED : 0));
    } else {
      // The Tab was moved within the TabDragContext where the drag
      // was initiated. Move it back to the starting location.

      // If the target index is to the right, then other unreverted tabs are
      // occupying indices between this tab and the target index. Those
      // unreverted tabs will later be reverted to the right of the target
      // index, so we skip those indices.
      if (target_index > index) {
        for (size_t i = drag_index + 1; i < drag_data_.size(); ++i) {
          if (drag_data_[i].contents)
            ++target_index;
        }
      }
      source_context_->GetTabStripModel()->MoveWebContentsAt(   [1]
          index, target_index, false);
    }
  } else {
    // The Tab was detached from the TabDragContext where the drag
    // began, and has not been attached to any other TabDragContext.
    // We need to put it back into the source TabDragContext.
    source_context_->GetTabStripModel()->InsertWebContentsAt(
        target_index, std::move(data->owned_contents),
        (data->pinned ? TabStripModel::ADD_PINNED : 0));
  }
  source_context_->GetTabStripModel()->UpdateGroupForDragRevert(
      target_index,
      data->tab_group_data.has_value()
          ? base::Optional<tab_groups::TabGroupId>{data->tab_group_data.value()
                                                       .group_id}
          : base::nullopt,
      data->tab_group_data.has_value()
          ? base::Optional<
                tab_groups::TabGroupVisualData>{data->tab_group_data.value()
                                                    .group_visual_data}
          : base::nullopt);
}
  ```
  [1] We can get info about these conditions judged by `if` from comment, `MoveWebContentsAt` do just like its name.
  ```c++
int TabStripModel::MoveWebContentsAt(int index,
                                     int to_position,
                                     bool select_after_move) {
  ReentrancyCheck reentrancy_check(&reentrancy_guard_);

  CHECK(ContainsIndex(index));

  to_position = ConstrainMoveIndex(to_position, IsTabPinned(index));   [2]

  if (index == to_position)
    return to_position;       [3]

  MoveWebContentsAtImpl(index, to_position, select_after_move);
  EnsureGroupContiguity(to_position);

  return to_position;
}
  ```

  [2] `to_position` is `target_index`, and it has been chenged, [3] return the `to_position` which has been chenged, we should assign it back to `target_index`. But in truth, we don't.

  ```c++
if (target_index > index) {
  for (size_t i = drag_index + 1; i < drag_data_.size(); ++i) {
    if (drag_data_[i].contents)
      ++target_index;
  }
}
source_context_->GetTabStripModel()->MoveWebContentsAt(    // The return value should assign to target_index
    index, target_index, false);
  ```
  
  **Poc**
  ```html

<button id="test1">Trigger</button>
<script>
	var test1 = document.getElementById("test1");
	test1.onclick = () =>{
		setTimeout(()=>{window.close();},3000);
	}
</script>
  ```
  
</details>

--------
