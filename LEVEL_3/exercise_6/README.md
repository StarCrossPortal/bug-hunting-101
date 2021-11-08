# Exercise 6

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21198
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.



### Details

In level 3, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1184399

</details>

--------

### Set environment

after fetch chromium
```sh
git reset --hard 983a7365ebe7fa934fb4660409105bae294d70a5
```



### Related code
```c++
// Typemapped such that arbitrarily large IPC::Message objects can be sent and
// received with minimal copying.
struct Message {
  mojo_base.mojom.BigBuffer buffer;
  array<mojo.native.SerializedHandle>? handles;
};
========================================
union BigBuffer {
  array<uint8> bytes;
  BigBufferSharedMemoryRegion shared_memory;
  bool invalid_buffer;
};
```
ipc/ipc_message_pipe_reader.cc
base/pickle.cc


tips: You just need to think how `shared_memory` can be used to breaking mojo.

### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  ```c++
// Typemapped such that arbitrarily large IPC::Message objects can be sent and
// received with minimal copying.
struct Message {
  mojo_base.mojom.BigBuffer buffer;       [1]
  array<mojo.native.SerializedHandle>? handles;
};
=======================================
union BigBuffer {
  array<uint8> bytes;
  BigBufferSharedMemoryRegion shared_memory;  [2]
  bool invalid_buffer;
};
  ```
  [1] | [2] `BigBuffer` is backed by an `array` of bytes when the message is **small**; but it's backed by `shared memory` if the message is **large**. This means that a malicious renderer can send legacy `IPC messages` backed by `shared memory`.

  ```c++
void MessagePipeReader::Receive(MessageView message_view) {
  if (!message_view.size()) {
    delegate_->OnBrokenDataReceived();
    return;
  }
  Message message(message_view.data(), message_view.size()); [3]
  if (!message.IsValid()) {
    delegate_->OnBrokenDataReceived();
    return;
  }
[ ... ]
  ```
  [3] `ipc::Message` inherits from `base::Pickle`
  ```c++
class IPC_MESSAGE_SUPPORT_EXPORT Message : public base::Pickle {
 public:
//[ ... ]
  // Initializes a message from a const block of data.  The data is not copied;
  // instead the data is merely referenced by this message.  Only const methods
  // should be used on the message when initialized this way.
  Message(const char* data, int data_len);
=============================================
Message::Message(const char* data, int data_len)
    : base::Pickle(data, data_len) {     [4]
  Init();
}
  ```
  The constructor of `Message` call `Pickle`'s constructor
  ```c++
Pickle::Pickle(const char* data, size_t data_len)
    : header_(reinterpret_cast<Header*>(const_cast<char*>(data))),
      header_size_(0),
      capacity_after_header_(kCapacityReadOnly),
      write_offset_(0) {
  if (data_len >= static_cast<int>(sizeof(Header)))     [5]
    header_size_ = data_len - header_->payload_size; 

  if (header_size_ > static_cast<unsigned int>(data_len))
    header_size_ = 0;
[ ... ]
  ```
  [5] after check `payload_size` at this point, we can chenge `payload_size` in Pickle's header by the other side which have the access of `shared_memory`

  Since `base::Pickle` expects to find a `header` at the start of the region, by changing the length field in that header after the checks in the `Pickle` constructor

  ```c++
PickleIterator::PickleIterator(const Pickle& pickle)
: payload_(pickle.payload()),
    read_index_(0),
    end_index_(pickle.payload_size()) {     [6]
}
===================================================
// This class provides facilities for basic binary value packing and unpacking.
//
// The Pickle's data has a header which contains the size of the Pickle's
// payload.  It can optionally support additional space in the header.  That
// space is controlled by the header_size parameter passed to the Pickle
// constructor.
//
class BASE_EXPORT Pickle {
 public:
 //[ ... ]
  // The payload is the pickle data immediately following the header.
  size_t payload_size() const {
    return header_ ? header_->payload_size : 0;    [7]
  }
  ```

  [6] ｜ [7] `PickleIterator`'s `end_index_ == header_->payload_size`, we can use it to oob read

  ```c++
// PickleIterator reads data from a Pickle. The Pickle object must remain valid
// while the PickleIterator object is in use.
class BASE_EXPORT PickleIterator {
 public:
  PickleIterator() : payload_(nullptr), read_index_(0), end_index_(0) {}
  explicit PickleIterator(const Pickle& pickle);

  // Methods for reading the payload of the Pickle. To read from the start of
  // the Pickle, create a PickleIterator from a Pickle. If successful, these
  // methods return true. Otherwise, false is returned to indicate that the
  // result could not be extracted. It is not possible to read from the iterator
  // after that.
  bool ReadBool(bool* result) WARN_UNUSED_RESULT;
  bool ReadInt(int* result) WARN_UNUSED_RESULT;
  bool ReadLong(long* result) WARN_UNUSED_RESULT;
  [ ... ]
  ```
  Set `PickleIterator.end_index_` to a huge num, we can get oob read by these Methods.

  **Poc**

  


</details>

--------
