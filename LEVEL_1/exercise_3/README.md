# Exercise 3

## CVE-2020-16005
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.

This time we do it by code audit, and download source code.
### Details

> When the WebGL2RenderingContext.drawRangeElements() API is processed in the ANGLE library, IndexDataManager::prepareIndexData is called internally.
In prepareIndexData, if glBuffer is a null pointer, the second argument of streamIndexData becomes indices. This value is the last parameter(offset) of the drawRangeElements, a 4-byte integer which can be arbitrarily set.

---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

   https://bugs.chromium.org/p/chromium/issues/detail?id=1139398
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
git  reset --hard 6e1259375f2d6f579fe7430442a9657e00d15656  # we get it by issue page
```

### Related code
we can analysis the source file [online](https://chromium.googlesource.com/angle/angle/+/6e1259375f2d6f579fe7430442a9657e00d15656/src/libANGLE/renderer/d3d/IndexDataManager.cpp#135) or offline.

```c++
    // This function translates a GL-style indices into DX-style indices, with their description
    // returned in translated.
    // GL can specify vertex data in immediate mode (pointer to CPU array of indices), which is not
    // possible in DX and requires streaming (Case 1). If the GL indices are specified with a buffer
    // (Case 2), in a format supported by DX (subcase a) then all is good.
    // When we have a buffer with an unsupported format (subcase b) then we need to do some translation:
    // we will start by falling back to streaming, and after a while will start using a static
    // translated copy of the index buffer.
    angle::Result IndexDataManager::prepareIndexData(const gl::Context *context,
                                                    gl::DrawElementsType srcType,
                                                    gl::DrawElementsType dstType,
                                                    GLsizei count,
                                                    gl::Buffer *glBuffer,
                                                    const void *indices,
                                                    TranslatedIndexData *translated)
    {
        GLuint srcTypeBytes = gl::GetDrawElementsTypeSize(srcType);
        GLuint srcTypeShift = gl::GetDrawElementsTypeShift(srcType);
        GLuint dstTypeShift = gl::GetDrawElementsTypeShift(dstType);

        BufferD3D *buffer = glBuffer ? GetImplAs<BufferD3D>(glBuffer) : nullptr;

        translated->indexType                 = dstType;
        translated->srcIndexData.srcBuffer    = buffer;
        translated->srcIndexData.srcIndices   = indices;
        translated->srcIndexData.srcIndexType = srcType;
        translated->srcIndexData.srcCount     = count;

        // Context can be nullptr in perf tests.
        bool primitiveRestartFixedIndexEnabled =
            context ? context->getState().isPrimitiveRestartEnabled() : false;

        // Case 1: the indices are passed by pointer, which forces the streaming of index data
        if (glBuffer == nullptr)
        {
            translated->storage = nullptr;
            return streamIndexData(context, indices, count, srcType, dstType,
                                primitiveRestartFixedIndexEnabled, translated);
        }

        // Case 2: the indices are already in a buffer
        unsigned int offset = static_cast<unsigned int>(reinterpret_cast<uintptr_t>(indices));
        ASSERT(srcTypeBytes * static_cast<unsigned int>(count) + offset <= buffer->getSize());

        bool offsetAligned = IsOffsetAligned(srcType, offset);

        // Case 2a: the buffer can be used directly
        if (offsetAligned && buffer->supportsDirectBinding() && dstType == srcType)
        {
            translated->storage     = buffer;
            translated->indexBuffer = nullptr;
            translated->serial      = buffer->getSerial();
            translated->startIndex  = (offset >> srcTypeShift);
            translated->startOffset = offset;
            return angle::Result::Continue;
        }

        translated->storage = nullptr;

        // Case 2b: use a static translated copy or fall back to streaming
        StaticIndexBufferInterface *staticBuffer = buffer->getStaticIndexBuffer();

        bool staticBufferInitialized = staticBuffer && staticBuffer->getBufferSize() != 0;
        bool staticBufferUsable =
            staticBuffer && offsetAligned && staticBuffer->getIndexType() == dstType;

        if (staticBufferInitialized && !staticBufferUsable)
        {
            buffer->invalidateStaticData(context);
            staticBuffer = nullptr;
        }

        if (staticBuffer == nullptr || !offsetAligned)
        {
            const uint8_t *bufferData = nullptr;
            ANGLE_TRY(buffer->getData(context, &bufferData));
            ASSERT(bufferData != nullptr);

            ANGLE_TRY(streamIndexData(context, bufferData + offset, count, srcType, dstType,
                                    primitiveRestartFixedIndexEnabled, translated));
            buffer->promoteStaticUsage(context, count << srcTypeShift);
        }
        else
        {
            if (!staticBufferInitialized)
            {
                const uint8_t *bufferData = nullptr;
                ANGLE_TRY(buffer->getData(context, &bufferData));
                ASSERT(bufferData != nullptr);

                unsigned int convertCount =
                    static_cast<unsigned int>(buffer->getSize()) >> srcTypeShift;
                ANGLE_TRY(StreamInIndexBuffer(context, staticBuffer, bufferData, convertCount, srcType,
                                            dstType, primitiveRestartFixedIndexEnabled, nullptr));
            }
            ASSERT(offsetAligned && staticBuffer->getIndexType() == dstType);

            translated->indexBuffer = staticBuffer->getIndexBuffer();
            translated->serial      = staticBuffer->getSerial();
            translated->startIndex  = (offset >> srcTypeShift);
            translated->startOffset = (offset >> srcTypeShift) << dstTypeShift;
        }

        return angle::Result::Continue;
    }
```





### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  


</details>

--------

