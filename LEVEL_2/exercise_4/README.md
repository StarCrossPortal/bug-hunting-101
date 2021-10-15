# Exercise 4

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2021-30565
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.


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


### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  ```c++
  std::unique_ptr<content::WebContents> TabStripModel::ReplaceWebContentsAt(
    int index,
    std::unique_ptr<WebContents> new_contents) {
  ReentrancyCheck reentrancy_check(&reentrancy_guard_);

  delegate()->WillAddWebContents(new_contents.get());

  DCHECK(ContainsIndex(index));

  FixOpeners(index);

  TabStripSelectionChange selection(GetActiveWebContents(), selection_model_);
  WebContents* raw_new_contents = new_contents.get();
  std::unique_ptr<WebContents> old_contents =
      contents_data_[index]->ReplaceWebContents(std::move(new_contents));

  // When the active WebContents is replaced send out a selection notification
  // too. We do this as nearly all observers need to treat a replacement of the
  // selected contents as the selection changing.
  if (active_index() == index) {
    selection.new_contents = raw_new_contents;
    selection.reason = TabStripModelObserver::CHANGE_REASON_REPLACED;
  }

  TabStripModelChange::Replace replace;
  replace.old_contents = old_contents.get();
  replace.new_contents = raw_new_contents;
  replace.index = index;
  TabStripModelChange change(replace);
  for (auto& observer : observers_)
    observer.OnTabStripModelChanged(this, change, selection);

  return old_contents;
}
```

</details>

--------
