# 应用角度

首先来看从应用层角度，如何通过binder驱动实现ipc通信

## 服务端

首先定义BBinder类

```cpp
#include <binder/IPCThreadState.h>
#include <binder/ProcessState.h>
#include <binder/IServiceManager.h>

using namespace android;
String16 serviceName(o: "test.binderAddInts");

struct options {
    unsigned int payloadSize;
} options = {
    .payloadSize=2,
};

class BasicService : public BBinder {
public:
    enum commands {
        ADD = 0x10
    };
    virtual status_t onTransact(uint32_t code, const Parcel &data, Parcel *reply, uint32_t flags = 0) override;
};

status_t BasicService::onTransact(uint32_t code, const Parcel &data, Parcel *reply, uint32_t flags) {
    status_t rv(0);
    switch (code) {
     case ADD:
         int res = 0;
         for(int i = 0; i < options.payloadSize; i++) {
             res += data.readInt32();
         }
         std::cout << "get a result: " << res << std::endl;
         reply->writeInt32(res);
         break;
     }
     return rv;
 }

 int main() {
     int rv;
     // Add the service
     sp<ProcessState> proc(ProcessState::self());
     sp<IServiceManager> sm = defaultServiceManager();
     if ((rv = sm->addService(serviceName, new BasicService())) != 0) {
        std::cerr << "addService " << serviceName << " failed, rv: " << rv << " errno: " << errno << std::endl;
    }
     // Start threads to handle server work
     proc->startThreadPool();
     // wait stop

     char ikey = 0;
     while(ikey != 'q')
     {
             ikey = getchar();
             std::cout << ikey;
     }
     std::cout << std::endl << "Server End" << std::endl;

     return 0;
 }
```

## 客户端

```cpp
using namespace android;
int main() {
    int rv;
    sp<IServiceManager> sm = defaultServiceManager();
    // Attach to service
    sp<IBinder> binder;
    do {
        binder = sm->getService(serviceName);
        if (binder != 0) break;
        std::cout << serviceName << " not published, waiting..." << std::endl;
        usleep(useconds: 500000); // 0.5 s
    } while(true);

    Parcel send, reply;
    int expected;
    int val1 = 1;
    int val2 = 1;
    expected = val1 + val2;  // Expect to get the sum back
    send.writeInt32(val: val1);
    send.writeInt32(val: val2);
    if ((rv = binder->transact(code: BasicService::ADD,
            data: send, reply: &reply)) != 0) {
        std::cerr << "binder->transact failed, rv: " << rv << " errno: " << errno << std::endl;
        exit(status: 10);
    }
    int result = reply.readInt32();
    if (result != expected) {
        std::cerr << "expected: " << expected << std::endl;
    }
 }
```