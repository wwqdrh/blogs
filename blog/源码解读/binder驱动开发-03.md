# 驱动文件

现在我们要开始与驱动文件打交道了，这一部分是封装在framework中

前面代码中涉及到的跟驱动相关的文件包括如下

- binder_open
- binder_write
- binder_loop
- binder_acquire
- binder_link_to_death

还有一些与ipc各个对象相关的指针，例如defaultServiceManager，序列化对象Parcel则放在后面来讲

# binder_open

```cpp
struct binder_state *binder_open(size_t mapsize)
{
    struct binder_state *bs;
    struct binder_version vers;
    bs = malloc(sizeof(*bs));
    if (!bs) {
        errno = ENOMEM;
        return NULL;
    }
    bs->fd = open(file: "/dev/binder", oflag: O_RDWR);
    if (bs->fd < 0) {
        fprintf(stream: stderr,format: "binder: cannot open device (%s)\n",
                strerror(errnum: errno));
        goto fail_open;
    }
    if ((ioctl(bs->fd, BINDER_VERSION, &vers) == -1) ||
        (vers.protocol_version != BINDER_CURRENT_PROTOCOL_VERSION)) {
        fprintf(stderr, "binder: kernel driver version (%d) differs from user space version (%d)\n",
                vers.protocol_version, BINDER_CURRENT_PROTOCOL_VERSION);
        goto fail_open;
    }
    bs->mapsize = mapsize;
    // 开辟mmap空间，需要从这块空间拷贝数据
    bs->mapped = mmap(NULL, mapsize, PROT_READ, MAP_PRIVATE, bs->fd, 0);
    if (bs->mapped == MAP_FAILED) {
        fprintf(stream: stderr,format: "binder: cannot map device (%s)\n",
                strerror(errnum: errno));
        goto fail_map;
    }
    return bs;
fail_map:
    close(fd: bs->fd);
fail_open:
    free(ptr: bs);
    return NULL;
}
```

# binder_loop

```cpp
void binder_loop(struct binder_state *bs, binder_handler func)
{
    int res;
    struct binder_write_read bwr;
    uint32_t readbuf[32];
    bwr.write_size = 0;
    bwr.write_consumed = 0;
    bwr.write_buffer = 0;
    readbuf[0] = BC_ENTER_LOOPER;
    // 传递BC_ENTER_LOOPER告诉驱动该服务已经进入循环了
    binder_write(bs, data: readbuf, len: sizeof(uint32_t));
    for (;;) {
        // 开始循环读取命令
        bwr.read_size = sizeof(readbuf);
        bwr.read_consumed = 0;
        bwr.read_buffer = (uintptr_t) readbuf;
        res = ioctl(bs->fd, BINDER_WRITE_READ, &bwr);
        if (res < 0) {
            ALOGE("binder_loop: ioctl failed (%s)\n", strerror(errnum: errno));
            break;
        }
        res = binder_parse(bs, 0, (uintptr_t) readbuf, bwr.read_consumed, func);
        if (res == 0) {
            ALOGE("binder_loop: unexpected reply?!\n");
            break;
        }
        if (res < 0) {
            ALOGE("binder_loop: io error %d %s\n", res, strerror(errnum: errno));
            break;
        }
    }
}
```

# binder_write

```cpp
int binder_write(struct binder_state *bs, void *data, size_t len)
{
    struct binder_write_read bwr;
    int res;
    bwr.write_size = len;
    bwr.write_consumed = 0;
    bwr.write_buffer = (uintptr_t) data;
    bwr.read_size = 0;
    bwr.read_consumed = 0;
    bwr.read_buffer = 0;
    res = ioctl(fd: bs->fd, request: BINDER_WRITE_READ, &bwr);
    if (res < 0) {
        fprintf(stream: stderr,format: "binder_write: ioctl failed (%s)\n",
                strerror(errnum: errno));
    }
    return res;
}
```

# binder_send_reply

```cpp
void binder_send_reply(struct binder_state *bs,
                       struct binder_io *reply,
                       binder_uintptr_t buffer_to_free,
                       int status)
{
    struct {
        uint32_t cmd_free;
        binder_uintptr_t buffer;
        uint32_t cmd_reply;
        struct binder_transaction_data txn;
    } __attribute__((packed)) data;
    data.cmd_free = BC_FREE_BUFFER;
    data.buffer = buffer_to_free;
    data.cmd_reply = BC_REPLY;
    data.txn.target.ptr = 0;
    data.txn.cookie = 0;
    data.txn.code = 0;
    if (status) {
        data.txn.flags = TF_STATUS_CODE;
        data.txn.data_size = sizeof(int);
        data.txn.offsets_size = 0;
        data.txn.data.ptr.buffer = (uintptr_t)&status;
        data.txn.data.ptr.offsets = 0;
    } else {
        data.txn.flags = 0;
        data.txn.data_size = reply->data - reply->data0;
        data.txn.offsets_size = ((char*) reply->offs) - ((char*) reply->offs0);
        data.txn.data.ptr.buffer = (uintptr_t)reply->data0;
        data.txn.data.ptr.offsets = (uintptr_t)reply->offs0;
    }
    binder_write(bs, data: &data, len: sizeof(data));
}
```
