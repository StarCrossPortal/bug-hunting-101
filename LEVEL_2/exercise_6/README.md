# Exercise 6

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2021-21190
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.


### Details

In level 2, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1166091

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset --hard 53913f6b138c7b0cd9771c1b6ab82a143996ef9e
```

### Related code
pdf/pdfium/pdfium_page.cc


### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  ```c++
// Function: FPDF_PageToDevice
//          Convert the page coordinates of a point to screen coordinates.
// Parameters:
//          page        -   Handle to the page. Returned by FPDF_LoadPage.
//          start_x     -   Left pixel position of the display area in
//                          device coordinates.
//          start_y     -   Top pixel position of the display area in device
//                          coordinates.
//          size_x      -   Horizontal size (in pixels) for displaying the page.
//          size_y      -   Vertical size (in pixels) for displaying the page.
//          rotate      -   Page orientation:
//                            0 (normal)
//                            1 (rotated 90 degrees clockwise)
//                            2 (rotated 180 degrees)
//                            3 (rotated 90 degrees counter-clockwise)
//          page_x      -   X value in page coordinates.
//          page_y      -   Y value in page coordinate.
//          device_x    -   A pointer to an integer receiving the result X
//                          value in device coordinates.
//          device_y    -   A pointer to an integer receiving the result Y
//                          value in device coordinates.
// Return value:
//          Returns true if the conversion succeeds, and |device_x| and
//          |device_y| successfully receives the converted coordinates.
// Comments:
//          See comments for FPDF_DeviceToPage().
FPDF_EXPORT FPDF_BOOL FPDF_CALLCONV FPDF_PageToDevice(FPDF_PAGE page,
                                                      int start_x,
                                                      int start_y,
                                                      int size_x,
                                                      int size_y,
                                                      int rotate,
                                                      double page_x,
                                                      double page_y,
                                                      int* device_x,
                                                      int* device_y) {
  if (!page || !device_x || !device_y)
    return false;

  IPDF_Page* pPage = IPDFPageFromFPDFPage(page);
  const FX_RECT rect(start_x, start_y, start_x + size_x, start_y + size_y);
  CFX_PointF page_point(static_cast<float>(page_x), static_cast<float>(page_y));
  absl::optional<CFX_PointF> pos =
      pPage->PageToDevice(rect, rotate, page_point);
  if (!pos.has_value())
    return false;

  *device_x = FXSYS_roundf(pos->x);
  *device_y = FXSYS_roundf(pos->y);
  return true;
}
  ```
  This cve reward 500, but the same issue exists in many places.

  ```c++
gfx::Rect PDFiumPage::PageToScreen(const gfx::Point& page_point,
                                   double zoom,
                                   double left,
                                   double top,
                                   double right,
                                   double bottom,
                                   PageOrientation orientation) const {
  if (!available_)
    return gfx::Rect();
 [ ... ]
  FPDF_BOOL ret = FPDF_PageToDevice(
      page(), static_cast<int>(start_x), static_cast<int>(start_y),
      static_cast<int>(ceil(size_x)), static_cast<int>(ceil(size_y)),
      ToPDFiumRotation(orientation), left, top, &new_left, &new_top);
  DCHECK(ret);
  ret = FPDF_PageToDevice(
      page(), static_cast<int>(start_x), static_cast<int>(start_y),
      static_cast<int>(ceil(size_x)), static_cast<int>(ceil(size_y)),
      ToPDFiumRotation(orientation), right, bottom, &new_right, &new_bottom);
  DCHECK(ret);     [1]
  [ ... ]
    }
  ```
  [1] `FPDF_PageToDevice` return false if `pos` memory uninitialized.

  > DCHECK() here isn't sufficient to prevent the use of uninitialized
  > memory should this someday return false.

  The purpose of using this cve is to remind everyone that if the return value means important, we must CHECK it not just DCHECK. For example, if there is func check whether the `var` has been initialize, return true or false. But we DCHECK the return value, it cause that if the `var` is uninitialized, we cann't prevent to use it in release build.

</details>

--------
