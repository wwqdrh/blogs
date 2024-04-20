# 应用角度

首先来看从应用层角度，如何通过binder驱动实现ipc通信

## 服务端

首先定义BBinder类

```cpp
String16 serviceName(o: "test.binderAddInts");

class AddIntsService : public BBinder
{
  public:
    AddIntsService(int cpu = unbound);
    virtual ~AddIntsService() {}

    enum command {
        ADD_INTS = 0x120,
    };

    virtual status_t onTransact(uint32_t code,
                                const Parcel& data,·
                                Parcel* reply,
                                uint32_t flags = 0);
  
  private:
    int cpu_;
};

// 核心函数，处理接受到的命令
status_t AddIntsService::onTransact(uint32_t code, const Parcel &data,
                                    Parcel* reply, uint32_t flags) {
    int val1, val2;
    status_t rv(0);
    int cpu;
    // If server bound to a particular CPU, check that
    // were executing on that CPU.
    if (cpu_ != unbound) {
        cpu = sched_getcpu();
        if (cpu != cpu_) {
            cerr << "server onTransact on CPU " << cpu << " expected CPU "
                  << cpu_ << endl;
            exit(status: 20);
        }
    }
    // Perform the requested operation
    switch (code) {
    case ADD_INTS:
        if (options.payloadSize == 0) {
            val1 = data.readInt32();
            val2 = data.readInt32();
            reply->writeInt32(val1 + val2);
        } else {
            val1 = data.readInt32();
            reply->writeInt32(val1);
        }
        break;
    default:
      cerr << "server onTransact unknown code, code: " << code << endl;
      exit(status: 21);
    }
    return rv;
}

// 服务启动核心函数
static void server(void)
{
    int rv;
    // Add the service
    sp<ProcessState> proc(ProcessState::self());
    // 获取servicemanager的指针
    sp<IServiceManager> sm = defaultServiceManager();
    if ((rv = sm->addService(serviceName, new AddIntsService(options.serverCPU))) != 0) {
        cerr << "addService " << serviceName << " failed, rv: " << rv
            << " errno: " << errno << endl;
    }
    // 启动线程池开始进行处理
    proc->startThreadPool();
}
```

## 客户端

```cpp
static void client(void)
{
    int rv;
    // 找到servicemanager
    sp<IServiceManager> sm = defaultServiceManager();
    double min = FLT_MAX, max = 0.0, total = 0.0; // Time in seconds for all
                                                  // the IPC calls.
    // If needed bind to client CPU
    if (options.clientCPU != unbound) {
        bindCPU(cpu: options.clientCPU);
    }
    // Attach to service
    sp<IBinder> binder;
    do {
        // 通过服务名获取服务指针
        binder = sm->getService(serviceName);
        if (binder != 0)
            break;
        cout << serviceName << " not published, waiting..." << endl;
        usleep(useconds: 500000); // 0.5 s
    } while(true);
    // Perform the IPC operations
    for (unsigned int iter = 0; iter < options.iterations; iter++) {
        Parcel send, reply;
        int expected;
        if (options.payloadSize == 0) {
            // Create parcel to be sent.  Will use the iteration cound
            // and the iteration count + 3 as the two integer values
            // to be sent.
            int val1 = iter;
            int val2 = iter + 3;
            expected = val1 + val2;  // Expect to get the sum back
            send.writeInt32(val: val1);
            send.writeInt32(val: val2);
        } else {
            expected = options.payloadSize;
            char *buf = new char[expected + 1];
            fill(first: buf, last: buf + expected, value: 'a');
            buf[expected] = 0;
            send.writeInt32(val: strlen(s: buf));
            send.writeCString(str: buf);
        }
        // Send the parcel, while timing how long it takes for
        // the answer to return.
        struct timespec start;
        clock_gettime(CLOCK_MONOTONIC, &start);
        // 调用服务的ADD_INTS方法
        if ((rv = binder->transact(AddIntsService::ADD_INTS,
            data: send, reply: &reply)) != 0) {
            cerr << "binder->transact failed, rv: " << rv
                << " errno: " << errno << endl;
            exit(status: 10);
        }
        struct timespec current;
        clock_gettime(clock_id: CLOCK_MONOTONIC, tp: &current);
        // Calculate how long this operation took and update the stats
        struct timespec deltaTimespec = tsDelta(first: &start, second: &current);
        double delta = ts2double(val: &deltaTimespec);
        min = (delta < min) ? delta : min;
        max = (delta > max) ? delta : max;
        total += delta;
        int result = reply.readInt32();
        if (result != expected) {
            cerr << "Unexpected result for iteration " << iter << endl;
            cerr << "  result: " << result << endl;
            cerr << "expected: " << expected << endl;
        } else if (options.payloadSize == 0) {
            cout << "#" << iter << " pass, result: " << result << " expected: " << expected << en»
        }
        if (options.iterDelay > 0.0) {
        >       testDelaySpin(amt: options.iterDelay);
        }
    }
    // Display the results
    cout << "Time per iteration min: " << min
        << " avg: " << (total / options.iterations)
        << " max: " << max
        << endl;
}
```