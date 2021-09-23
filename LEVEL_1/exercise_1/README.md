# Exercise 1

## CVE-2020-6542
I choose **CVE-2020-6542**, and I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.

In order to reduce the difficulty of setting up the environment, we do it by code auditing.


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

--------

### Version

Google Chrome  84.0.4147.89

Google Chrome  85.0.4169.0 (Developer Build) (64-bit)

we can analysis the source file [online](https://chromium.googlesource.com/angle/angle/+/034a8b3f3c5c8e7e1629b8ac88cadb72ea68cf23/src/libANGLE/renderer/d3d/d3d11/VertexArray11.cpp#246)

### Related code

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

    ANGLE_TRY(bufferD3D->getData(context, &sourceData));   // call Buffer11::getData

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
  	// call getBufferStorage
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

```c++
call trace:
VertexArray11::updateDirtyAttribs ->
  VertexDataManager::StoreStaticAttrib ->
  	Buffer11::getData ->  // maybe we needn't call getBufferStorage
  		Buffer11::getBufferStorage
```

### Do it

Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.

---------
<details>
  <summary>My answer</summary>

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
  [1] `Buffer11::getData` can return return `angle::Result::Stop` if `mSize == 0`
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

