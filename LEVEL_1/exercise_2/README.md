# Exercise 2

## CVE-2020-6542
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in true, Poc can proof that we are right.

In order to reduce the difficulty of setting up the environment, we do it by code auditing.

## Details

> Use-After-Free vulnerability in libglesv2!gl::Texture::onUnbindAsSamplerTexture

---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

   https://bugs.chromium.org/p/chromium/issues/detail?id=1065186
</details>

--------

### Tested Versions

Google Chrome 83.0.4093.3 dev (64 bit)


### related code
\src\third_party\angle\src\libANGLE\Texture.h
we can analysis the source file [online](https://chromium.googlesource.com/angle/angle/+/50a2725742948702720232ba46be3c1f03822ada/src/libANGLE/renderer/d3d/d3d11/Buffer11.cpp#801)



```c++
Buffer11::Buffer11(const gl::BufferState &state, Renderer11 *renderer)
    : BufferD3D(state, renderer),
      mRenderer(renderer),
      mSize(0),
      mMappedStorage(nullptr),
      mBufferStorages({}),   // empty
      mLatestBufferStorage(nullptr),
      mDeallocThresholds({}),
      mIdleness({}),
      mConstantBufferStorageAdditionalSize(0),
      mMaxConstantBufferLruCount(0),
      mStructuredBufferStorageAdditionalSize(0),
      mMaxStructuredBufferLruCount(0)
{}
Buffer11::~Buffer11()
{
    for (BufferStorage *&storage : mBufferStorages) //A way of interval iteration, like python
    {
        SafeDelete(storage);
    }
[ .... ]
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
template <typename T>
void SafeDelete(T *&resource)
{
    delete resource;
    resource = nullptr;
}
```

```c++
// The order of this enum governs priority of 'getLatestBufferStorage'.
enum BufferUsage
{
    BUFFER_USAGE_SYSTEM_MEMORY,
    BUFFER_USAGE_STAGING,
    BUFFER_USAGE_VERTEX_OR_TRANSFORM_FEEDBACK,
    BUFFER_USAGE_INDEX,
    BUFFER_USAGE_INDIRECT,
    BUFFER_USAGE_PIXEL_UNPACK,
    BUFFER_USAGE_PIXEL_PACK,
    BUFFER_USAGE_UNIFORM,
    BUFFER_USAGE_STRUCTURED,
    BUFFER_USAGE_EMULATED_INDEXED_VERTEX,
    BUFFER_USAGE_RAW_UAV,

    BUFFER_USAGE_COUNT,
};
```

```c++
Buffer11::BufferStorage *Buffer11::allocateStorage(BufferUsage usage)
{
    updateDeallocThreshold(usage);
    switch (usage)
    {
        case BUFFER_USAGE_PIXEL_PACK:
            return new PackStorage(mRenderer);
        case BUFFER_USAGE_SYSTEM_MEMORY:
            return new SystemMemoryStorage(mRenderer);
        case BUFFER_USAGE_EMULATED_INDEXED_VERTEX:
            return new EmulatedIndexedStorage(mRenderer);
        case BUFFER_USAGE_INDEX:
        case BUFFER_USAGE_VERTEX_OR_TRANSFORM_FEEDBACK:
            return new NativeStorage(mRenderer, usage, this);
        case BUFFER_USAGE_STRUCTURED:
            return new StructuredBufferStorage(mRenderer, usage, nullptr);
        default:
            return new NativeStorage(mRenderer, usage, nullptr);
    }
}
```

Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.




---------

<details>
  <summary>My answer</summary>


  we can see that the begain of `getBufferStorage` have range check. But in true, it can oob read. Because `BUFFER_USAGE_COUNT` not equal to the length of `mBufferStorages`, so it can trade one buffer which not valid as `BufferStorage`.

  What's more, the `newStorage` which we get from `allocateStorage` func, have not inserted into `mBufferStorages`, this mean if we call it twice, it will allocate twice for the same use. Maybe it can be used for breaking it, but I have not further analysis.

  If you are instread of how to construct the Poc, you can get help form [this](https://bugs.chromium.org/p/chromium/issues/attachmentText?aid=457249).

</details>

--------

