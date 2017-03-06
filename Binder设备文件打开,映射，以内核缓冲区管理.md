##5.1 Binder设备文件打开,映射，以内核缓冲区管理
1. 打开过程
	1) 一个需要使用binder通信的进程，首先要打开设备文件/dev/binder/, 此时binder_open函数被调用，该函数中将为该进程创建一个binder_proc.
	2) 创建的binder_proc将被保存在参数filp的private_data中。因为进程调用open函数打开/dev/binder函数后，内核会返回一个文件描述符给进程，而该文件描述符是与filp 所指向的打开文件结构体关联在一起的，因此后面以这个文件描述符为参数调用其他关键函数来与Binder驱动交互，内核就会将与该文件描述符相关联的打开文件结构体传递给Binder驱动，Binder驱动就可以通过filp的private_data成员取出该进程的Binder_proc.
2. 设备文件内存映射过程
	1). /dev/binder是一个虚拟的设备，将它映射到进程的地址空间的目的不是为了获取其内容，而数据为了给进程分配内核缓冲区，该缓冲区被用来传输进程间通信数据。
	2).　Binder驱动程序为进程分配的内核缓冲区在用户空间只读，不能拷贝，并禁止设置可能进行的写操作标志位设置。
	3). Bindder驱动分配的内核缓冲区有两个虚地址，一个用户空间地址，由参数vma指向的vm_area_struct描述；另外一个是内核空间地址，由变量area指向的vm_struct结构体描述。
	4) 在binder_mmap函数中，”proc->free_async_space = proc->buffer_size/2“来设定了最大可以用于异步事务的内核缓冲区为总的内核缓冲区大小的一半。
3. 内核缓冲去管理
	1) 物理内存的分配以页为单位，但是一次使用却不是一页为单位，因此，驱动为进程维护了一个内核缓冲区池，缓冲区池中的每一块内存都用结构体binder_buffer描述，并保存到一个列表中。
	2)内存缓冲区分配时机：　当一个进程以命令BC_TRANSACTION和HC_REPLAY向另外一个进程传递数据时，Binder驱动需要将这些数据从用户空间拷贝到内核空间，然后再传递给目标进程，此时驱动就需要在目标进程的内存池中分配小块内核缓冲区来保存这些数据。
	3)非常重要的说明：
![Screenshot from　 2016-06-30 15:44:20.png](/home/xb/Desktop/Screenshot from 2016-06-30 15:44:20.png)
![8148ab458b17c620564d6c777d6a921b.jpeg](/home/xb/Desktop/8148ab458b17c620564d6c777d6a921b.jpeg)
	4)查询内核缓冲区
![Screenshot from 2016-06-30 16:20:14.png](/home/xb/Desktop/Screenshot from 2016-06-30 16:20:14.png)
![Screenshot from 2016-06-30 16:21:49.png](/home/xb/Desktop/Screenshot from 2016-06-30 16:21:49.png)
