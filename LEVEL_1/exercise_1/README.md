# Exercise 1

## CVE-2020-6542
I choose **CVE-2020-6542**, and I sugget you don't search any report about it to prevents get too much info like patch.


### Details

> Google Chrome WebGL Buffer11::getBufferStorage Code Execution Vulnerability
>
> Google Chrome is a cross-platform web browser developed by Google. It supports many features, including WebGL (Web Graphics Library), a JavaScript API for rendering interactive 2-D and 3-D graphics.
>
> In some specific cases after binding a zero size buffer we could end up trying to use a buffer storage that was no longer valid. Fix this by ensuring we don't flush dirty bits when we have an early exit due to a zero size buffer. 

---------
<details>
  <summary>For more info click me!</summary>

   Chromium crashes inside the `Buffer11::getBufferStorage` function. This is because newStorage element points to previously freed memory, leading to a use-after-free vulnerability.
</details>

**You'd better read some doc about ANGLE to understand the source code**

--------

### Version

Google Chrome  84.0.4147.89

Google Chrome  85.0.4169.0 (Developer Build) (64-bit)

we can analysis the source file [online](https://chromium.googlesource.com/angle/angle/+/034a8b3f3c5c8e7e1629b8ac88cadb72ea68cf23/src/libANGLE/renderer/d3d/d3d11/VertexArray11.cpp#246)

### Related code

May be you need fetch the source code
```
git clone https://chromium.googlesource.com/angle/angle
cd angle
git reset --hard 50a2725742948702720232ba46be3c1f03822ada
```
```c++
angle::Result VertexArray11::updateDirtyAttribs(const gl::Context *context,
                                                const gl::AttributesMask &activeDirtyAttribs)
{
    const auto &glState  = context->getState();
    const auto &attribs  = mState.getVertexAttributes();
    const auto &bindings = mState.getVertexBindings();

    for (size_t dirtyAttribIndex : activeDirtyAttribs)
    {
        mAttribsToTranslate.reset(dirtyAttribIndex);

        auto *translatedAttrib   = &mTranslatedAttribs[dirtyAttribIndex];
        const auto &currentValue = glState.getVertexAttribCurrentValue(dirtyAttribIndex);

        // Record basic attrib info
        translatedAttrib->attribute        = &attribs[dirtyAttribIndex];
        translatedAttrib->binding          = &bindings[translatedAttrib->attribute->bindingIndex];
        translatedAttrib->currentValueType = currentValue.Type;
        translatedAttrib->divisor =
            translatedAttrib->binding->getDivisor() * mAppliedNumViewsToDivisor;

        switch (mAttributeStorageTypes[dirtyAttribIndex])
        {
            case VertexStorageType::DIRECT:
                VertexDataManager::StoreDirectAttrib(context, translatedAttrib);
                break;
            case VertexStorageType::STATIC:
            {
              	// can early exit
                ANGLE_TRY(VertexDataManager::StoreStaticAttrib(context, translatedAttrib));
                break;
            }
            case VertexStorageType::CURRENT_VALUE:
                // Current value attribs are managed by the StateManager11.
                break;
            default:
                UNREACHABLE();
                break;
        }
    }

    return angle::Result::Continue;
}
=========================================================
template <size_t N, typename BitsT, typename ParamT>
BitSetT<N, BitsT, ParamT> &BitSetT<N, BitsT, ParamT>::reset(ParamT pos)
{
    ASSERT(mBits == (mBits & Mask(N)));
    mBits &= ~Bit<BitsT>(pos);
    return *this;
}
=========================================================
#define ANGLE_TRY(EXPR) ANGLE_TRY_TEMPLATE(EXPR, ANGLE_RETURN)

#define ANGLE_TRY_TEMPLATE(EXPR, FUNC)                \
    do                                                \
    {                                                 \
        auto ANGLE_LOCAL_VAR = EXPR;                  \
        if (ANGLE_UNLIKELY(IsError(ANGLE_LOCAL_VAR))) \
        {                                             \
            FUNC(ANGLE_LOCAL_VAR);                    \
        }                                             \
    } while (0)
=========================================================
inline bool IsError(angle::Result result)
{
    return result == angle::Result::Stop;
}
```
```c++
angle::Result VertexDataManager::StoreStaticAttrib(const gl::Context *context,
                                                   TranslatedAttribute *translated)
{
    ASSERT(translated->attribute && translated->binding);
    const auto &attrib  = *translated->attribute;
    const auto &binding = *translated->binding;

    gl::Buffer *buffer = binding.getBuffer().get();
    ASSERT(buffer && attrib.enabled && !DirectStoragePossible(context, attrib, binding));
    BufferD3D *bufferD3D = GetImplAs<BufferD3D>(buffer);

    // Compute source data pointer
    const uint8_t *sourceData = nullptr;
    const int offset          = static_cast<int>(ComputeVertexAttributeOffset(attrib, binding));

    ANGLE_TRY(bufferD3D->getData(context, &sourceData));

    if (sourceData)
    {
        sourceData += offset;
    }
[ ... ]
```

```c++
angle::Result Buffer11::getData(const gl::Context *context, const uint8_t **outData)
{
    if (mSize == 0)
    {
        // TODO(http://anglebug.com/2840): This ensures that we don't crash or assert in robust
        // buffer access behavior mode if there are buffers without any data. However, technically
        // it should still be possible to draw, with fetches from this buffer returning zero.
        return angle::Result::Stop;
    }

    SystemMemoryStorage *systemMemoryStorage = nullptr;
  	
    ANGLE_TRY(getBufferStorage(context, BUFFER_USAGE_SYSTEM_MEMORY, &systemMemoryStorage));

    ASSERT(systemMemoryStorage->getSize() >= mSize);

    *outData = systemMemoryStorage->getSystemCopy()->data();
    return angle::Result::Continue;
}
```

```c++
template <typename StorageOutT>
angle::Result Buffer11::getBufferStorage(const gl::Context *context,
                                         BufferUsage usage,
                                         StorageOutT **storageOut)
{
    ASSERT(0 <= usage && usage < BUFFER_USAGE_COUNT);
    BufferStorage *&newStorage = mBufferStorages[usage];

    if (!newStorage)
    {
        newStorage = allocateStorage(usage);
    }

    markBufferUsage(usage);

    // resize buffer
    if (newStorage->getSize() < mSize)
    {
        ANGLE_TRY(newStorage->resize(context, mSize, true));
    }

    ASSERT(newStorage);

    ANGLE_TRY(updateBufferStorage(context, newStorage, 0, mSize));
    ANGLE_TRY(garbageCollection(context, usage));

    *storageOut = GetAs<StorageOutT>(newStorage);
    return angle::Result::Continue;
}
```


### Do it

Do this exercise by yourself, If you find my answer have something wrong, please correct it.

---------
<details>
  <summary>My answer</summary>

  The answer I write is incomplete, the following answer doesn't mention the reletion between patch and uaf -_-. Recently I have no time to debug PoC to get the truely answer. So I hope you can correct this.

  patch:
  ```diff
diff --git a/src/libANGLE/renderer/d3d/d3d11/VertexArray11.cpp b/src/libANGLE/renderer/d3d/d3d11/VertexArray11.cpp
index 6bb0bf8..a5f8b6a 100644
--- a/src/libANGLE/renderer/d3d/d3d11/VertexArray11.cpp
+++ b/src/libANGLE/renderer/d3d/d3d11/VertexArray11.cpp
@@ -253,8 +253,6 @@
 
     for (size_t dirtyAttribIndex : activeDirtyAttribs)
     {
-        mAttribsToTranslate.reset(dirtyAttribIndex);
-
         auto *translatedAttrib   = &mTranslatedAttribs[dirtyAttribIndex];
         const auto &currentValue = glState.getVertexAttribCurrentValue(dirtyAttribIndex);
 
@@ -282,6 +280,9 @@
                 UNREACHABLE();
                 break;
         }
+
+        // Make sure we reset the dirty bit after the switch because STATIC can early exit.
+        mAttribsToTranslate.reset(dirtyAttribIndex);
     }
 
     return angle::Result::Continue;
  ```
  doc about [dirty bits](https://chromium.googlesource.com/angle/angle/+/50a2725742948702720232ba46be3c1f03822ada/doc/DirtyBits.md)
  
  ```c++
    angle::Result Buffer11::getData(const gl::Context *context, const uint8_t **outData)
    {
        if (mSize == 0)
        {
            return angle::Result::Stop;  [1]
        }
        SystemMemoryStorage *systemMemoryStorage = nullptr;
        // call getBufferStorage
        ANGLE_TRY(getBufferStorage(context, BUFFER_USAGE_SYSTEM_MEMORY, &systemMemoryStorage));
    }
  ```
  incomplete answer:
  [1] `Buffer11::getData` can return `angle::Result::Stop` if `mSize == 0`
  ```c++
    angle::Result VertexDataManager::StoreStaticAttrib(const gl::Context *context,
                                                    TranslatedAttribute *translated)
    {
        BufferD3D *bufferD3D = GetImplAs<BufferD3D>(buffer);
        // Compute source data pointer
        const uint8_t *sourceData = nullptr;
        const int offset          = static_cast<int>(ComputeVertexAttributeOffset(attrib, binding));

        ANGLE_TRY(bufferD3D->getData(context, &sourceData));   [2]

        if (sourceData)
        {
            sourceData += offset;
        }
    [ ... ]
    ==========================================================
    Buffer11::~Buffer11()
    {
        for (BufferStorage *&storage : mBufferStorages)
        {
            SafeDelete(storage);
        }
    [ ... ]
  ```
  [2] call `Buffer11::getData` in `ANGLE_TRY`, because `mSize == 0` it can return `Stop` then exit early. Finally call `Buffer11::~Buffer11()`, it can free `mBufferStorages`.
  
  One possible situation (maybe wrong) is we can get the raw buffer early because the "early exit", and the `Buffer11::~Buffer11()` have not been trigger. I can call `getBufferStorage` in other path during `~Buffer11()` before `mBufferStorages[usage]` was set to null.
  ```c++
    template <typename StorageOutT>
    angle::Result Buffer11::getBufferStorage(const gl::Context *context,
                                            BufferUsage usage,
                                            StorageOutT **storageOut)
    {
        ASSERT(0 <= usage && usage < BUFFER_USAGE_COUNT);
        BufferStorage *&newStorage = mBufferStorages[usage];   [3] already freed

        if (!newStorage)
        {
            newStorage = allocateStorage(usage);
        }

        markBufferUsage(usage);

        // resize buffer
        if (newStorage->getSize() < mSize)     [4] trigger uaf
        {
            ANGLE_TRY(newStorage->resize(context, mSize, true));
        }
    }
  ```
  In other path call `getBufferStorage`  will trigger uaf, like
  ```c++
    syncVertexBuffersAndInputLayout ->
        applyVertexBuffers ->
            getBuffer ->
                getBufferStorage
  ```
  I get this call tree by this [report](https://talosintelligence.com/vulnerability_reports/TALOS-2020-1127)

  If you are instread of how to construct the Poc, you can get help form [this](https://bugs.chromium.org/p/chromium/issues/attachmentText?aid=457249).

</details>

--------

