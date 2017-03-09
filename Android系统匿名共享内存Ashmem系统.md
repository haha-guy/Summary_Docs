#Android系统匿名共享内存Ashmem系统
##Android系统匿名共享内存Ashmem（Anonymous Shared Memory）驱动程序源代码分析
1. 。Ashmem驱动程序实现在kernel/common/mm/ashmem.c文件中，它的模块初始化函数定义为ashmem_init：
```cpp
static struct file_operations ashmem_fops = {
    .owner = THIS_MODULE,
    .open = ashmem_open,
    .release = ashmem_release,
    .mmap = ashmem_mmap,
    .unlocked_ioctl = ashmem_ioctl,
    .compat_ioctl = ashmem_ioctl,
};
static struct miscdevice ashmem_misc = {
    .minor = MISC_DYNAMIC_MINOR,
    .name = "ashmem",
    .fops = &ashmem_fops,
};
static int __init ashmem_init(void)
{
    int ret;
    ......
    ret = misc_register(&ashmem_misc);
    if (unlikely(ret)) {
        printk(KERN_ERR "ashmem: failed to register misc device!\n");
        return ret;
    }
    ......
    return 0;
}
```
Ahshmem驱动程序在加载时，会创建一个/dev/ashmem的设备文件，这是一个misc类型的设备。这个设备文件提供了open、mmap、release和ioctl四种操作。

2. 匿名共享内存的创建操作
在Android应用程序框架层提供MemoryFile类的构造函数中，进行了匿名共享内存的创建操作，我们先来看一下这个构造函数的实现
```java
public class MemoryFile
{
    ......
    private static native FileDescriptor native_open(String name, int length) throws IOException;
    ......
    private FileDescriptor mFD;        // ashmem file descriptor
    ......
    private int mLength;    // total length of our ashmem region
    ......
    /**
    * Allocates a new ashmem region. The region is initially not purgable.
    *
    * @param name optional name for the file (can be null).
    * @param length of the memory file in bytes.
    * @throws IOException if the memory file could not be created.
    */
    public MemoryFile(String name, int length) throws IOException {
        mLength = length;
        mFD = native_open(name, length);
        ......
    }
    ......
}
```
这个构造函数最终是通过JNI方法native_open来创建匿名内存共享文件。
```cpp
static jobject android_os_MemoryFile_open(JNIEnv* env, jobject clazz, jstring name, jint length)
{
    const char* namestr = (name ? env->GetStringUTFChars(name, NULL) : NULL);
    int result = ashmem_create_region(namestr, length);
    if (name)
        env->ReleaseStringUTFChars(name, namestr);
    if (result < 0) {
        jniThrowException(env, "java/io/IOException", "ashmem_create_region failed");
        return NULL;
    }
    return jniCreateFileDescriptor(env, result);
}
```
通过运行时库提供的接口ashmem_create_region来创建匿名共享内存
```cpp
/*
 * ashmem_create_region - creates a new ashmem region and returns the file
 * descriptor, or <0 on error
 *
 * `name' is an optional label to give the region (visible in /proc/pid/maps)
 * `size' is the size of the region, in page-aligned bytes
 */
int ashmem_create_region(const char *name, size_t size)
{
    int fd, ret;
    fd = open(ASHMEM_DEVICE, O_RDWR);
    if (fd < 0)
        return fd;
    if (name) {
        char buf[ASHMEM_NAME_LEN];
        strlcpy(buf, name, sizeof(buf));
        ret = ioctl(fd, ASHMEM_SET_NAME, buf);
        if (ret < 0)
            goto error;
    }
    ret = ioctl(fd, ASHMEM_SET_SIZE, size);
    if (ret < 0)
        goto error;
    return fd;
error:
    close(fd);
    return ret;
}
```
```cpp
/*
 * ashmem_area - anonymous shared memory area
 * Lifecycle: From our parent file's open() until its release()
 * Locking: Protected by `ashmem_mutex'
 * Big Note: Mappings do NOT pin this structure; it dies on close()
 */
struct ashmem_area {
    char name[ASHMEM_FULL_NAME_LEN];/* optional name for /proc/pid/maps */
    struct list_head unpinned_list; /* list of all ashmem areas */
    struct file *file;      /* the shmem-based backing file */
    size_t size;            /* size of the mapping, in bytes */
    unsigned long prot_mask;    /* allowed prot bits, as vm_flags */
};
```
域name表示这块共享内存的名字，这个名字会显示/proc/<pid>/maps文件中，<pid>表示打开这个共享内存文件的进程ID；域unpinned_list是一个列表头，它把这块共享内存中所有被解锁的内存块连接在一起，下面我们讲内存块的锁定和解锁操作时会看到它的用法；域file表示这个共享内存在临时文件系统tmpfs中对应的文件，在内核决定要把这块共享内存对应的物理页面回收时，就会把它的内容交换到这个临时文件中去；域size表示这块共享内存的大小；域prot_mask表示这块共享内存的访问保护位。
 在Ashmem驱动程中，所有的ashmem_area实例都是从自定义的一个slab缓冲区创建的。这个slab缓冲区是在驱动程序模块初始化函数创建的，我们来看一个这个初始化函数的相关实现：
```cpp
static int __init ashmem_init(void)
{
    int ret;
    ashmem_area_cachep = kmem_cache_create("ashmem_area_cache",
        sizeof(struct ashmem_area),
        0, 0, NULL);
    if (unlikely(!ashmem_area_cachep)) {
        printk(KERN_ERR "ashmem: failed to create slab cache\n");
        return -ENOMEM;
    }
    ......
    return 0;
}
```
回到前面的ashmem_create_region函数中,调用这个open函数最终会进入到Ashmem驱动程序中的ashmem_open函数中去：
```cpp
static int ashmem_open(struct inode *inode, struct file *file)
{
    struct ashmem_area *asma;
    int ret;
    ret = nonseekable_open(inode, file);
    if (unlikely(ret))
        return ret;
    asma = kmem_cache_zalloc(ashmem_area_cachep, GFP_KERNEL);
    if (unlikely(!asma))
        return -ENOMEM;
    INIT_LIST_HEAD(&asma->unpinned_list);
    memcpy(asma->name, ASHMEM_NAME_PREFIX, ASHMEM_NAME_PREFIX_LEN);
    asma->prot_mask = PROT_MASK;
    file->private_data = asma;
    return 0;
}
```
首先是通过nonseekable_open函数来设备这个文件不可以执行定位操作，即不可以执行seek文件操作。接着就是通过kmem_cache_zalloc函数从刚才我们创建的slab缓冲区ashmem_area_cachep来创建一个ashmem_area结构体了，并且保存在本地变量asma中。再接下去就是初始化变量asma的其它域，其中，域name初始为ASHMEM_NAME_PREFIX
**函数的最后是把这个ashmem_area结构保存在打开文件结构体的private_data域中，这样，Ashmem驱动程序就可以在其它地方通过这个private_data域来取回这个ashmem_area结构了。**
再回到ashmem_create_region函数中，又调用了两次ioctl文件操作分别来设备这块新建的匿名共享内存的名字和大小。
ASHMEM_SET_NAME和ASHMEM_SET_SIZE的定义为:
```cpp
    #define ASHMEM_NAME_LEN     256
    #define __ASHMEMIOC     0x77
    #define ASHMEM_SET_NAME     _IOW(__ASHMEMIOC, 1, char[ASHMEM_NAME_LEN])
    #define ASHMEM_SET_SIZE     _IOW(__ASHMEMIOC, 3, size_t)
```
先来看ASHMEM_SET_NAME命令的ioctl调用，它最终进入到Ashmem驱动程序的ashmem_ioctl函数中
```cpp
static long ashmem_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct ashmem_area *asma = file->private_data;
    long ret = -ENOTTY;
    switch (cmd) {
    case ASHMEM_SET_NAME:
        ret = set_name(asma, (void __user *) arg);
        break;
    ......
    }
    return ret;
}
```
```cpp
static long ashmem_ioctl(struct file *file, unsigned int cmd, unsigned long arg)
{
    struct ashmem_area *asma = file->private_data;
    long ret = -ENOTTY;
    switch (cmd) {
    ......
    case ASHMEM_SET_SIZE:
        ret = -EINVAL;
        if (!asma->file) {
            ret = 0;
/*注意并没有实际分配size大小的内存，后面应该会在此处有额外的操作*/
            asma->size = (size_t) arg;
        }
        break;
    ......
    }
    return ret;
}
```
Ashmem驱动程序不提供read和write文件操作，进程若要访问这个共享内存，必须要把这个设备文件映射到自己的进程空间中，然后进行直接内存访问，这就是我们下面要介绍的匿名共享内存设备文件的内存映射操作了。

3. 匿名共享内存设备文件的内存映射操作
在MemoryFile类的构造函数中，进行了匿名共享内存的创建操作后，下一步就是要把匿名共享内存设备文件映射到进程空间来了：
mAddress = native_mmap(mFD, length, PROT_READ | PROT_WRITE);
映射匿名共享内存设备文件到进程空间是通过JNI方法native_mmap来进行的。这个JNI方法实现在frameworks/base/core/jni/adroid_os_MemoryFile.cpp文件中：
```cpp
static jint android_os_MemoryFile_mmap(JNIEnv* env, jobject clazz, jobject fileDescriptor,
        jint length, jint prot)
{
/*将java文件描述符转为cpp文件描述符*/
    int fd = jniGetFDFromFileDescriptor(env, fileDescriptor);
    jint result = (jint)mmap(NULL, length, prot, MAP_SHARED, fd, 0);
    if (!result)
        jniThrowException(env, "java/io/IOException", "mmap failed");
    return result;
}
```
```cpp
static int ashmem_mmap(struct file *file, struct vm_area_struct *vma)
{
    struct ashmem_area *asma = file->private_data;
    int ret = 0;
    mutex_lock(&ashmem_mutex);
    /* user needs to SET_SIZE before mapping */
    if (unlikely(!asma->size)) {
        ret = -EINVAL;
        goto out;
    }
   /* requested protection bits must match our allowed protection mask */
    if (unlikely((vma->vm_flags & ~asma->prot_mask) & PROT_MASK)) {
        ret = -EPERM;
        goto out;
    }
    if (!asma->file) {
        char *name = ASHMEM_NAME_DEF;
        struct file *vmfile;
        if (asma->name[ASHMEM_NAME_PREFIX_LEN] != '\0')
            name = asma->name;
        /* ... and allocate the backing shmem file */
/在此处完成了实际内存大小的分配/
        vmfile = shmem_file_setup(name, asma->size, vma->vm_flags);
        if (unlikely(IS_ERR(vmfile))) {
            ret = PTR_ERR(vmfile);
            goto out;
        }
        asma->file = vmfile;
    }
    get_file(asma->file);
    if (vma->vm_flags & VM_SHARED)
        shmem_set_file(vma, asma->file);
    else {
        if (vma->vm_file)
            fput(vma->vm_file);
        vma->vm_file = asma->file;
    }
    vma->vm_flags |= VM_CAN_NONLINEAR;
out:
    mutex_unlock(&ashmem_mutex);
    return ret;
}
```
