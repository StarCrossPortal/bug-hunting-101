# Exercise 5

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2021-21159
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.


### Details

In level 2, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1171049

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset --hard ae7b398ad2ba00cbf901fda43305ad9b371d534a
```

### Related code
chrome/browser/ui/views/tabs/tab_drag_controller.cc
chrome/browser/ui/tabs/tab_strip_model.cc

you have to read the `tab_drag_controller.h` and `tab_strip_model.h` to understand some nouns.
tips:
**TabDragController**
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
**TabStripModel**
```c++
// TabStripModel
//
// A model & low level controller of a Browser Window tabstrip. Holds a vector
// of WebContentses, and provides an API for adding, removing and
// shuffling them, as well as a higher level API for doing specific Browser-
// related tasks like adding new Tabs from just a URL, etc.
//
// Each tab may be pinned. Pinned tabs are locked to the left side of the tab
// strip and rendered differently (small tabs with only a favicon). The model
// makes sure all pinned tabs are at the beginning of the tab strip. For
// example, if a non-pinned tab is added it is forced to be with non-pinned
// tabs. Requests to move tabs outside the range of the tab type are ignored.
// For example, a request to move a pinned tab after non-pinned tabs is ignored.
//
// A TabStripModel has one delegate that it relies on to perform certain tasks
// like creating new TabStripModels (probably hosted in Browser windows) when
// required. See TabStripDelegate above for more information.
//
// A TabStripModel also has N observers (see TabStripModelObserver above),
// which can be registered via Add/RemoveObserver. An Observer is notified of
// tab creations, removals, moves, and other interesting events. The
// TabStrip implements this interface to know when to create new tabs in
// the View, and the Browser object likewise implements to be able to update
// its bookkeeping when such events happen.
//
// This implementation of TabStripModel is not thread-safe and should only be
// accessed on the UI thread.
```
### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  ```c++
  // Restores |initial_selection_model_| to the |source_context_|.
void TabDragController::RestoreInitialSelection() {
  // First time detaching from the source tabstrip. Reset selection model to
  // initial_selection_model_. Before resetting though we have to remove all
  // the tabs from initial_selection_model_ as it was created with the tabs
  // still there.
  ui::ListSelectionModel selection_model = initial_selection_model_;   [1]
  for (DragData::const_reverse_iterator i(drag_data_.rbegin());
       i != drag_data_.rend(); ++i) {
    if (i->source_model_index != TabStripModel::kNoTab)
      selection_model.DecrementFrom(i->source_model_index);
  }
  // We may have cleared out the selection model. Only reset it if it
  // contains something.
  if (selection_model.empty())
    return;

  // The anchor/active may have been among the tabs that were dragged out. Force
  // the anchor/active to be valid.
  if (selection_model.anchor() == ui::ListSelectionModel::kUnselectedIndex)
    selection_model.set_anchor(*selection_model.selected_indices().begin());
  if (selection_model.active() == ui::ListSelectionModel::kUnselectedIndex)
    selection_model.set_active(*selection_model.selected_indices().begin());
  source_context_->GetTabStripModel()->SetSelectionFromModel(selection_model); [2]
}
=================================================================
void TabStripModel::SetSelectionFromModel(ui::ListSelectionModel source) {
  DCHECK_NE(ui::ListSelectionModel::kUnselectedIndex, source.active());
  SetSelection(std::move(source), TabStripModelObserver::CHANGE_REASON_NONE,  [3]
               /*triggered_by_other_operation=*/false);
}
  ```
  [1] make `selection_model == initial_selection_model_`

  > Tabs in |source_context_| may have closed since the drag began. In that
  > case, |initial_selection_model_| may include indices that are no longer
  > valid in |source_context_|.

  [2]|[3] call `SetSelection` and `selection_model` as its parameter which have unvalid indices.

  ```c++
TabStripSelectionChange TabStripModel::SetSelection(
    ui::ListSelectionModel new_model,
    TabStripModelObserver::ChangeReason reason,
    bool triggered_by_other_operation) {
  TabStripSelectionChange selection;
  selection.old_model = selection_model_;
  selection.old_contents = GetActiveWebContents();
  selection.new_model = new_model;
  selection.reason = reason;

  // This is done after notifying TabDeactivated() because caller can assume
  // that TabStripModel::active_index() would return the index for
  // |selection.old_contents|.
  selection_model_ = new_model;                                 [4]
  selection.new_contents = GetActiveWebContents();

  if (!triggered_by_other_operation &&
      (selection.active_tab_changed() || selection.selection_changed())) {
    if (selection.active_tab_changed()) {
      auto now = base::TimeTicks::Now();
      if (selection.new_contents &&
          selection.new_contents->GetRenderWidgetHostView()) {
        auto input_event_timestamp =
            tab_switch_event_latency_recorder_.input_event_timestamp();
        // input_event_timestamp may be null in some cases, e.g. in tests.
        selection.new_contents->GetRenderWidgetHostView()
            ->SetRecordContentToVisibleTimeRequest(
                !input_event_timestamp.is_null() ? input_event_timestamp : now,
                resource_coordinator::ResourceCoordinatorTabHelper::IsLoaded(
                    selection.new_contents),
                /*show_reason_tab_switching=*/true,
                /*show_reason_unoccluded=*/false,
                /*show_reason_bfcache_restore=*/false);
      }
      tab_switch_event_latency_recorder_.OnWillChangeActiveTab(now);
    }
    TabStripModelChange change;
    auto visibility_tracker = InstallRenderWigetVisibilityTracker(selection);
    for (auto& observer : observers_)
      observer.OnTabStripModelChanged(this, change, selection);
  }

  return selection;
}
  ```
  [4] Notice that `SetSelection` have no check about whether the `new_model.selected_indices()` are exist.

  In a word, detaching a drag after a tab closed will trigger the uaf.
</details>

--------
