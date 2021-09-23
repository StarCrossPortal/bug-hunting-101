# Exercise 2

## CVE-2020-6542
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.

This time we do it by code audit, and download source code.
### Details


> When a new texture is bound, the texture binding state is updated before
> updating the active texture cache. With this ordering, it is possible to delete
> the currently bound texture when the binding changes and then use-after-free it
> when updating the active texture cache.
>
> The bug reason in angle/src/libANGLE/State.cpp

---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

   https://bugs.chromium.org/p/chromium/issues/detail?id=1065186
</details>

--------

### Set environment
We download the ANGLE
```sh
git clone https://chromium.googlesource.com/angle/angle
```
Then checkout the branch, we set the commit hash
```sh
cd angle
git  reset --hard b83b0f5e9f63261d3d95a75b74ad758509d7a349 # we get it by issue page
```

### Related code
we can analysis the source file [online](https://chromium.googlesource.com/angle/angle/+/e514b0cb7e6b8956ea0c93ceca01b63d5deb621d/src/libANGLE/State.cpp#1171) or offline.


```c++
void State::setSamplerTexture(const Context *context, TextureType type, Texture *texture)
{
    mSamplerTextures[type][mActiveSampler].set(context, texture);
    if (mProgram && mProgram->getActiveSamplersMask()[mActiveSampler] &&
        IsTextureCompatibleWithSampler(type, mProgram->getActiveSamplerTypes()[mActiveSampler]))
    {
        updateActiveTexture(context, mActiveSampler, texture);
    }
    mDirtyBits.set(DIRTY_BIT_TEXTURE_BINDINGS);
}
=================================================================
ANGLE_INLINE void State::updateActiveTexture(const Context *context,
                                             size_t textureIndex,
                                             Texture *texture)
{
    const Sampler *sampler = mSamplers[textureIndex].get();
    mCompleteTextureBindings[textureIndex].bind(texture);

    if (!texture)
    {
        mActiveTexturesCache.reset(textureIndex);
        mDirtyBits.set(DIRTY_BIT_TEXTURE_BINDINGS);
        return;
    }

    updateActiveTextureState(context, textureIndex, sampler, texture);
}
```

```cpp
using TextureBindingMap    = angle::PackedEnumMap<TextureType, TextureBindingVector>;
=======================================================
TextureBindingMap mSamplerTextures;
```

```c++
    void set(const ContextType *context, ObjectType *newObject)
    {
        // addRef first in case newObject == mObject and this is the last reference to it.
        if (newObject != nullptr)
        {
            reinterpret_cast<RefCountObject<ContextType, ErrorType> *>(newObject)->addRef();
        }

        // Store the old pointer in a temporary so we can set the pointer before calling release.
        // Otherwise the object could still be referenced when its destructor is called.
        ObjectType *oldObject = mObject;
        mObject               = newObject;
        if (oldObject != nullptr)
        {
            reinterpret_cast<RefCountObject<ContextType, ErrorType> *>(oldObject)->release(context); 
        }
    }
```


### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  By reading detail, we can know `the texture binding state is updated before updating the active texture cache`.
  ```c++
    void State::setSamplerTexture(const Context *context, TextureType type, Texture *texture)
    {
        mSamplerTextures[type][mActiveSampler].set(context, texture);    [1]
        if (mProgram && mProgram->getActiveSamplersMask()[mActiveSampler] &&
            IsTextureCompatibleWithSampler(type, mProgram->getActiveSamplerTypes()[mActiveSampler]))
        {
            updateActiveTexture(context, mActiveSampler, texture);   [2]
        }
        mDirtyBits.set(DIRTY_BIT_TEXTURE_BINDINGS);
    }
  ```
  [1] means update the binding state, and [2] means update the active texture cache. What can it delete currently bound texture?
  ```c++
    void set(const ContextType *context, ObjectType *newObject)
    {
        // addRef first in case newObject == mObject and this is the last reference to it.
        if (newObject != nullptr)
        {
            reinterpret_cast<RefCountObject<ContextType, ErrorType> *>(newObject)->addRef();
        }

        // Store the old pointer in a temporary so we can set the pointer before calling release.
        // Otherwise the object could still be referenced when its destructor is called.
        ObjectType *oldObject = mObject;
        mObject               = newObject;
        if (oldObject != nullptr)
        {
            reinterpret_cast<RefCountObject<ContextType, ErrorType> *>(oldObject)->release(context);  [3]
        }
    }
  ```
  There is release in set func, and the Triggering condition is `oldObject != nullptr` we can easily get this by set same `texture` twice. If we call `State::setSamplerTexture` twice with same `texture`, it can trigger uaf at `updateActiveTexture` in the second call.

</details>

--------

