# Exercise 3

## CVE-2020-16005
I sugget you don't search any report about it to prevents get too much info like patch.

This time we do it by code audit
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
Download depot_tools and ninja
```sh
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
echo 'export PATH=$PATH:"/path/to/depot_tools"' >> ~/.bashrc # or zshrc  

git clone https://github.com/ninja-build/ninja.git
cd ninja && ./configure.py --bootstrap && cd ..
echo 'export PATH=$PATH:"/path/to/ninja"' >> ~/.bashrc
```

Sync all standalone dependencies
```sh
# open new terminal to update env
cd src/third_party/angle
python scripts/bootstrap.py   # download depot_tools in advance
gclient sync
```
Generate ANGLE standalone build files and build
```sh
gn gen out/Debug              # download ninja in advance
ninja -j 10 -k1 -C out/Debug
```

more detile in [offical](https://chromium.googlesource.com/angle/angle/+/HEAD/doc/BuildingAngleForChromiumDevelopment.md)

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
            return streamIndexData(context, indices, count, srcType, dstType,  //call streamIndexData
                                primitiveRestartFixedIndexEnabled, translated);
        }

        // Case 2: the indices are already in a buffer
        unsigned int offset = static_cast<unsigned int>(reinterpret_cast<uintptr_t>(indices));
        ASSERT(srcTypeBytes * static_cast<unsigned int>(count) + offset <= buffer->getSize());

        bool offsetAligned = IsOffsetAligned(srcType, offset);

        [ ... ]
==============================================================================
angle::Result IndexDataManager::streamIndexData(const gl::Context *context,
                                                const void *data,
                                                unsigned int count,
                                                gl::DrawElementsType srcType,
                                                gl::DrawElementsType dstType,
                                                bool usePrimitiveRestartFixedIndex,
                                                TranslatedIndexData *translated)
{
    const GLuint dstTypeShift = gl::GetDrawElementsTypeShift(dstType);

    IndexBufferInterface *indexBuffer = nullptr;
    ANGLE_TRY(getStreamingIndexBuffer(context, dstType, &indexBuffer));
    ASSERT(indexBuffer != nullptr);

    unsigned int offset;
    ANGLE_TRY(StreamInIndexBuffer(context, indexBuffer, data, count, srcType, dstType,  // call StreamInIndexBuffer
                                  usePrimitiveRestartFixedIndex, &offset));

    translated->indexBuffer = indexBuffer->getIndexBuffer();
    translated->serial      = indexBuffer->getSerial();
    translated->startIndex  = (offset >> dstTypeShift);
    translated->startOffset = offset;

    return angle::Result::Continue;
}
===================================================================================
angle::Result StreamInIndexBuffer(const gl::Context *context,
                                  IndexBufferInterface *buffer,
                                  const void *data,
                                  unsigned int count,
                                  gl::DrawElementsType srcType,
                                  gl::DrawElementsType dstType,
                                  bool usePrimitiveRestartFixedIndex,
                                  unsigned int *offset)
{
    const GLuint dstTypeBytesShift = gl::GetDrawElementsTypeShift(dstType);

    bool check = (count > (std::numeric_limits<unsigned int>::max() >> dstTypeBytesShift));
    ANGLE_CHECK(GetImplAs<ContextD3D>(context), !check,
                "Reserving indices exceeds the maximum buffer size.", GL_OUT_OF_MEMORY);

    unsigned int bufferSizeRequired = count << dstTypeBytesShift;
    ANGLE_TRY(buffer->reserveBufferSpace(context, bufferSizeRequired, dstType));

    void *output = nullptr;
    ANGLE_TRY(buffer->mapBuffer(context, bufferSizeRequired, &output, offset));

    ConvertIndices(srcType, dstType, data, count, output, usePrimitiveRestartFixedIndex);  // call

    ANGLE_TRY(buffer->unmapBuffer(context));
    return angle::Result::Continue;
}
============================================================================================
void ConvertIndices(gl::DrawElementsType sourceType,
                    gl::DrawElementsType destinationType,
                    const void *input,
                    GLsizei count,
                    void *output,
                    bool usePrimitiveRestartFixedIndex)
{
    if (sourceType == destinationType)
    {
        const GLuint dstTypeSize = gl::GetDrawElementsTypeSize(destinationType);
        memcpy(output, input, count * dstTypeSize);
        return;
    }

    if (sourceType == gl::DrawElementsType::UnsignedByte)
    {
        ASSERT(destinationType == gl::DrawElementsType::UnsignedShort);
        ConvertIndexArray<GLubyte, GLushort>(input, sourceType, output, destinationType, count,
                                             usePrimitiveRestartFixedIndex);
    }
    else if (sourceType == gl::DrawElementsType::UnsignedShort)
    {
        ASSERT(destinationType == gl::DrawElementsType::UnsignedInt);
        ConvertIndexArray<GLushort, GLuint>(input, sourceType, output, destinationType, count,
                                            usePrimitiveRestartFixedIndex);
    }
    else
        UNREACHABLE();
}
```
```c++
    angle::Result drawRangeElements(const gl::Context *context,
                                    gl::PrimitiveMode mode,
                                    GLuint start,
                                    GLuint end,
                                    GLsizei count,
                                    gl::DrawElementsType type,
                                    const void *indices) override;
```



### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>

  You can get info about [WebGL2RenderingContext.drawRangeElements()](https://developer.mozilla.org/en-US/docs/Web/API/WebGL2RenderingContext/drawRangeElements) to know how to construct Poc.
  Notice that if we call `drawRangeElements()` with a invalide parameter, the bug can be trigger. It is the easiest challenge of the three.
  ```c++
        angle::Result IndexDataManager::prepareIndexData(const gl::Context *context,
                                                    gl::DrawElementsType srcType,
                                                    gl::DrawElementsType dstType,
                                                    GLsizei count,
                                                    gl::Buffer *glBuffer,
                                                    const void *indices,
                                                    TranslatedIndexData *translated)
    {
        // Case 1: the indices are passed by pointer, which forces the streaming of index data
        if (glBuffer == nullptr)                                               [1]
        {
            translated->storage = nullptr;
            return streamIndexData(context, indices, count, srcType, dstType,  [2]
                                primitiveRestartFixedIndexEnabled, translated);
        }
  ```
  if `glBuffer == nullptr`, `indices` can be second parameter of `streamIndexData`.
  ```c++
    angle::Result IndexDataManager::streamIndexData(const gl::Context *context,
                                                const void *data,       <----------
                                                unsigned int count,
                                                gl::DrawElementsType srcType,
                                                gl::DrawElementsType dstType,
                                                bool usePrimitiveRestartFixedIndex,
                                                TranslatedIndexData *translated)
    {
        unsigned int offset;
        ANGLE_TRY(StreamInIndexBuffer(context, indexBuffer, data, count, srcType, dstType,  [3] indices as third parameter
                                    usePrimitiveRestartFixedIndex, &offset));

        return angle::Result::Continue;
    }
    =================================================================================
    angle::Result StreamInIndexBuffer(const gl::Context *context,
                                  IndexBufferInterface *buffer,
                                  const void *data,  <---------------
                                  unsigned int count,
                                  gl::DrawElementsType srcType,
                                  gl::DrawElementsType dstType,
                                  bool usePrimitiveRestartFixedIndex,
                                  unsigned int *offset)
    {
        ConvertIndices(srcType, dstType, data, count, output, usePrimitiveRestartFixedIndex);  [4] indices as third parameter

        ANGLE_TRY(buffer->unmapBuffer(context));
        return angle::Result::Continue;
    }
    ========================================================================================
    void ConvertIndices(gl::DrawElementsType sourceType,
                    gl::DrawElementsType destinationType,
                    const void *input,     <------------------
                    GLsizei count,
                    void *output,
                    bool usePrimitiveRestartFixedIndex)
    {
        if (sourceType == destinationType)
        {
            const GLuint dstTypeSize = gl::GetDrawElementsTypeSize(destinationType);
            memcpy(output, input, count * dstTypeSize);    [5] call memcpy and we can control input as any value
            return;
        }
        // other type of sourceType, but they all have assignment operation
        [ .... ]
    }
  ```
  This time I encourage you to try construct Poc, you just need call `gl.drawRangeElements(mode, start, end, count, type, offset);` and fill the last parameter named `offset` with a invalid value like `0xdeadbeef`. But there is a lot of pre-work before this, you need to search some info to reach. And test it on the angle your build ago.
  

</details>

--------

