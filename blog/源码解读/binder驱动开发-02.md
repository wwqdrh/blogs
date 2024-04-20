# 服务中心servicemanager

ipc通信中

服务端需要使用servicemanager进行注册

客户端也需要使用servicemanager通过服务名获取到对应的服务指针

> 那么在这一个流程中，servicemanager起到什么作用呢

# 核心结构

核心结构仅有一个存储svcinfo的链表

```cpp
struct svcinfo
{
    struct svcinfo *next;
    uint32_t handle;
    struct binder_death death;
    int allow_isolated;
    size_t len;
    uint16_t name[0];
};
struct svcinfo* svclist = NULL;

int main() {
    struct binder_state *bs;
    bs = binder_open(128*1024);
    // ...
    binder_loop(bs, svcmgr_handler);
    return 0;
}

// 入口函数，通过传递的命令确认是注册还是获取服务
int svcmgr_handler(struct binder_state *bs,
                   struct binder_transaction_data *txn,
                   struct binder_io *msg,
                   struct binder_io *reply)
{
    // skip some code
     switch(txn->code) {
     case SVC_MGR_GET_SERVICE:
     case SVC_MGR_CHECK_SERVICE:
         s = bio_get_string16(msg, &len);
         if (s == NULL) {
             return -1;
         }
         handle = do_find_service(bs, s, len, txn->sender_euid, txn->sender_pid);
         if (!handle)
             break;
         bio_put_ref(bio: reply, handle);
         return 0;

     case SVC_MGR_ADD_SERVICE:
         s = bio_get_string16(msg, &len);
         if (s == NULL) {
             return -1;
         }
         handle = bio_get_ref(msg);
         allow_isolated = bio_get_uint32(msg) ? 1 : 0;
         if (do_add_service(bs, s, len, handle, txn->sender_euid,
             allow_isolated, txn->sender_pid))
             return -1;
         break;
     }
}
```

# 注册流程

```cpp
int do_add_service(struct binder_state *bs,
                   const uint16_t *s, size_t len,
                   uint32_t handle, uid_t uid, int allow_isolated,
                   pid_t spid)
{
    struct svcinfo *si;
    //ALOGI("add_service('%s',%x,%s) uid=%d\n", str8(s, len), handle,
    //        allow_isolated ? "allow_isolated" : "!allow_isolated", uid);
    if (!handle || (len == 0) || (len > 127))
        return -1;
    if (!svc_can_register(name: s, name_len: len, spid)) {
        ALOGE("add_service('%s',%x) uid=%d - PERMISSION DENIED\n",
             str8(x: s, x_len: len), handle, uid);
        return -1;
    }
    si = find_svc(s16: s, len);
    if (si) {
        if (si->handle) {
            ALOGE("add_service('%s',%x) uid=%d - ALREADY REGISTERED, OVERRIDE\n",
                 str8(x: s, x_len: len), handle, uid);
            svcinfo_death(bs, ptr: si);
        }
        si->handle = handle;
    } else {
        si = malloc(size: sizeof(*si) + (len + 1) * sizeof(uint16_t));
        if (!si) {
            ALOGE("add_service('%s',%x) uid=%d - OUT OF MEMORY\n",
                 str8(x: s, x_len: len), handle, uid);
            return -1;
        }
        si->handle = handle;
        si->len = len;
        memcpy(dest: si->name, src: s, n: (len + 1) * sizeof(uint16_t));
        si->name[len] = '\0';
        si->death.func = (void*) svcinfo_death;
        si->death.ptr = si;
        si->allow_isolated = allow_isolated;
        si->next = svclist;
        svclist = si;
    }
    binder_acquire(bs, target: handle);
    binder_link_to_death(bs, target: handle, death: &si->death);
    return 0;
}
```

# 查找流程

```cpp
uint32_t do_find_service(struct binder_state *bs, const uint16_t *s, size_t len, uid_t uid, pid_t spid)
{
    struct svcinfo *si = find_svc(s16: s, len);
    if (!si || !si->handle) {
        return 0;
    }
    // if (!si->allow_isolated) {
    //     // If this service doesn't allow access from isolated processes,
    //     // then check the uid to see if it is isolated.
    //     uid_t appid = uid % AID_USER;
    //     if (appid >= AID_ISOLATED_START && appid <= AID_ISOLATED_END) {
    //         return 0;
    //     }
    // }
    if (!svc_can_find(s, len, spid)) {
        return 0;
    }
    return si->handle;
}

struct svcinfo* find_svc(const uint16_t *s16, size_t len)
{
    struct svcinfo *si;
    for (si = svclist; si; si = si->next) {
        if ((len == si->len) &&
            !memcmp(s1: s16, s2: si->name, n: len * sizeof(uint16_t))) {
            return si;
        }
    }
    return NULL;
}
```