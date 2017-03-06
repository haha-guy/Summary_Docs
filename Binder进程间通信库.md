##Binder进程间通信库和Binder对象死亡通知机制

###Binder进程间通信库
1. Android系统在应用程序框架层中将各种Binder驱动程序操作封装成一个Binder库，一边通过调用库提供的接口来实现进程间通信。
2. 组件间的对应关系：
![fbba5e1633fa2de4d0c8bc22436ca0e7.jpeg](/home/xb/Desktop/fbba5e1633fa2de4d0c8bc22436ca0e7.jpeg)

3.　模板类BnInterface继承了BBinder类，BBinder为Binder本地对象提供了抽象的进程间通信接口
4. BBinder类
![Screenshot from 2016-06-30 17:16:58.png](/home/xb/Desktop/Screenshot from 2016-06-30 17:16:58.png)
**当一个Binder代理对象通过Binder驱动程序向一个Binder本地对象发出一个通信请求时，驱动就会调用该Binder本地对象的成员函数transact函数来处理请求,而成员函数onTransact函数由BBinder的子类，即Binder本地对象类来实现，该函数的功能是分发与业务相关的进程间通信请求**
5. BpRefBase类
	BpInterface继承BpRefBase类，BoRefBase类为Binder代理对象提供了抽象的进程间通信接口。
	BpRefBase类有个重要的成员变量mRemote，指向一个BpBinder对象。
6. IPCThreadState 类
	BBinder类和BpBinder类，都是通过IPCThreadState类来和Binder驱动交互的。
	1) 每一个使用了Binder进程间通信机制的进程都有一个Binder线程池，用来处理进程间通信请求。而对于每一个Binder线程池来说，它的内部都有一个IPCThreadState对象，通过IPCThreadState类的静态成员函数slef来获取，并且调用它的成员函数transact来与Binder交互。
	2)　IPCThreadState类有一个成员变量mProcess，指向一个ProcessState对象，对于每一个使用了Binder进程间通信机制的进程来说，内部都有一个PorcessState对象，负责初始化Binder设备，即打开并映射/dev/binder，该对象进程范围内唯一，由进程和线程的关系知，每一个线程都可以通过它与Binder驱动建立连接。
7. ProcessState类
![Screenshot from 2016-07-02 11:09:56.png](/home/xb/Desktop/Screenshot from 2016-07-02 11:09:56.png)

###Binder死亡通知机制
	Binder通信中采用的死亡通知机制是，首先Binder代理对象将一个死亡通知注册到Binder驱动程序，然后Binder驱动程序监测到它引用的Binder本地对象死亡时，Binder驱动程序就会向它发送一个死亡通知。此外，当一个Binder代理对象不再需要接收它引用的Binder对象时的死亡通知时，也可以注销之前注册的接收通知。
	1)　自定义的死亡通知接受者必须要重写父类DeathRecipient的成员函数bindeDied。当驱动程序通知一个Binder代理对象其其所引用的Binder本地对象已经死亡时，就会调用它所指定的死亡通知接收者的成员函数binderDied.