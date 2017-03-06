#Android硬件抽象层
1. Android系统的硬件抽象层以模块的形式来管理各个硬件访问接口。每一个硬件模块都对应有一个动态链接库文件，这些动态链接库文件的命令需要符合一定的规范。同时，在系统内部，每一个硬件抽象层模块都使用结构体hw_module_t来描述，而硬件设备则使用结构体hw_device_t来描述。
```cpp
//Value for the hw_module_t.tag field
#define MAKE_TAG_CONSTANT(A，B，C，D) (((A) << 24) | ((B) << 16) | ((C) << 8) | (D))
#define HARDWARE_MODULE_TAG MAKE_TAG_CONSTANT('H'， 'W'， 'M'， 'T')
//Name of the hal_module_info
#define HAL_MODULE_INFO_SYM HMI
/*Every hardware module must have a data structure named HAL_MODULE_INFO_SYM and the fields of this data structure must begin with hw_module_t* followed by module specifi information.*/
typedef struct hw_module_t {
    //tag must be initialized to HARDWARE_MODULE_TAG
    uint32_t tag;
    //major version number for the module 
    uint16_t version_major;
    //minor version number of the module */
    uint16_t version_minor;
    //Identifier of module
    const char *id;
    //Name of this modul
    const char *name;
    //Author/owner/implementor of the module
    const char *author;
    //Modules methods */
    struct hw_module_methods_t* methods;
    //module's dso
    void* dso;
    //padding to 128 bytes， reserved for future use
    uint32_t reserved[32-7];
} hw_module_t;
```

2. Every hardware module must have a data structure named HAL_MODULE_INFO_SY and the fields of this data structure must begin with hw_module_t followed by module specific information.
```cpp
typedef struct hw_module_methods_t {
    /** Open a specific device */
    int (*open)(const struct hw_module_t* module， const char* id，
            struct hw_device_t** device);
} hw_module_methods_t;
```
结构体hw_module_methods_t只有一个成员变量，它是一个函数指针，用来打开硬件抽象层模块中的硬件设备。其中，参数module表示要打开的硬件设备所在的模块;参数id表示要打开的硬件设备的ID；参数device是一个输出参数，用来描述一个已经打开的硬件设备。

3. struct hw_device_t
```cpp
#define HARDWARE_DEVICE_TAG MAKE_TAG_CONSTANT('H'， 'W'， 'D'， 'T')
/**
 * Every device data structure must begin with hw_device_t
 * followed by module specific public methods and attributes.
 */
typedef struct hw_device_t {
    /** tag must be initialized to HARDWARE_DEVICE_TAG */
    uint32_t tag;
    /** version number for hw_device_t */
    uint32_t version;
    /** reference to the module this device belongs to */
    struct hw_module_t* module;
    /** padding reserved for future use */
    uint32_t reserved[12];
   /** Close this device */
    int (*close)(struct hw_device_t* device);
} hw_device_t;
```

##开发freg设备的硬件抽象层模块
1. 目录结构
![Screenshot from 2017-02-28 11:03:55.png](/home/xb/Desktop/ScreenShots/Screenshot from 2017-02-28 11:03:55.png)

2. freg.h
```cpp
#ifndef ANDROID_FREG_INTERFACE_H
#define ANDROID_FREG_INTERFACE_H
#include <hardware/hardware.h>
__BEGIN_DECLS
/*定义模块ID*/
#define FREG_HARDWARE_MODULE_ID "freg"
/*定义设备ID*/
#define FREG_HARDWARE_DEVICE_ID "freg"
/*自定义模块结构体*/
struct freg_module_t {
    struct hw_module_t common;
};
/*自定义设备结构体*/
struct freg_device_t {
    struct hw_device_t common;
    int fd;
    int (*set_val)(struct freg_device_t* dev， int val);
    int (*get_val)(struct freg_device_t* dev， int* val);
};
__END_DECLS
#endif
```

3. freg.cpp
```cpp
#define LOG_TAG "FregHALStub"
#include <hardware/hardware.h>
#include <hardware/freg.h>
#include <fcntl.h>
#include <errno.h>
#include <cutils/log.h>
#include <cutils/atomic.h>
#define DEVICE_NAME "/dev/freg"
#define MODULE_NAME "Freg"
#define MODULE_AUTHOR "shyluo@gmail.com"
/*设备打开和关闭接口*/
static int freg_device_open(const struct hw_module_t* module， const char* id， struct hw_device_t** device);
static int freg_device_close(struct hw_device_t* device);
/*设备寄存器读写接口*/
static int freg_get_val(struct freg_device_t* dev， int* val);
static int freg_set_val(struct freg_device_t* dev， int val);
/*定义模块操作方法结构体变量*/
static struct hw_module_methods_t freg_module_methods = {
    open: freg_device_open
};
/*定义模块结构体变量*/
struct freg_module_t HAL_MODULE_INFO_SYM = {
    common: {
        tag: HARDWARE_MODULE_TAG，
        version_major: 1,
        version_minor: 0,
        id: FREG_HARDWARE_MODULE_ID,
        name: MODULE_NAME,
        author: MODULE_AUTHOR,
        methods: &freg_module_methods,
    }
};
```
按照硬件抽象层模块编写规范，每一个硬件抽象层模块必须导出一个名称为HAL_MODULE_INFO_SYM的符号，它指向一个自定义的硬件抽象层模块结构体，而且它的第一个类型为hw_module_t的成员变量的tag值必须设置为HARDWARE_MODULE_TAG。

##Android硬件抽象层模块加载过程
1. 学习Android硬件抽象层模块的加载过程有助于理解它的编写规范以及实现原理。Android系统中的硬件抽象层模块是由系统统一加载的，当调用者需要加载这些模块时，只要指定它们的ID值就可以了。

2. 负责加载硬件抽象层模块的函数是hw_get_module，其实现如下：
```cpp
/** Base path of the hal modules */
#define HAL_LIBRARY_PATH1 "/system/lib/hw"
#define HAL_LIBRARY_PATH2 "/vendor/lib/hw"
static const char *variant_keys[] = {
    "ro.hardware", /* This goes first so that it can pick up a different
                      file on the emulator. */
    "ro.product.board",
    "ro.board.platform",
    "ro.arch"
};
static const int HAL_VARIANT_KEYS_COUNT =
    (sizeof(variant_keys)/sizeof(variant_keys[0]));
int hw_get_module(const char *id, const struct hw_module_t **module)
{
    int status;
    int i;
    const struct hw_module_t *hmi = NULL;
    char prop[PATH_MAX];
    char path[PATH_MAX];
    /*
     * Here we rely on the fact that calling dlopen multiple times on
     * the same .so will simply increment a refcount (and not load
     * a new copy of the library).
     * We also assume that dlopen() is thread-safe.
     */
    /* Loop through the configuration variants looking for a module */
    for (i=0 ; i<HAL_VARIANT_KEYS_COUNT+1 ; i++) {
        if (i < HAL_VARIANT_KEYS_COUNT) {
            if (property_get(variant_keys[i], prop, NULL) == 0) {
                continue;
            }
            snprintf(path, sizeof(path), "%s/%s.%s.so"，
                    HAL_LIBRARY_PATH1, id, prop);
            if (access(path， R_OK) == 0) break;
            snprintf(path， sizeof(path)， "%s/%s.%s.so"，
                    HAL_LIBRARY_PATH2, id, prop);
            if (access(path， R_OK) == 0) break;
        } else {
            snprintf(path, sizeof(path), "%s/%s.default.so"，
                    HAL_LIBRARY_PATH1, id);
            if (access(path, R_OK) == 0) break;
        }
    }
    status = -ENOENT;
    if (i < HAL_VARIANT_KEYS_COUNT+1) {
        /* load the module， if this fails， we're doomed， and we should not try
         * to load a different variant. */
        status = load(id, path, module);
    }
    return status;
}
```

##处理硬件设备访问权限问题
1. Android提供了另外的一个uevent机制，可以在系统启动时修改设备文件的访问权限。将需要修改权限的设备文件写到uevented.rc中即可。

##定义硬件访问服务接口
1. Android系统提供了一种描述语言来定义具有跨进程访问能力的服务接口，这种描述语言称为Android接口描述语言（Android Interface Definition Language，AIDL）。以AIDL定义的服务接口文件是以aidl为后缀名的，在编译时，编译系统会将它们转换成Java文件，然后再对它们进行编译。在本节中，我们将使用AIDL来定义硬件访问服务接口IFregService。在Android系统中，通常把硬件访问服务接口定义在frameworks/base/core/java/android/os目录中。
2.由于服务接口IFregService是使用AIDL语言描述的，因此，我们需要将其添加到编译脚本文件中，这样编译系统才能将其转换为Java文件，然后再对它进行编译。进入到frameworks/base目录中，打开里面的Android.mk文件，修改LOCAL_SRC_FILES变量的值。
3.在Android系统中，通常把硬件访问服务实现在frameworks/base/services/java/com/android/server目录中。
4.在Android系统中，通常把硬件访问服务的JNI方法实现在frameworks/base/services/jni目录中。
5.硬件访问服务FregService的JNI方法编写完成之后，我们还需要修改frameworks/base/services/jni目录下的onload.cpp文件，在里面增加register_android_server_FregService函数的声明和调用。onload.cpp文件实现在libandroid_servers模块中。当系统加载libandroid_servers模块时，就会调用实现在onload.cpp文件中的JNI_OnLoad函数。

##硬件抽象层学习总结
1. 按照驱动编写一般规则在kernel层编写驱动程序，完成最基本的硬件操作并向上提供访问接口。
2. 硬件抽象层模块通过open/read/write/ioctl来操作硬件，而抽象中的函数只是将这些操作进行封装。
3. hw_get_module函数来负责加载相应模块，硬件抽象层模块本身有一个导出的设备描述对象，函数在加载时将模块结构体保存到参数module中。
4. 硬件访问服务需要定义访问接口，由于硬件服务运行在SS中，因此需要Binder通信，扯出了aidl和java层硬件访问接口实现。
5. 而在硬件访问接口实现中java层需要通过jni来访问native层的函数来完成相关工作，扯出了jni。
6. 在freg_init这个jni方法中，会调用hw_get_module来加载硬件抽象层模块，这样就拿到一个hw_module_t对象，而再调用freg_device_open函数时，就会分配hw_device_t对象，并将该对象的地址转换为句柄返回到java层。而java层的set，get函数就通过该句柄来访问对应的jni方法，对应的jni方法中将句柄转换为hw_device_t对象,这样就可以调用设备的相关方法了。
7. 由于硬件访问服务需要运行在SystemServer中，因此需要：
ServiceManager.addService("freg"， new FregService());
8. 在app中，需要通过IFregService.Stub.asInterface(ServiceManager.getService("freg"))来获得相应的代理接口。