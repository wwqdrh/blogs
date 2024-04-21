# 服务指针

这部分列出关于服务指针相关

# serviceManager

handle值是0的server就是serviceManager

```cpp
sp<IServiceManager> defaultServiceManager()
{
    if (gDefaultServiceManager != NULL) return gDefaultServiceManager;
    {
        AutoMutex _l(&gDefaultServiceManagerLock);
        while (gDefaultServiceManager == NULL) {
            gDefaultServiceManager = interface_cast<IServiceManager>(ProcessState::self()->getContextObject(caller: NULL));
            if (gDefaultServiceManager == NULL)
                sleep(seconds: 1);
        }
    }
    return gDefaultServiceManager;
}

sp<IBinder> ProcessState::getContextObject(const sp<IBinder>& /*caller*/)
{
    return getStrongProxyForHandle(handle: 0);
}

sp<IBinder> ProcessState::getStrongProxyForHandle(int32_t handle)
{
    sp<IBinder> result;
    AutoMutex _l(&mutex: mLock);
    handle_entry* e = lookupHandleLocked(handle);
    if (e != NULL) {
        IBinder* b = e->binder;
        if (b == NULL || !e->refs->attemptIncWeak(id: this)) {
            if (handle == 0) {
                Parcel data;
                status_t status = IPCThreadState::self()->transact(
                        0, IBinder::PING_TRANSACTION, data, NULL, 0);
                if (status == DEAD_OBJECT)
                   return NULL;
            }
            b = new BpBinder(handle);·
            e->binder = b;
            if (b) e->refs = b->getWeakRefs();
            result = b;
        } else {
            result.force_set(other: b);
            e->refs->decWeak(id: this);
        }
    }
    return result;
}
```

# server

```cpp
class BpServiceManager : public BpInterface<IServiceManager>
{
public:
     virtual sp<IBinder> getService(const String16& name) const
     {
         unsigned n;
         for (n = 0; n < 5; n++) {
             sp<IBinder> svc = checkService(name);
             if (svc != NULL)
                 return svc;
             ALOGI("Waiting for service %s...\n", String8(name).string());
             sleep(seconds: 1);
         }
         return NULL;
     }

     virtual sp<IBinder> checkService( const String16& name) const
     {
         Parcel data, reply;
         data.writeInterfaceToken(IServiceManager::getInterfaceDescriptor());
         data.writeString16(name);
         remote()->transact(CHECK_SERVICE_TRANSACTION, data, &reply);
         return reply.readStrongBinder();
     }

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
}
```
