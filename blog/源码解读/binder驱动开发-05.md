# 服务注册流程

主要是下面的`sp<IBinder>`是什么类型，然后data.writeStrongBinder的时候，是将其转换为什么类型给内核

```cpp
 virtual status_t addService(const String16& name, const sp<IBinder>& service,
             bool allowIsolated)
{
    Parcel data, reply;
    data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
    data.writeString16(name);
    data.writeStrongBinder(service);
    data.writeInt32(allowIsolated ? 1 : 0);
    status_t err = remote()->transact(ADD_SERVICE_TRANSACTION, data, &reply);
    return err == NO_ERROR ? reply.readExceptionCode() : err;
}
```

`IBinder`是一个协议类, 在使用`data.writeStrongBiner`函数的时候会将其转换成`flat_binder_object`

```cpp
struct flat_binder_object {
    struct binder_object_header hdr;
    __u32 flags;
    union {
        binder_uintptr_t binder;
        __u32 handle;
    };
    binder_uintptr_t cookie;
};
```