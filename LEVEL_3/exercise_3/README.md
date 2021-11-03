# Exercise 3

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21223
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.



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
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


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
  
</details>

--------
