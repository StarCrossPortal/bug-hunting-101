# Exercise 3

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene. therefore in LEVEL 2 we do the same as LEVEL 1 without the help of Details.

## CVE-2021-21112
I sugget you don't search any report about it to prevents get too much info like patch.


### Details

In level 2, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1151298

</details>

--------

### Set environment

after you fetch chromium
```sh
git reset --hard 13aa7b32816e52bf1242d073ada2c892798190e7 
```

### Related code

[third_party/blink/renderer/modules/compression/deflate_transformer.cc](https://chromium.googlesource.com/chromium/src.git/+/13aa7b32816e52bf1242d073ada2c892798190e7/third_party/blink/renderer/modules/compression/deflate_transformer.cc)
[third_party/blink/renderer/modules/compression/inflate_transformer.cc](https://chromium.googlesource.com/chromium/src.git/+/13aa7b32816e52bf1242d073ada2c892798190e7/third_party/blink/renderer/modules/compression/inflate_transformer.cc)

You can read this [doc](https://docs.google.com/document/d/1TovyqqeC3HoO0A4UUBKiCyhZlQSl7jM_F7KbWjK2Gcs/edit)  and [this](https://github.com/WICG/compression/blob/main/explainer.md) to get some info about `CompressionStreams` in chrome

Read comment of [third_party/zlib/zlib.h](https://chromium.googlesource.com/chromium/src.git/+/13aa7b32816e52bf1242d073ada2c892798190e7/third_party/zlib/zlib.h) to get detailed about `deflate` or `inflate`

<details>
  <summary>If you find not release operation, you can get some tips here</summary>

  We can write the target data to chunk for compress by `CompressionStream('deflate').writable.getWriter().write([data])`, also we can read the compressed output.

  At the end of read operation, we can set "then" prototype to some javascript code to free the compressing buffer. But, how can we trigger uaf? Is the compression continue after we free? When should we release it?

</details>

### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>
  
  I have assumed a lot of vulnerability types, but finally they all false. I find not free operation during the compression. I know the compress need allocate chunk to save data, and free some of them. But none of them exist in deflate func. Maybe it’s because I’m not familiar enough with it. In order to reduce the difficulty, I write some tips I get from author.

  Author supply a method can be universally used which I said above.

  If you have read the tow docs about `CompressionStreams`, you can understand the source code quickily. If chunk too large, we need divide it into multiple small to compress one by one.

  As I side above, the compression operation is carried out step by step. The output can be read step by step, and at the end of every read operation, we can set "then" prototype to some javascript code to free the compressing buffer.

  ```c++
void DeflateTransformer::Deflate(const uint8_t* start,
                                 wtf_size_t length,
                                 IsFinished finished,
                                 TransformStreamDefaultController* controller,
                                 ExceptionState& exception_state) {
  stream_.avail_in = length;
  // Zlib treats this pointer as const, so this cast is safe.
  stream_.next_in = const_cast<uint8_t*>(start);

  do {
    stream_.avail_out = out_buffer_.size();
    stream_.next_out = out_buffer_.data();
    int err = deflate(&stream_, finished ? Z_FINISH : Z_NO_FLUSH);
    DCHECK((finished && err == Z_STREAM_END) || err == Z_OK ||
           err == Z_BUF_ERROR);

    wtf_size_t bytes = out_buffer_.size() - stream_.avail_out;   [1]
    if (bytes) {
      controller->enqueue(                                    [2]
          script_state_,
          ScriptValue::From(script_state_,
                            DOMUint8Array::Create(out_buffer_.data(), bytes)),
          exception_state);
      if (exception_state.HadException()) {
        return;
      }
    }
  } while (stream_.avail_out == 0);
}
  ```
  [1] calculate the remaining data needs to be compressed, and [2] call `enqueue` to compress them next time.

  A part of result will output after every time compression. We can read it and after we finish the read of once compression output, `then` func trigged. After free the compressing buffer, the next time of compression will trigger uaf.



  

</details>

--------
